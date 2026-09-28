# 青柠起始页 审查报告（Chrome + Edge）

**扩展 ID**：Chrome `aajmeahnoefldipnjgodkobfdcpmapno` ／ Edge `pcpnigdkpcgemocnjhebmajldpjlbeom`
**被检查版本**：1.4.0（两家官方商店原始包）
**哈希**：

| 商店 | 包 | 大小 | MD5 | SHA256 |
|---|---|---|---|---|
| Chrome | `lime.crx` | 26,665,489 B | `2936aeef53a1c9bb079bfda3cf7e93d3` | `fa9856e0a4a2e6ed4aba14539b699b2eaaba646145eaba0663aaa70239b3dee9` |
| Edge | `lime-edge.crx` | 26,659,073 B | `7990e470de73852d188b09f84ba826c7` | `fff0bb921da6ecdfca2b54c3ea65b740fc73d3a69eb6e1c964073e68dcef3998` |

**取证时间**：2026-09-26
**取证方式**：官方商店接口下载原始包 → 静态分析（S0–S3）→ 远程站点特征核验（S4）→ 两版逐字节比对
**审查级别**：**A — 未发现推广注入或返利劫持行为**

> 流程依据：根目录 `../AGENTS.md`（S0–S6）；参考规范：`../docs/SPEC.md`。

---

## 一、结论

在本次检查的 **1.4.0 版本**（Chrome 与 Edge 两个商店包）中，**未发现**推广注入、返利跳转、指纹采集、浏览记录读取或远程链接改写行为。

其权限申请克制，且**权限用途收窄**（例如清理缓存仅限自家域名），不具备实现"改写其他网站链接"所需的任何能力。

**两版关系**：Edge 版与 Chrome 版的**全部可执行代码逐字节相同**，仅 `manifest.json` 的 `update_url` 不同（外加 3 个打包残留文件）。详见 `raw/04-diff-chrome-vs-edge.txt`。

---

## 二、关于"阅读浏览记录权限"的预警（重点核实）

Edge 商店评论区担心该扩展**读取浏览记录**。逐项核实如下：

| 核实项 | 结果 |
|---|---|
是否申请 `history`（"读取和更改您的浏览历史记录"）权限 | **否** |
是否调用 `chrome.history` API | **否**（0 处） |
是否调用 `chrome.cookies` / `document.cookie` | **否**（0 处） |
申请的 `browsingData` 权限是什么 | 仅"**清除**浏览数据"，**不包含读取能力** |
`browsingData` 的实际调用范围 | `remove({origins:["https://www.limestart.cn"]}, …)` —— **收窄到自家域名** |

**要点**：`browsingData` 是**"清除"**类权限（安装提示文案为 *Clear your browsing data / 清除您的浏览数据*），其 API 只有 `remove*` 系列方法，**结构上无法读取**历史记录。要读取历史必须申请 `history` 权限（安装提示为 *Read and change your browsing history*），本扩展**未申请**。

代码中唯一的 `browsingData` 调用出现在 `scripts/popup-ext.js` 的"修复缓存"按钮，且限定了 `origins:["https://www.limestart.cn"]`：

```js
await chrome.browsingData.remove(
  { origins: ["https://www.limestart.cn"] },
  { cache:true, cacheStorage:true, cookies:true, fileSystems:true,
    indexedDB:true, localStorage:true, serviceWorkers:true })
```

即：**只清除青柠自家域名下的缓存/Cookie 等**，不触碰全局数据，也不能读取任何历史。

> 结论：**该预警不成立**。它很可能是把权限名 `browsingData`（"浏览数据"）误读为"读取浏览记录"。真正读取历史需要 `history` 权限，本扩展没有。

---

## 三、检查项目与结果

| 检查项目 | 结果 |
|---|---|
默认导航/快捷方式中的电商链接 | **0 处**（包内无 jd.com / taobao.com / tmall.com / 1688.com） |
仿冒域（把官方域名当子域前缀） | **0 处** |
第三方跳转域与 302 链路 | **0 处**（无 `/clk`、`/click`、`/track`、`/union`、`/cps` 路径） |
推广码（`tn=` / `pid=` / `unionid=` / `cps=` 等） | **0 处** |
服务端配置改写链接 | **无**（不存在"服务端下发 → 覆写 href"的代码路径） |
指纹 / 位置采集 | **0 处** |
动态执行（`eval` / `new Function` / `atob`） | **0 处** |
远程脚本注入（`createElement('script')` / `insertAdjacentHTML`） | **0 处** |
**浏览记录读取（`chrome.history`）** | **0 处** |
**Cookie 读取（`chrome.cookies` / `document.cookie`）** | **0 处** |
权限与功能匹配度 | **匹配**（见第五节） |

---

## 四、全部代码（包内脚本合计约 12 KB，已逐行阅读）

| 文件 | 作用 |
|---|---|
`scripts/background.js`（438 B） | 快捷键 `add-shortcut` → 打开 `https://www.limestart.cn/?addUrl=<当前网址>` |
`scripts/newtab-ext.js`（681 B） | 新标签页：设置 favicon；若用户开启 `redirectNewTab` 则跳转到自家站点 |
`scripts/customize-limestart.js`（245 B） | 在自家域名注入一个隐藏 `<div id="ext">` 标记扩展版本号 |
`scripts/popup-ext.js`（10.9 KB） | 弹窗：修复缓存、跳转设置页、打开自家/合作方链接、跳转官方商店、更新日志 |
`newtab-ext.html`（521 B） | **`<iframe src="https://www.limestart.cn/">`** |

- `href=` 全部 3 处：`location.href=frontend`（指向自家站点，且仅在用户**主动开启** `redirectNewTab` 或 Android 下生效）与设置 favicon —— 均为正常用法。
- 全部 `window.open(...)` 均为**用户点击后**打开：自家站点、设置页、`dunight.cn`（合作方"度夜"扩展）、`www.retiehe.com`（热铁盒软件）、QQ 群、微博，以及 Chrome / Edge / Firefox 三个官方商店页面。
- 全部 `chrome.*` API 调用见 `raw/03-scan-results.txt` 与 `raw/05-permissions-and-usage.txt`：仅 `tabs.create/query/getCurrent/remove/reload`、`storage.sync`、`commands`、`runtime`、`browsingData.remove`——**无 history、无 cookies、无 webRequest、无 scripting**。

---

## 五、权限与用途对照（逐项核实）

```json
"permissions": ["activeTab", "browsingData", "commands", "storage"],
"content_scripts": [{
  "matches": ["https://beta.limestart.cn/*", "https://www.limestart.cn/*"],
  "js": ["scripts/customize-limestart.js"], "all_frames": true }],
"web_accessible_resources": [{ "resources": ["wallpapers/*"],
  "matches": ["https://beta.limestart.cn/*", "https://www.limestart.cn/*"] }],
"chrome_url_overrides": { "newtab": "newtab-ext.html" },
"chrome_settings_overrides": { "homepage": "https://www.limestart.cn/?from=ext" }
```

**未申请**：`history`、`scripting`、`webRequest`、`declarativeNetRequest`、`tabs`（仅 `activeTab`）、`<all_urls>`、`proxy`、`bookmarks`。
→ 即：它在结构上**无法**读取浏览历史、**无法**读取 Cookie、**无法**向第三方网站注入脚本、**无法**拦截/重写网络请求。

| 权限 | 实际用途（来自代码） | 评价 |
|---|---|---|
`activeTab` + `commands` | 快捷键"把当前网页加入捷径"（`tabs.query` 取当前标签 URL） | 合规 |
`browsingData` | 弹窗"修复缓存"，`remove({origins:["https://www.limestart.cn"]}, …)` | **收窄到自家域名**，仅清除、不读取，合规 |
`storage` | 保存用户设置（如 `redirectNewTab`） | 合规 |

**内容脚本**只匹配 `limestart.cn` 自家域名；`web_accessible_resources` 仅暴露壁纸文件，且同样限定自家域名。

**需知悉的行为（非风险项）**：`chrome_settings_overrides.homepage` 会把浏览器主页设为 `https://www.limestart.cn/?from=ext`（自家域名 + 自家 `from=ext` 标记，非第三方返利域）。这是起始页类扩展的常规声明项，且在安装时由浏览器呈现。

---

## 六、Chrome 版与 Edge 版比对（逐字节）

| 比对项 | 结果 |
|---|---|
仅 Chrome 版有 | 无 |
仅 Edge 版有 | `.DS_Store`、`_locales/.DS_Store`、`.web-extension-id`（打包残留，非代码） |
共有文件 | 21 个 |
内容不同的文件 | `manifest.json`、`_metadata/verified_contents.json` |
`manifest.json` 差异 | **仅** `update_url`（Chrome→`clients2.google.com`，Edge→`edge.microsoft.com`） |

→ 全部可执行代码（`scripts/*.js`、`*.html`、`*.css`、图片、壁纸）**逐字节相同**。详见 `raw/04-diff-chrome-vs-edge.txt`。

---

## 七、能力边界（必须与结论一并表述）

1. 该扩展的**核心界面由远程站点 `limestart.cn` 渲染**（`newtab-ext.html` 内嵌 iframe），包内静态检查无法覆盖该页面未来的运行时内容；
2. 但作为网页，它**无法**读取浏览器历史、**无法**读取 Cookie、**无法**向其他网站注入脚本、**无法**改写其他站点的链接、**无法**修改浏览器默认搜索引擎；
3. 上述能力差异，是它与存在推广注入行为的扩展（如 iTab）之间的**结构性区别**——后者普遍申请 `scripting` + `<all_urls>` 或 `webRequest`，并内置跳转域与推广码。

---

## 八、如何自行验证

```bash
# 1. 下载原始包并校验哈希（应与本报告一致）
curl -L -o lime.crx "https://clients2.google.com/service/update2/crx?response=redirect&prodversion=120.0&acceptformat=crx2,crx3&x=id%3Daajmeahnoefldipnjgodkobfdcpmapno%26uc"
curl -L -o lime-edge.crx "https://edge.microsoft.com/extensionwebstorebase/v1/crx?response=redirect&prodversion=120.0.0.0&x=id%3Dpcpnigdkpcgemocnjhebmajldpjlbeom%26installsource%3Dondemand%26uc"
md5sum lime.crx lime-edge.crx          # 2936aeef… / 7990e470…
sha256sum lime.crx lime-edge.crx

# 2. 解压（CRX3 = 文件头 + ZIP）
python -c "import zipfile,io;raw=open('lime-edge.crx','rb').read();i=raw.find(b'PK\x03\x04');zipfile.ZipFile(io.BytesIO(raw[i:])).extractall('lime-edge')"

# 3. 看权限与内容脚本范围（确认无 history / cookies）
python -c "import json;m=json.load(open('lime-edge/manifest.json',encoding='utf-8'));print(m['permissions']);print(m['content_scripts'])"

# 4. 特征扫描（应全为 0）
grep -rEo 'jd\.com|taobao\.com|tmall\.com|1688\.com|coml\.cc|tn=[A-Za-z0-9_]+' lime-edge/

# 5. 确认没有读取历史的 API
grep -rn 'chrome\.history\|browsingHistory\|chrome\.cookies\|document\.cookie' lime-edge/scripts/

# 6. 查看 browsingData 的调用是否收窄到自家域名
grep -o 'browsingData.remove([^)]*)' lime-edge/scripts/popup-ext.js
```

`ext/` 目录为 Edge 版解包源码（与 Chrome 版逐字节相同），可直接用于上述 grep 复核。

---

## 九、说明与免责

- 本结论**仅适用于 1.4.0 版本、上表所列 MD5 的安装包**；扩展更新后需重新审查；
- 本报告不涉及任何个人信息，不对开发者作任何主观评价；
- 文中提及的第三方品牌/域名仅为技术描述，不构成指控。
