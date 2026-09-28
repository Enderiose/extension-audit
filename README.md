# 浏览器扩展审查 · 证据包

[English](README_EN.md) ｜ **简体中文**

> 对浏览器扩展（新标签页 / 起始页 / 工具类）的**推广注入、返利劫持、隐蔽采集**行为进行可复现取证，
> 并沉淀出一套**可复用的审查标准流程**。
> 所有结论**只陈述可用命令复现的事实**，并绑定被检查的版本与哈希。

仓库：https://github.com/Enderiose/extension-audit

---

## 一、这是什么

一个**浏览器扩展审查证据包**：包含两个真实案例的完整证据（官方原始安装包、代码片段、抓包、截图、哈希），以及一套面向 AI 代理与人类审查员的 **S0–S6 标准审查流程**。

产出物是文档与证据，**不是可执行程序**。目录内全部使用相对路径，可整体移动到任意位置。

---

## 二、案例结论

| 扩展 | 被检查版本 / 哈希 | 判定 | 一句话结论 |
|---|---|---|---|
| **iTab 新标签页** | Chrome 2.3.13<br>`247d7106…`<br>Edge 2.2.25<br>`a8b1a8f9…` | **D — 确认返利劫持 + 推广注入** | 默认导航的电商图标指向自有跳转域 `*.coml.cc`，实测 302 至 `extdd1.browser.360.cn/browser/clk`；服务端下发推广码并强制改写搜索引擎链接；采集浏览器指纹与城市 |
| **青柠起始页** | Chrome 1.4.0<br>`2936aeef…`<br>Edge 1.4.0<br>`7990e470…` | **A — 未发现问题** | 无电商域、无仿冒域、无推广码、无跟踪跳转；权限仅 4 项且用途收窄（清缓存限定自家域名）；两版代码逐字节相同；未申请 `history`，无浏览记录读取 |

> 判定分级：**A** 未发现问题 ／ **B** 存在风险点 ／ **C** 确认推广注入 ／ **D** 确认返利劫持（定义见 [`AGENTS.md`](AGENTS.md) 第 4 节）。
>
> 结论**仅适用于上表标注的版本与哈希**；扩展更新后需重新审查。

---

## 三、仓库结构

```
extension-audit/
├── README.md / README_EN.md    ← 中英文说明（互链）
├── AGENTS.md                   ★ 通用扩展审查标准流程（S0–S6）
├── docs/
│   └── SPEC.md                 参考规范：特征库 / 权限矩阵 / 误报清单 / 基准案例
├── 00-总索引.md                 全包总索引
│
├── 01-iTab-确认推广注入/        案例一（D 级）
│   ├── iTab-曝光说明-公开版.md     对外发布主文档
│   ├── 举报材料-12315.md          向市场监管部门举报
│   ├── 举报材料-扩展商店.md       向 Chrome / Edge 商店举报（中英双语）
│   ├── itab-chrome.crx / itab-edge.crx   官方原始包（只读）
│   ├── raw/                       原始文本证据
│   └── shots/                     代码 / 抓包截图
│
└── 02-青柠起始页-未发现问题/     案例二（A 级，Chrome + Edge）
    ├── REPORT.md                  统一审查报告
    ├── lime.crx / lime-edge.crx / lime-site.html
    ├── ext/                       解包源码（两版逐字节相同）
    └── raw/                       哈希 / manifest / 扫描 / 比对 / 权限
```

> 内部资料目录（`_内部资料（勿上传）/`）**已通过 `.gitignore` 排除**，不包含在本公开库中。

---

## 四、怎么用

| 目的 | 文件 |
|---|---|
| 审查一个新的扩展 | 按 [`AGENTS.md`](AGENTS.md) 的 **S0–S6** 流程执行（含可直接复制的命令） |
| 查阅特征库 / 权限矩阵 / 误报 / 基准案例 | [`docs/SPEC.md`](docs/SPEC.md) |
| 看 iTab 案例的完整证据 | [`01-iTab-确认推广注入/`](01-iTab-确认推广注入/) |
| 看青柠起始页的审查报告 | [`02-青柠起始页-未发现问题/REPORT.md`](02-青柠起始页-未发现问题/REPORT.md) |
| 全包导航 | [`00-总索引.md`](00-总索引.md) |

---

## 五、快速自证（示例）

### iTab（Chrome 2.3.13）

```bash
# 1. 从 Chrome 官方商店下载原始包
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

### 青柠起始页（Chrome 1.4.0）

```bash
curl -L -o lime.crx "https://clients2.google.com/service/update2/crx?response=redirect&prodversion=120.0&acceptformat=crx2,crx3&x=id%3Daajmeahnoefldipnjgodkobfdcpmapno%26uc"
md5sum lime.crx          # 应为 2936aeef53a1c9bb079bfda3cf7e93d3
python -c "import zipfile,io;raw=open('lime.crx','rb').read();i=raw.find(b'PK\x03\x04');zipfile.ZipFile(io.BytesIO(raw[i:])).extractall('lime')"
grep -rEo 'jd\.com|taobao\.com|coml\.cc|tn=[A-Za-z0-9_]+' lime/   # 应为空
```

---

## 六、审查原则

- **只写可复现的事实**：每条结论附文件路径 + 代码片段 + 可执行命令；
- **结论绑定版本与哈希**：版本更新后旧结论自动失效；
- **措辞可证伪**：用「指向 / 302 跳转到 / 强制改写 / 未标注」，不用「偷 / 恶意 / 故意」；
- **不做人身攻击**：只评价行为，不评价个人，不公布个人信息；
- **原始包只读**：下载后立即记录哈希，永不修改。

---

## 七、说明与免责

- 所有结论**仅适用于文中标注的版本与哈希**；扩展更新后需重新审查；
- 本仓库不含任何个人信息，不对任何开发者作主观评价；
- 文中提及的第三方公司 / 品牌 / 域名仅为**技术描述与链路说明**，不构成指控；
- 本仓库仅用于技术研究与消费者知情，不构成法律意见。

---

[English](README_EN.md) ｜ **简体中文**
