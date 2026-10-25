# 空境 · 新标签页 审查报告（Chrome）

**扩展 ID**：`hiikbcjfgbpnbnkgddndechehhhmgfbg`
**被检查版本**：2.1.1（Chrome 官方商店原始包）
**哈希**：

| 商店 | 包 | 大小 | MD5 | SHA256 |
|---|---|---|---|---|
| Chrome | `kongjing-chrome.crx` | 278,724 B | `4990456f8fadf816ab9b6a9cd336b3cc` | `7fd6d818281c1d7db99c6dd0f63734035bd592e61900e7611ca9058fe0b9f008` |

**取证时间**：2026-10-26
**取证方式**：Chrome 商店接口下载原始包 → 静态分析（S0–S3）→ 网络侧核验（S4）→ 包完整性复算
**审查级别**：**A — 未发现推广注入或返利劫持行为**

> 流程依据：根目录 `../AGENTS.md`（S0–S6）；参考规范：`../docs/SPEC.md`。
> Edge 版本次未取得（Edge 商店检索页动态渲染，未返回可核对的条目），结论仅针对上表 Chrome 包。

---

## 一、结论

在本次检查的 **2.1.1 版本**中，**未发现**推广注入、返利跳转、推广码下发、搜索引擎改写、导航链接改写、指纹采集或浏览记录读取行为。

- 新标签页是**完全本地**的页面（`index.html` + 本地打包 JS，无 iframe、无远程脚本，CSP `script-src 'self'`）；
- 默认书签只有 6 个常规站点，包内**无电商域、无仿冒域、无点击跟踪路径**；
- 搜索走 `chrome.search.query`（浏览器默认引擎），且 `chrome_settings_overrides` 未声明，**结构上无法修改默认搜索引擎**；
- 包内代码的自动外发请求只有两类："书签域名→图标代取"和"用户主动配对后的端到端加密同步"；前者已在官方隐私政策中披露。

> 补充说明：该扩展**不经服务端下发配置改写任何链接**；同步数据在设备端以 AES-256-GCM 加密后上传，服务端持有的是密文信封。

---

## 二、检查项目与结果

| 检查项目 | 结果 |
|---|---|
默认导航/快捷方式中的电商链接 | **0 处**（无 jd.com / taobao.com / tmall.com / 1688.com / pinduoduo / suning） |
仿冒域（把官方域名当子域前缀） | **0 处** |
第三方跳转域与 302 链路 | **0 处**（无 `/clk`、`/click`、`/track`、`/union`、`/cps`、`/tg?`） |
推广码（`tn=` / `pid=` / `unionid=` / `cps=` 等） | **0 个**（扫描命中 14 处，全部为 React `ref=` 与局部变量 `const tn=`，见 `raw/04-suspicious-hits.txt`） |
服务端配置改写链接 | **无**（无"拉取远程配置 → 覆写 href / 搜索引擎"的代码路径） |
搜索引擎改写 | **无**（`chrome_settings_overrides` 未声明；默认搜索调用系统引擎） |
指纹 / 位置采集 | **0 处**（无 canvas.toDataURL、无 WebGL、无 navigator.plugins/screen 枚举；`devicePixelRatio` 仅用于界面渲染；`deviceId` 为同步协议的设备公钥短 ID） |
动态执行（`eval` / `new Function`） | **0 处** |
远程脚本注入（`createElement('script')`） | **0 处** |
`atob`（3 处） | 密文信封 / PoW 挑战的 base64url 解码，输入来自扩展自身数据 |
`insertAdjacentHTML`（1 处） | `bar.js` 写入写死的内联 SVG 图标 |
`href=` 赋值（5 处） | 用户点击书签直接跳转、导出备份、Vite 预加载 link、React 内部，**无静默改写** |
浏览记录读取（`chrome.history`） | **0 处**（未申请 `history`） |
Cookie 读取（`chrome.cookies` / `document.cookie`） | **0 处**（未申请 `cookies`） |
远程代码加载（`importScripts` / 远程 script / 动态 import 远程 URL） | **无**（动态 `import()` 仅加载包内 chunk） |
权限与功能匹配度 | **匹配**（见第四节；宽口径权限为可选、默认关闭） |
包完整性 | `_metadata/verified_contents.json` 复算 **23/23 文件一致** |

---

## 三、关键证据

### 3.1 本地新标签页（无远程界面）

`manifest.json`：

```json
"chrome_url_overrides": { "newtab": "index.html" },
"content_security_policy": { "extension_pages": "script-src 'self'; object-src 'self'; base-uri 'self'" }
```

`index.html` 只引用包内 `boot.css / boot.js / assets/*.js`，无 `<iframe>`、无远程 `<script src>`。

### 3.2 默认书签（`background.js`）

```js
bookmarks:[["github","GitHub","https://github.com"],
           ["bilibili","哔哩哔哩","https://www.bilibili.com"],
           ["zhihu","知乎","https://www.zhihu.com"],
           ["youtube","YouTube","https://www.youtube.com"],
           ["gmail","Gmail","https://mail.google.com"],
           ["sspai","少数派","https://sspai.com"]]
```

均为站点首页，无跳转域、无渠道参数、无电商链接。

### 3.3 搜索（`assets/index-*.js`）

```js
if(e.id!=="default") return Se(e.url.replace("%s",encodeURIComponent(r)), a);
await chrome.search.query({text:r, disposition:a?"NEW_TAB":"CURRENT_TAB"});
```

默认引擎交给浏览器；内置预设只有 Google / 必应 / 百度 / DuckDuckGo，均为官方搜索地址、**无 `tn=` 等渠道参数**（`assets/v2-*.js`）：

```js
{id:"google",url:"https://www.google.com/search?q=%s"},
{id:"bing",url:"https://www.bing.com/search?q=%s"},
{id:"baidu",url:"https://www.baidu.com/s?wd=%s"},
{id:"duckduckgo",url:"https://duckduckgo.com/?q=%s"}
```

### 3.4 同步数据在设备端加密（`background.js`）

```js
// HKDF(SHA-256) 派生密钥，AES-GCM 加密，信封 = version|iv|ct
async function it(o,t,e,s){const r=await Z(o,await ge(e),A(t),s);
  return{v:1,iv:E(r.iv),ct:E(r.ct)}}
// journal / snapshot / blob 三条上传路径全部走密文信封
const yt=(o,t,e,s,r)=>it(o,"kj/j/"+t+"/"+e,A(JSON.stringify(s)),r);
async function H(o,t,e){if(e?.v!==1)throw ...; return me(await tt(o,b(e.iv),b(e.ct),A(t)))}
```

上传体示例（`journal`）：

```js
body: JSON.stringify({ items: a })   // a = [{id, env:{v,iv,ct}}]，env 为密文
```

→ 服务端只能看到房间/设备标识、密文、大小与时间；**看不到书签 URL、标题、设置内容**。未配对时无同步请求。

### 3.5 图标来源（唯一的明文域名外发）

```js
// background.js
const Ct = o => pt() ? chrome.runtime.getURL(`_favicon/?pageUrl=${encodeURIComponent(o)}&size=64`) : "";
// 本地缓存命中则用本地；缺失时回退：
return await r(`${t.replace(/\/$/,"")}/v1/icon?host=${encodeURIComponent(new URL(o).hostname)}`)
```

- 只发送 **hostname**（如 `github.com`），不含路径、标题、Cookie；请求为普通 GET、无认证头、`credentials:"omit"`；
- 实测（`raw/09-network-verify.txt`）：`GET https://kongjing.inc.re/v1/icon?host=github.com` → `200 image/x-icon`，无 302；
- 官方隐私政策第 2 节已明确披露：**"缺失时将书签网站的域名（例如 github.com，不含页面路径）发给所选图标服务"**。

### 3.6 空境栏（可选权限）的注入范围（`background.js`）

```js
chrome.scripting.registerContentScripts([{
  id:"kongjing-bar", js:["bar.js"],
  matches:["http://*/*","https://*/*"],
  excludeMatches:["https://chromewebstore.google.com/*",
                  "https://chrome.google.com/webstore/*",
                  "https://microsoftedge.microsoft.com/addons/*"],
  runAt:"document_idle", allFrames:false
}]);
```

前置条件：`settings.barEnabled && chrome.permissions.contains({origins:["http://*/*","https://*/*"]})`；
关闭或权限被收回时立即 `unregisterContentScripts`。

`bar.js`（全文约 12 KB）只做三件事：创建 `<kongjing-bar>` + closed shadow DOM、渲染收藏项、在用户点击时 `location.href = e.url`。
不读取页面正文，不改页面内链接，不监听输入。
加载时会用 `sender.tab.url` 与本地书签比对，用于显示"是否已收藏"；用户点击"收藏本页"时才读取 `location.href` 与 `document.title`。

---

## 四、权限与用途对照

| 权限 | 类型 | 实际用途 | 评价 |
|---|---|---|---|
`storage` | 必需 | 本地保存书签/设置/配对信息（`chrome.storage.local`；**无** `storage.sync`） | 合规 |
`favicon` | 必需 | 本地 `_favicon` 读缓存图标 | 合规（该权限本身不触网；缺失时的回退见 3.5） |
`activeTab` | 必需 | 上下文菜单"收藏此页/此链接"时读取当前页 URL | 合规 |
`contextMenus` | 必需 | 2 个右键菜单项 | 合规 |
`alarms` | 必需 | 1 分钟心跳驱动同步重试 | 合规 |
`search` | 必需 | `chrome.search.query` 用系统默认引擎搜索 | 合规；不改默认引擎 |
`host: kongjing.inc.re` | 必需 | 同步中继 + 图标代取 | 单一域名，合规 |
`scripting` + `http(s)://*/*` | **可选** | 仅在开启"空境栏"后注入 `bar.js` | 宽口径但**默认关闭**、需用户授权、行为受限；见 3.6 |

**未申请**：`history`、`bookmarks`、`cookies`、`tabs`、`webRequest`、`declarativeNetRequest`、`proxy`、`management`、`nativeMessaging`、`downloads`。
**未声明**：`chrome_settings_overrides`、`externally_connectable`、`web_accessible_resources`。
**内容脚本**：manifest 中无 `content_scripts`；全站注入只可能通过上述可选权限临时注册。

---

## 五、与"推广注入 / 返利劫持"模式的对照

| 模式特征（见 `../docs/SPEC.md` 1.2） | 空境 2.1.1 |
|---|---|
内置电商链接指向自有跳转域（如 `*.coml.cc`） | 无（默认书签仅 6 个常规站点） |
跳转域 302 至点击跟踪服务（`/browser/clk` 等） | 无（自动请求仅同步中继与图标服务，且均无跳转；其余外链均为用户点击） |
服务端下发推广码并 `e.href=…tn=${o}` 强制改写搜索 | 无（`chrome.search.query`，无参数追加） |
远程 JS 加载并执行 | 无（CSP `'self'`，无 `createElement('script')`） |
采集指纹/城市上报 | 无（无 canvas/WebGL/UA 上报；UA 仅用于生成平台标签） |
按渠道分配多个推广码 | 无（0 个推广码） |

与 iTab 案例的结构性差异：iTab 同时具备"内置仿冒跳转域 + 302 跟踪 + 服务端下发的 `tn` 推广码 + `e.href=` 改写"；空境不具备其中任何一项，且新标签页内容完全随包分发，不存在远程页面可变的风险面。

---

## 六、需要知悉的行为（非风险项）

1. **图标代取会向中继发送书签域名（明文）**：仅 hostname，不含路径/标题；代码中该路径不依赖配对状态（启动时补全缺失图标即会触发）。官方隐私政策已单列说明；同步设置中可将中继替换为自建服务（图标请求同样走所选地址）。
2. **可选权限口径较宽**：`scripting` + `http(s)://*/*` 可读取任意站点内容；但代码实际只注入 `bar.js` 渲染空境栏，且默认关闭、需 `chrome.permissions.request` 明确授权，可随时关闭/按站点隐藏。
3. **E2E 的边界**：端到端加密只覆盖同步内容；图标域名查询、连接元数据（IP、时间、房间/设备标识）不属于加密范围——隐私政策已如实写明。
4. **包内残留旧文案**：`assets/index-*.js` 仍保留一段"空境会直接访问你收藏的网站首页…不会上传任何数据"的图标权限说明，与当前实现（本地缓存 → 中继代取）不符；该 dialog 分支在本版本无调用点，属文案遗留，建议开发者清理。

---

## 七、能力边界（必须与结论一并表述）

- 本报告只覆盖**静态包**：包内没有远程配置下发、没有远程脚本加载，因此静态结论的覆盖面较好；但同步中继（`kongjing.inc.re`）与图标服务属于服务端，其日志与保留策略不在包内可验证范围；
- 该扩展**不具备**推广注入所需的结构性能力：不能改默认搜索引擎（未声明 `chrome_settings_overrides`），不能拦截/重写网络请求（未申请 `webRequest` / `declarativeNetRequest`），未授权时不能向第三方页面注入脚本；
- 结论**仅对 2.1.1 / 上述哈希**有效；扩展更新后需重新审查。

---

## 八、如何自行验证

完整命令见 `raw/05-verify-cmds.txt`。最短路径：

```bash
# 1. 下载并校验
curl -L -o kongjing-chrome.crx \
  "https://clients2.google.com/service/update2/crx?response=redirect&prodversion=120.0&acceptformat=crx2,crx3&x=id%3Dhiikbcjfgbpnbnkgddndechehhhmgfbg%26uc"
md5sum kongjing-chrome.crx
# 4990456f8fadf816ab9b6a9cd336b3cc

# 2. 解包
python -c "import zipfile,io;raw=open('kongjing-chrome.crx','rb').read();i=raw.find(b'PK\x03\x04');zipfile.ZipFile(io.BytesIO(raw[i:])).extractall('ext')"

# 3. 关键扫描（应全为 0）
grep -rEo 'jd\.com|taobao\.com|tmall\.com|1688\.com|coml\.cc|/clk|/union' ext/ || echo NONE
grep -rEo 'chrome\.history|chrome\.cookies|document\.cookie' ext/ || echo NONE
grep -rEo "createElement\(['\"]script|eval\(|new Function\(" ext/ || echo NONE

# 4. 确认默认书签与搜索行为
grep -o 'bookmarks:\[\["github"[^]]*\]' ext/background.js
grep -o '.\{60\}chrome\.search\.query.\{100\}' ext/assets/index-*.js

# 5. 验证图标代取端点（无用户数据）
curl -s -o /dev/null -w '%{http_code} %{content_type}\n' \
  "https://kongjing.inc.re/v1/icon?host=github.com"
# 200 image/x-icon

# 6. 包完整性（应输出 ok 23 bad 0）
python - <<'PY'
import json,base64,hashlib
d=json.load(open('ext/_metadata/verified_contents.json'))
p=json.loads(base64.urlsafe_b64decode(d[0]['signed_content']['payload']+'==='))
bs=p['content_hashes'][0]['block_size']; b=lambda x: base64.urlsafe_b64encode(x).decode().rstrip('=')
ok=bad=0
for f in p['content_hashes'][0]['files']:
    data=open('ext/'+f['path'],'rb').read()
    hs=[hashlib.sha256(data[i:i+bs]).digest() for i in range(0,len(data),bs)]
    root=b(hs[0]) if len(hs)==1 else b(hashlib.sha256(b''.join(hs)).digest())
    ok+=root==f['root_hash']; bad+=root!=f['root_hash']
print('ok',ok,'bad',bad)
PY
```

---

## 九、说明与免责

- 本结论**仅适用于 2.1.1 版本、上表所列哈希的 Chrome 安装包**；扩展更新后需重新审查；
- 文中提及的服务端域名与第三方品牌仅为技术描述，不构成指控；
- 本报告不涉及个人信息，不对开发者作主观评价；
- 原始安装包只读保存，未执行包内代码；网络核验未携带任何用户 Cookie 或账号信息。
