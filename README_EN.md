# Browser Extension Audit · Evidence Pack

**English** ｜ [简体中文](README.md)

> Reproducible forensics on browser extensions (new-tab / start-page / utility) for
> **promotion injection, affiliate-link hijacking, and covert collection** —
> plus a **reusable standard audit process**.
> Every conclusion states **only facts reproducible by command**, tied to the exact version and hash checked.

Repository: https://github.com/Enderiose/extension-audit

---

## 1. What this is

An **evidence pack for browser-extension audits**: complete evidence for three real cases (official original packages, code excerpts, HTTP captures, screenshots, hashes) and a **S0–S6 standard audit process** for AI agents and human reviewers.

The deliverables are documents and evidence, **not executable software**. All documents use relative paths and can be moved anywhere.

---

## 2. Case verdicts

| Extension | Version / hash checked | Verdict | One-line finding |
|---|---|---|---|
| **iTab New Tab** | Chrome 2.3.13<br>`247d7106…`<br>Edge 2.2.25<br>`a8b1a8f9…` | **D — Confirmed affiliate hijacking + promotion injection** | Built-in shopping icons point to its own redirect domain `*.coml.cc`, which 302-redirects to `extdd1.browser.360.cn/browser/clk`; the server issues promo codes and forcibly rewrites search-engine links; browser fingerprint and city are collected |
| **Lime Start Page** | Chrome 1.4.0<br>`2936aeef…`<br>Edge 1.4.0<br>`7990e470…` | **A — No issue found** | No shopping domains, look-alike domains, promo codes, or tracking redirects; only 4 permissions, each narrowly scoped (cache clearing limited to its own domain); the two builds are byte-identical; no `history` permission, no browsing-history reads |
| **Kongjing · New Tab** | Chrome 2.1.1<br>`4990456f…` | **A — No issue found** | New-tab page is fully local (no iframe / no remote scripts); the 6 default bookmarks are ordinary sites; search uses `chrome.search.query` and never changes the default engine; sync is AES-256-GCM end-to-end encrypted; all-site access is optional, off by default, and only used for the optional bookmark bar; when a favicon is missing, the bookmark hostname is sent in plaintext to the relay, as disclosed in the privacy policy |

> Grading: **A** no issue found / **B** risk points present / **C** confirmed promotion injection / **D** confirmed affiliate hijacking (definitions in [`AGENTS.md`](AGENTS.md) §4).
>
> Verdicts apply **only to the versions and hashes listed above**; re-audit after any extension update.

---

## 3. Repository layout

```
extension-audit/
├── README.md / README_EN.md   ← bilingual docs (cross-linked)
├── AGENTS.md                  ★ Standard extension audit process (S0–S6)
├── docs/
│   └── SPEC.md                Reference spec: feature library / permission matrix / false positives / benchmark cases
├── 00-总索引.md                Master index (Chinese)
│
├── 01-iTab-确认推广注入/        Case 1 (grade D)
│   ├── iTab-曝光说明-公开版.md    Public disclosure document
│   ├── itab-chrome.crx / itab-edge.crx   Official original packages (read-only)
│   ├── raw/                     Raw text evidence
│   └── shots/                   Code / capture screenshots
│
├── 02-青柠起始页-未发现问题/    Case 2 (grade A, Chrome + Edge)
│   ├── REPORT.md                Unified audit report
│   ├── lime.crx / lime-edge.crx / lime-site.html
│   ├── ext/                     Extracted source (byte-identical across builds)
│   └── raw/                     Hashes / manifests / scan / diff / permissions
│
└── 03-空境新标签页-未发现问题/  Case 3 (grade A, Chrome)
    ├── REPORT.md                Unified audit report
    ├── kongjing-chrome.crx      Official original package (read-only)
    ├── ext/                     Extracted source (local new-tab page)
    └── raw/                     Hashes / permissions / domains / hit review / self-verify / integrity / store snapshot
```

> The internal-materials folder (`_内部资料（勿上传）/`) is **excluded via `.gitignore`** and is not part of this public repository.

---

## 4. How to use

| Goal | File |
|---|---|
| Audit a new extension | Follow **S0–S6** in [`AGENTS.md`](AGENTS.md) (commands ready to copy) |
| Feature library / permission matrix / false positives / benchmarks | [`docs/SPEC.md`](docs/SPEC.md) |
| Full iTab evidence | [`01-iTab-确认推广注入/`](01-iTab-确认推广注入/) |
| Lime Start Page report | [`02-青柠起始页-未发现问题/REPORT.md`](02-青柠起始页-未发现问题/REPORT.md) |
| Kongjing New Tab report | [`03-空境新标签页-未发现问题/REPORT.md`](03-空境新标签页-未发现问题/REPORT.md) |
| Master index | [`00-总索引.md`](00-总索引.md) |

---

## 5. Quick self-verification (examples)

### iTab (Chrome 2.3.13)

```bash
# 1. Download the original package from the Chrome Web Store
curl -L -o itab.crx "https://clients2.google.com/service/update2/crx?response=redirect&prodversion=120.0&acceptformat=crx2,crx3&x=id%3Dmhloojimgilafopcmlcikiidgbbnelip%26uc"
md5sum itab.crx          # expected: 247d71066852e38d74a64d9bee31033e

# 2. Unpack (CRX3 = header + ZIP)
python -c "import zipfile,io;raw=open('itab.crx','rb').read();i=raw.find(b'PK\x03\x04');zipfile.ZipFile(io.BytesIO(raw[i:])).extractall('itab')"

# 3. Where the built-in shopping icons point
grep -o 'https://[a-z0-9.]*coml\.cc/' itab/chunks/defaultNavConfig-*.js

# 4. Promo codes and link rewriting
grep -o 'tn=[0-9A-Za-z_]*' itab/chunks/*.js | sort -u
grep -o 'e.href=s\[e.key\]||e.href' itab/chunks/*.js

# 5. Verify the redirect target
curl -I https://jd.coml.cc/
```

### Lime Start Page (Chrome 1.4.0)

```bash
curl -L -o lime.crx "https://clients2.google.com/service/update2/crx?response=redirect&prodversion=120.0&acceptformat=crx2,crx3&x=id%3Daajmeahnoefldipnjgodkobfdcpmapno%26uc"
md5sum lime.crx          # expected: 2936aeef53a1c9bb079bfda3cf7e93d3
python -c "import zipfile,io;raw=open('lime.crx','rb').read();i=raw.find(b'PK\x03\x04');zipfile.ZipFile(io.BytesIO(raw[i:])).extractall('lime')"
grep -rEo 'jd\.com|taobao\.com|coml\.cc|tn=[A-Za-z0-9_]+' lime/   # expected: empty
```

### Kongjing · New Tab (Chrome 2.1.1)

```bash
curl -L -o kongjing.crx "https://clients2.google.com/service/update2/crx?response=redirect&prodversion=120.0&acceptformat=crx2,crx3&x=id%3Dhiikbcjfgbpnbnkgddndechehhhmgfbg%26uc"
md5sum kongjing.crx      # expected: 4990456f8fadf816ab9b6a9cd336b3cc
python -c "import zipfile,io;raw=open('kongjing.crx','rb').read();i=raw.find(b'PK\x03\x04');zipfile.ZipFile(io.BytesIO(raw[i:])).extractall('kj')"
grep -rEo 'jd\.com|taobao\.com|coml\.cc|chrome\.history|chrome\.cookies' kj/   # expected: empty
grep -o '.\{60\}chrome\.search\.query.\{80\}' kj/assets/index-*.js
```

---

## 6. Audit principles

- **Only reproducible facts**: every conclusion carries a file path + code excerpt + runnable command.
- **Tied to version and hash**: verdicts expire once the extension updates.
- **Falsifiable wording**: use "points to / 302-redirects to / forcibly rewrites / not labeled", never "steals / malicious / intentionally".
- **No personal attacks**: judge behavior, not people; publish no personal information.
- **Original packages are read-only**: hash immediately after download, never modify.

---

## 7. Disclaimer

- All conclusions apply **only to the versions and hashes stated**; re-audit after any extension update.
- This repository contains no personal information and makes no subjective judgement about any developer.
- Third-party companies / brands / domains are mentioned **solely as technical description and link-chain explanation**; no allegation is made.
- This repository is for technical research and consumer awareness only and does not constitute legal advice.

---

**English** ｜ [简体中文](README.md)
