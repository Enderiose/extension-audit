# iTab 问题证据包 · 清单

**被检查对象**：iTab 新标签页扩展
- Edge 商店版 `inedkoakiaeepjoblbiiipedngonadhn`，版本 **2.2.25**（本机此前实际安装的版本）
- Chrome 商店版 `mhloojimgilafopcmlcikiidgbbnelip`，版本 **2.3.13**（当前最新版）

**取证时间**：2026-09-26 ｜ **取证方式**：从两家的官方应用商店下载原始安装包后做静态分析 + 对外可重复的网络请求

## 文件一览

| 文件 | 说明 |
|---|---|
`AIGC.png` | 对话截图（1900×1365） |
`iTab-曝光说明-公开版.md` | **主文档**：事实、证据、复现步骤、诉求（可直接对外发布） |
`举报材料-12315.md` | 向市场监管部门举报"未标示广告"的正式材料模板 |
`举报材料-扩展商店.md` | 向 Chrome / Edge 商店举报扩展滥用的中英双语材料 |
`itab-edge.crx` | Edge 官方商店原始包 v2.2.25（3,321,529 bytes，未做任何修改） |
`itab-chrome.crx` | Chrome 官方商店原始包 v2.3.13（3,610,151 bytes，未做任何修改） |
`raw/00-hashes.txt` | 两个原始包的 MD5 / SHA256 与权限信息 |
`raw/01-default-nav-icons.txt` | 默认导航里电商图标的真实 URL（原文） |
`raw/02-rewrite-search-url.txt` | 强制改写搜索引擎链接的代码（原文） |
`raw/03-promo-codes.txt` | 包内 `tn=` 推广码出现位置全量列表 |
`raw/04-redirect-302.txt` | 302 跳转的完整响应 |
`raw/05-edge-package-2.2.25.txt` | Edge 版（本机实际安装版本）的对应证据 |
`shots/01-default-nav-icons.png` | 证据图 1：默认导航里电商图标的真实 URL |
`shots/02-redirect-302.png` | 证据图 2：`jd.coml.cc` 实测 302 跳转至 360 推广服务 |
`shots/03-rewrite-search-url.png` | 证据图 3：代码强制改写搜索引擎链接 |
`shots/04-all-promo-codes.png` | 证据图 4：包内出现的全部推广码 |
`shots/05-local-machine-proof.png` | 证据图 5：本机实测（历史记录反复出现推广地址） |
`shots/06-edge-package-2.2.25.png` | 证据图 6：Edge 版（本机实际安装版本）同样如此 |

> 私用备注与重复文件已移至包根目录的 `_内部资料（勿上传）/`，**不要上传**。

## 一分钟自证

```bash
# 1. 从官方商店下载原始包（Chrome 版）
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
