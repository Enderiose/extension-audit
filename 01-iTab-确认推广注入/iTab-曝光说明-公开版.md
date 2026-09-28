# iTab（新标签页扩展）行为说明：默认电商图标指向推广跳转域，并会改写搜索引擎链接

**被检查对象**：iTab 新标签页扩展
- Edge 应用商店版 ID `inedkoakiaeepjoblbiiipedngonadhn`，版本 **2.2.25**（本机此前实际安装的版本）
- Chrome 应用商店版 ID `mhloojimgilafopcmlcikiidgbbnelip`，版本 **2.3.13**（当前最新版）

**取证方式**：从两家官方应用商店分别下载原始安装包（未做任何修改）后做静态分析，并对其中涉及的域名做可重复的网络请求验证
**取证时间**：2026-09-26
**原始包哈希**：
- `itab-edge.crx`（v2.2.25）MD5 `a8b1a8f9e7dec0ed89dc68785ba24462`
- `itab-chrome.crx`（v2.3.13）MD5 `247d71066852e38d74a64d9bee31033e` ｜ SHA256 `6b03ae67f69114740adc0ec375dd12f61d122ed1b78b7d788a35ca62955f9088`

---

## 一、结论摘要

1. **默认导航图标里的淘宝 / 京东 / 天猫 / 1688 全部指向扩展自己的跳转域 `*.coml.cc`**，而不是官方域名。它把官方域名当作子域名前缀，拼在自己的 `coml.cc` 域名下，肉眼几乎无法分辨。
2. 实测这些域名**全部 302 跳转到 360 的推广点击跟踪服务**（`extdd1.browser.360.cn/browser/clk?m=...`）。
3. 扩展代码会**采集浏览器指纹与地理位置上报**，接收服务端下发的**百度推广码**，并把**搜索引擎链接强制改写**为带推广码的地址；服务端还可以**按 key 覆写任意搜索引擎的链接**。
4. 其后果是：浏览器历史、地址栏补全库里会**反复出现带推广码的百度地址**，用户手动删除后**会再次自动生成**。

本文只陈述可复现的事实，不对开发者的主观动机作评价。

---

## 二、证据

### 证据 1：默认导航图标指向 `*.coml.cc`（见 `shots/01-default-nav-icons.png`；两个版本结果完全一致）

文件：`chunks/defaultNavConfig-B7uRQ5vb.js`

| 图标名称 | 代码中实际写入的 URL | 官方域名应为 |
|---|---|---|
| 淘宝 | `https://taobao.coml.cc/` | `taobao.com` |
| 京东商城 | `https://jd.coml.cc/` | `jd.com` |
| 天猫精选 | `https://tmall.coml.cc/` | `tmall.com` |
| 阿里巴巴 | `https://1688.coml.cc/` | `1688.com` |
| 百度 | `https://www.baidu.com/?tn=15007414_23_dg` | 无推广参数 |

原始代码（节选）：

```js
{url:"https://taobao.coml.cc/",  type:"icon", name:"淘宝",     src:".../icons/taobao.svg", ...}
{url:"https://jd.coml.cc/",      type:"icon", name:"京东商城", src:".../icons/jd.svg", ...}
{url:"https://tmall.coml.cc/",   type:"icon", name:"天猫精选", src:".../icons/tmall.svg", ...}
{url:"https://1688.coml.cc/",    type:"icon", name:"阿里巴巴", src:".../icons/1688.svg", ...}
{url:"https://www.baidu.com/?tn=15007414_23_dg", type:"icon", name:"百度", ...}
```

### 证据 2：实测 302 跳转到 360 推广服务（见 `shots/02-redirect-302.png`）

```
$ curl -I https://jd.coml.cc/
HTTP/2 302
location: https://extdd1.browser.360.cn/browser/clk?m=<超长 token>&st=...&dd=...&ud=...
```

`taobao.coml.cc` 与 `tmall.coml.cc` 返回**同样结构**的 302。完整响应见 `raw/04-redirect-302.txt`。

### 证据 3：代码强制改写搜索引擎链接（见 `shots/03-rewrite-search-url.png`）

文件：`chunks/statistics-BK8vUFO4.js`

```js
let d = { fp: r(), tn: i, city: o };          // 上报：浏览器指纹 + 随机推广位 + 城市
s(d).then(i => {
    let s = i.data || {},
        o = s.tn || "52176495_dg&ch=2",       // 服务端下发的百度推广码（兜底值硬编码在包内）
        r = s.so360;                          // 服务端下发的 360 推广链接
    ...
    m.value.searchEngine.forEach(e => {
      "baidu" === e.key
        ? e.href = `https://www.baidu.com/s?wd=%s&tn=${o}&ie=utf-8`   // 强制拼上推广码
        : "360" === e.key && r
          ? e.href = r
          : e.href = s[e.key] || e.href       // 服务端可按 key 覆写任意搜索引擎的链接
    });
    t.set("app-baiduTn", o);                  // 推广码被持久化，后续所有百度跳转都会带上
```

要点：
- 这不是"默认值"，而是**直接对 `href` 赋值改写**；
- 推广码由**服务器下发**，且随「浏览器指纹 + 城市」一起请求（`bdRodomTn` 是本地缓存的"百度随机推广位"）；
- 「360」引擎有专门的推广链接下发通道（`s.so360`）；
- 兜底推广码 `52176495_dg&ch=2` 硬编码在扩展包内。

### 证据 4：包内共出现 3 个不同推广码（见 `shots/04-all-promo-codes.png`）

```
chunks/defaultNavConfig-B7uRQ5vb.js : https://www.baidu.com/?tn=15007414_23_dg
chunks/openEngineSearch-H9nYcJKw.js : https://www.baidu.com/s?&tn=15007414_23_dg
chunks/public-api-DajUmvL1.js       : https://www.baidu.com/s?&tn=52176495_dg
chunks/statistics-BK8vUFO4.js       : o = s.tn || "52176495_dg&ch=2"   ← 服务端下发
chunks/main-pR7HcDL9.js             : `https://www.baidu.com/s?wd=${hitokoto}&tn=${e}`
```

同一个包内存在**多个不同推广码**，说明推广位是**按渠道 / 按用户分配**的。

### 证据 5：本机实测后果（见 `shots/05-local-machine-proof.png`）

在一台 Windows + Edge 的机器上实测：

- 扩展自身存储中出现 `app-baiduTn = "75144485_6_dg&ch=4"`（服务端按用户下发）；
- 浏览器历史与地址栏补全库中反复出现 `https://www.baidu.com/?tn=75144485_6_dg&ch=4`（`visit_count=21`）；
- 地址栏补全库中存在 `user_text="baidu" → url=https://www.baidu.com/?tn=75144485_6_dg&ch=4` 的条目；
- **手动删除上述记录后，再次在地址栏输入 `baidu` 并回车，记录会立刻重新出现**（因为每次跳转都在重新拼接推广地址）。

### 证据 6：权限范围

```
permissions: bookmarks, activeTab, contextMenus, scripting, storage, alarms, unlimitedStorage, search
optional_host_permissions: ["https://www.baidu.com/", "<all_urls>"]
```

包含 `scripting` 权限，并可申请 `<all_urls>`（即"在所有网站上注入脚本"）。这些行为与其核心功能"新标签页"无关。

### 证据 7：Edge 版（本机此前实际安装的版本）同样如此（见 `shots/06-edge-package-2.2.25.png`）

为排除"只有 Chrome 版有问题"的可能，另行下载了 **Edge 官方商店包（v2.2.25）** 核对，结论一致：

```
MD5 a8b1a8f9e7dec0ed89dc68785ba24462

chunks/defaultNavConfig-B7uRQ5vb.js
    淘宝      -> https://taobao.coml.cc/
    京东商城  -> https://jd.coml.cc/
    天猫精选  -> https://tmall.coml.cc/
    阿里巴巴  -> https://1688.coml.cc/
    百度      -> https://www.baidu.com/?tn=15007414_23_dg

chunks/statistics-zvMbGPTs.js
    "baidu" === e.key ? e.href = `https://www.baidu.com/s?wd=%s&tn=${o}&ie=utf-8` : ...
    t.set("app-baiduTn", o)

包内 tn= 推广码：
    chunks/store-ve3N8KMS.js      : tn=15007414_23_dg
    chunks/public-api-tJRK9Yyi.js : tn=52176495_dg
    chunks/App-lmVczxdM.js        : tn=15007414_23_dg
    chunks/main-Wfs5oDmC.js       : tn=15007414_23_dg
    content-scripts/content.js    : tn=15007414_23_dg
```

即：**Edge 版与 Chrome 版行为一致**，同样的默认跳转域、同样的推广码、同样的链接改写逻辑。

---

## 三、如何自行验证

```bash
# 1. 从官方商店下载原始包
curl -L -o itab.crx "https://clients2.google.com/service/update2/crx?response=redirect&prodversion=120.0&acceptformat=crx2,crx3&x=id%3Dmhloojimgilafopcmlcikiidgbbnelip%26uc"
md5sum itab.crx          # 应为 247d71066852e38d74a64d9bee31033e

# 2. 解压（CRX3 = 文件头 + ZIP）
python -c "import zipfile,io;raw=open('itab.crx','rb').read();i=raw.find(b'PK\x03\x04');zipfile.ZipFile(io.BytesIO(raw[i:])).extractall('itab')"

# 3. 默认导航里电商图标指向哪里
grep -o 'https://[a-z0-9.]*coml\.cc/' itab/chunks/defaultNavConfig-*.js

# 4. 推广码与链接改写
grep -o 'tn=[0-9A-Za-z_]*' itab/chunks/*.js | sort -u
grep -o 'e.href=s\[e.key\]||e.href' itab/chunks/*.js

# 5. 验证跳转去向
curl -I https://jd.coml.cc/
```

安装扩展后，可在新标签页上右键查看内置图标的目标地址，或在浏览器历史中搜索 `tn=`。

---

## 四、影响

1. **购物入口被替换**：用户以为自己点的是京东 / 淘宝 / 天猫 / 1688，实际先经过第三方跳转域，再到 360 的推广跟踪服务。
2. **搜索引擎被注入推广码**：用户所有百度搜索都会带上由服务端指定的推广码，且**每次跳转都会重新写入历史记录，用户无法通过常规方式清除**。
3. **隐私**：跳转请求与配置请求中带有浏览器指纹、城市等标识信息。
4. 上述行为**均未在扩展商店页面中向用户明示**。

---

## 五、诉求

1. 请扩展作者就此作出说明：默认电商图标的跳转域、服务端下发推广码并改写搜索引擎链接，是否为有意设计；
2. 请在扩展描述中**明确披露**推广与跳转行为，或直接移除；
3. 请 Chrome / Edge 扩展商店依规审核该扩展是否存在**欺骗性重定向**与**未经明示的推广注入**；
4. 已安装用户建议先卸载该扩展，并把自己导航栏中的电商图标地址改回官方域名。

---

## 六、说明与免责

- 本文所有结论均来自**对扩展原始安装包的静态分析与对外可重复的网络请求**，复现步骤见第三节；
- 文中提及的 360、京东、淘宝、天猫、1688、百度等，仅为**跳转链路的终点或被仿冒的品牌**，本文不对其作任何指控；
- 本文不涉及任何个人信息，也不对开发者个人作评价；
- 引用的代码片段仅为说明问题所必需的**最小必要片段**。

---

## 七、证据清单

```
itab-edge.crx                      Edge 官方商店原始包 v2.2.25（本机实际安装版本）
itab-chrome.crx                    Chrome 官方商店原始包 v2.3.13
raw/00-hashes.txt                  两个包的 MD5 / SHA256 与权限信息
raw/01-default-nav-icons.txt       默认导航图标 URL 原文
raw/02-rewrite-search-url.txt      链接改写代码原文
raw/03-promo-codes.txt             包内全部 tn= 位置
raw/04-redirect-302.txt            302 跳转完整响应
raw/05-edge-package-2.2.25.txt     Edge 版对应证据
shots/01-default-nav-icons.png     证据图 1  默认导航图标真实 URL
shots/02-redirect-302.png          证据图 2  实测 302 跳转至 360 推广服务
shots/03-rewrite-search-url.png    证据图 3  强制改写搜索引擎链接
shots/04-all-promo-codes.png       证据图 4  包内全部推广码
shots/05-local-machine-proof.png   证据图 5  本机实测后果
shots/06-edge-package-2.2.25.png   证据图 6  Edge 版（本机实际安装版本）同样如此
```
