# SPEC.md — 扩展审查参考规范（非流程）

> 本文件收纳**非流程性**的参考数据与领域知识：特征库、权限矩阵、误报清单、基准案例等。
> **审查流程本身见根目录 `../AGENTS.md`**。

---

## 1. 检测特征库（含扫描脚本）

### 1.1 扫描脚本（S2 参考实现）

对包内全部 `.js / .html / .json / .css` 做特征扫描：

```bash
python - <<'PY'
import zipfile,io,re,collections
raw=open('ext.crx','rb').read(); i=raw.find(b'PK\x03\x04')
z=zipfile.ZipFile(io.BytesIO(raw[i:]))
t='\n'.join(z.read(n).decode('utf-8','replace') for n in z.namelist() if n.endswith(('.js','.html','.json','.css')))
PATS={
 '电商域':       r'(jd\.com|taobao\.com|tmall\.com|1688\.com|pinduoduo|suning|vip\.com)',
 '仿冒域':       r'\b[a-z0-9\-]+\.(?:coml|cm|con|orn|om)\.(?:cc|cn|com|net)\b',
 '推广码参数':   r'\b(?:tn|pid|unionid|union_id|aff|affid|cps|ref|spm|scm|ali_refid|ali_trackid)=',
 '点击/跳转路径':r'(/clk|/click|/track|/redirect|/jump|/goto|/union|/cps|/tg\?)',
 '指纹采集':     r'(fingerprint|canvas\.toDataURL|webgl|deviceId|fp_?id)',
 '动态执行':     r'(eval\s*\(|new\s+Function\s*\(|atob\s*\()',
 '远程脚本注入': r'(createElement\([\'"]script|insertAdjacentHTML)',
 'href 改写':    r'\.href\s*=\s*',
 '历史记录读取': r'(chrome\.history|browsingHistory|\bhistory\.search\b)',
 'cookie 读取':  r'(chrome\.cookies|document\.cookie)',
}
for k,p in PATS.items():
    m=re.findall(p,t); print(f'{k}: {len(m)}' + (f'  {sorted(set(m))[:8]}' if m else ''))
print('--- 远程域名 ---')
for d,n in collections.Counter(re.findall(r'https?://[a-zA-Z0-9\.\-]+\.[a-z]{2,}',t)).most_common(30):
    print(f'  {n:>4}× {d}')
PY
```

**默认导航/书签配置**：优先找 `defaultNavConfig*`、`shortcuts*`、`nav*`、`bookmark*`、`bookmarks.html` 等文件——它们最常藏推广 URL。

### 1.2 致命（单独成立即可判定为推广注入/劫持）

- 内置默认导航/快捷方式中存在**非官方域名的电商链接**（如 `jd.coml.cc` 这类把官方域名当子域前缀的仿冒形态）
- 上述链接 **302 跳转到第三方点击跟踪服务**（`*/clk`、`*/union/*` 等）
- **服务端下发推广码并覆写链接**：形如
  ```js
  o = s.tn || "<硬编码推广码>"      // 推广码由服务器分配，兜底值硬编码
  e.href = `https://www.baidu.com/s?wd=%s&tn=${o}`   // 直接赋值改写
  s(d).then(...)                    // d 内含指纹/城市等上报字段
  ```
- 存在**远程 JS 加载并执行**（`createElement('script')` + `eval`/`new Function`）

### 1.3 高危（需结合上下文，通常共现）

- 同一包内出现**多个不同推广码**（说明按渠道/用户分配）
- 采集浏览器指纹 / 城市 / 时区等并向非官方域上报
- 上报接口带 `fp=`、`deviceId=`、`tn=` 等字段
- 对新标签页/搜索引擎的跳转做**静默重定向**（未通过用户点击）

### 1.4 中危（需评估是否与声明功能相符）

- 权限与功能不匹配：新标签页类扩展申请 `scripting` / `webRequest` / `tabs` / `<all_urls>`
- 内容脚本注入**第三方站点**（`matches` 含 `*://*/*` 或 `<all_urls>`）
- 混淆：`_0x[0-9a-f]{4,}` 大量出现、base64 大字符串 + `atob`
- `externally_connectable` 允许外部站点驱动扩展逻辑

---

## 2. 权限风险矩阵（速查）

| 权限 | 风险 | 合规用途示例 | 需警惕的信号 |
|---|---|---|---|
`storage` | 低 | 保存配置 | 保存推广码、上报队列 |
`activeTab` | 低 | 对当前标签操作 | 配合 `scripting` 注入任意站点 |
`commands` | 低 | 快捷键 | — |
`bookmarks` | 中 | 收藏同步 | 读取并**改写**用户书签 |
`tabs` | 中 | 管理标签 | 批量重定向、替换新标签 |
`browsingData` | **中高** | 提供"清理本站缓存" | 全局 `remove({},...)`（不限定 origin）即高危 |
`history` | **中高** | 读取浏览历史（需明示用途） | 读取历史并上报 |
`webRequest` | **高** | — | 拦截/重写请求头与 URL |
`declarativeNetRequest` | **高** | — | 规则集重定向流量 |
`scripting` + `<all_urls>` | **最高** | — | 可向任意页面注入脚本，可改页面内链接 |
`proxy` | **最高** | — | 可接管全部网络流量 |

> **判定要点：看用途是否收窄。**
> - `browsingData.remove({hostnames:["own.site"]}, {cache:true})` → 合规（**注意：`browsingData` 只能"清除"，不能"读取"**；读取历史需 `history` 权限）。
> - `browsingData.remove({}, {cache:true,cookies:true})` → 全局清除，需重点质疑。

---

## 3. 常见误报（不得作为判定依据）

- `w3.org` / `schema.org` 命名空间
- `alicdn.com` 等**静态资源 CDN**（字体、图标）
- `chrome.google.com` / `microsoftedge.microsoft.com` / `addons.mozilla.org`（商店互推链接）
- 指向**自家域名**的 `?from=` 来源标记
- 用户在弹窗里**主动点击**才打开的外链
- 把权限名 `browsingData`（"清除浏览数据"）误读为"读取浏览记录"——二者不同

---

## 4. 基准案例（判定尺度校准）

| 案例 | 扩展 ID | 版本 / 哈希 | 关键事实 | 判定 |
|---|---|---|---|---|
**iTab 新标签页** | Chrome `mhloojimgilafopcmlcikiidgbbnelip`<br>Edge `inedkoakiaeepjoblbiiipedngonadhn` | Chrome 2.3.13 / MD5 `247d71066852e38d74a64d9bee31033e`<br>Edge 2.2.25 / MD5 `a8b1a8f9e7dec0ed89dc68785ba24462` | 默认导航 `taobao/jd/tmall/1688 → *.coml.cc`，实测 302 至 `extdd1.browser.360.cn/browser/clk`；代码中 `o = s.tn \|\| "52176495_dg&ch=2"` 后 `e.href = ...tn=${o}` 强制改写；上报 `fp`+`city`；权限含 `scripting` + 可选 `<all_urls>`；包内 3 个不同推广码 | **D** |
**青柠起始页** | Chrome `aajmeahnoefldipnjgodkobfdcpmapno`<br>Edge `pcpnigdkpcgemocnjhebmajldpjlbeom` | Chrome 1.4.0 / MD5 `2936aeef53a1c9bb079bfda3cf7e93d3`<br>Edge 1.4.0 / MD5 `7990e470de73852d188b09f84ba826c7` | 无任何电商域/仿冒域/推广码/跟踪路径；权限仅 `activeTab, browsingData, commands, storage`；内容脚本仅注入自家域名；`browsingData` 收窄为自家域名；未申请 `history`；新标签页为自家站点 iframe；全部外链均为用户主动点击；两版代码逐字节相同 | **A** |

---

## 5. 远程内容类扩展的附加要求

若扩展的"新标签页/主页"是**远程站点的 iframe**（如 `newtab.html` 内嵌 `https://<vendor>/`）：

1. 包内代码干净**不等于**页面内容永久干净——远程页面可随时变更；
2. 但**必须同时说明其能力边界**：作为网页，它**无法**注入其他网站、无法改写其他站点的链接、无法修改浏览器默认搜索引擎、无法读取浏览器历史或 Cookie；
3. 报告应补充一句："该扩展的核心内容由远程站点渲染，包内静态检查无法覆盖其运行时界面；如需完全确认，需持续观察其页面内容。"；
4. 附加检查：抓取该远程站点首页，扫描电商域/推广码特征（作为参考，不作为判定依据）。

---

## 6. 本项目证据包

### 6.1 定位

一个**浏览器扩展推广注入与返利劫持**的**取证与举报证据包**。产出物是文档、原始安装包、原始文本证据与截图，**不是可执行程序**。工作语言中文，面向公众发布与监管举报。

### 6.2 目录结构

```
extension-audit/
├── AGENTS.md                         ← 通用扩展审查标准流程
├── docs/SPEC.md                      ← 本文件（参考规范）
├── 00-总索引.md                      ← 全包总索引，结构变更后同步更新
├── 01-iTab-确认推广注入/              ← 案例一（D 级）
│   ├── AIGC.png                      对话截图
│   ├── 00-README-证据清单.md          证据清单 + 一分钟自证
│   ├── iTab-曝光说明-公开版.md         对外主文档
│   ├── 举报材料-12315.md              向市场监管举报
│   ├── 举报材料-扩展商店.md           向 Chrome/Edge 商店举报（中英双语）
│   ├── itab-chrome.crx / itab-edge.crx  官方原始包（只读）
│   ├── raw/  00–05 原始文本证据
│   └── shots/ 01–06 代码/抓包截图
├── 02-青柠起始页-未发现问题/          ← 案例二（A 级，Chrome + Edge）
│   ├── REPORT.md                     统一审查报告
│   ├── lime.crx / lime-edge.crx / lime-site.html
│   ├── ext/                          解包源码（两版逐字节相同，共用一份）
│   └── raw/  哈希 / manifest / 扫描 / 比对 / 权限
└── _内部资料（勿上传）/              ★ 对外分享时整个目录排除
    ├── 私用-素材与文案（勿公开）.md
    ├── Edge 策略加固记录.md
    └── 用户自行下载/                 lime.crx 的重复副本（可删）
```

### 6.3 关键事实速查（勿凭记忆改写）

| 对象 | 版本 / 哈希 | 判定 |
|---|---|---|
iTab（Chrome `mhloojimgilafopcmlcikiidgbbnelip`） | 2.3.13 / MD5 `247d71066852e38d74a64d9bee31033e` | **D** |
iTab（Edge `inedkoakiaeepjoblbiiipedngonadhn`） | 2.2.25 / MD5 `a8b1a8f9e7dec0ed89dc68785ba24462` | **D** |
青柠起始页（Chrome `aajmeahnoefldipnjgodkobfdcpmapno`） | 1.4.0 / MD5 `2936aeef53a1c9bb079bfda3cf7e93d3` | **A** |
青柠起始页（Edge `pcpnigdkpcgemocnjhebmajldpjlbeom`） | 1.4.0 / MD5 `7990e470de73852d188b09f84ba826c7` | **A** |

iTab 核心事实：默认导航电商图标指向 `*.coml.cc` → 302 至 `extdd1.browser.360.cn/browser/clk`；`statistics-*.js` 采集 `fp`+`city`，服务端下发 `tn` 推广码并 `e.href=` 强制改写搜索链接（兜底 `52176495_dg&ch=2`）；包内 3 个推广码。

### 6.4 对外发布注意

- 主文档：`01-iTab-确认推广注入/iTab-曝光说明-公开版.md`
- 举报：`举报材料-12315.md`、`举报材料-扩展商店.md`
- 发布前排除 `_内部资料（勿上传）/`；发帖注意事项见该目录内私用文档。
- 结构变更后更新 `00-总索引.md`（目录树、结论表、使用说明）保持一致。

---

## 7. 变更记录

| 日期 | 变更 |
|---|---|
2026-09-26 | 首版。基准案例：iTab（D）、青柠起始页（A） |
2026-09-26 | 增补青柠起始页 Edge 版（A）；拆分出本参考规范 `docs/SPEC.md` |
