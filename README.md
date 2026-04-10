# CMIO 工作日誌 / CMIO Log

台南新樓醫院 智慧醫療團隊，陳禮揚醫師的日常開發記錄。  
追蹤以 `D:\CC` 為主的多個 repo，涵蓋 AI 輔助臨床工作流程開發。

> Daily development log for Li-yang Chen (陳禮揚), CMIO / AI Team, Tainan Sin-lau Hospital.  
> Tracks AI-powered clinical workflow development across multiple repos at `D:\CC`.

---

## 專案說明 / Projects

| 專案 | 說明 | Description |
|------|------|-------------|
| **SLH-HIS** | 院內 HIS 整合應用，含 Dashboard、QueryMRN、交班單 CLI | HIS integration — dashboard, QueryMRN service, handoff CLI |
| **Agent Portal** | 統一 HIS 資料存取閘道，含認證、稽核、資料清洗 | Unified HIS data gateway with auth, audit, and data recipes |
| **Inpatient App** | AI 輔助住院醫療文件 Next.js 應用 | AI-assisted inpatient clinical documentation app (Next.js) |
| **Portable CC** | 可攜式 Claude Code 環境，含 skills、bootstrap、LAN 同步 | Portable Claude Code runtime with skills, bootstrap, LAN sync |
| **CMIO Log** | 本 repo — 開發日誌 | This repo — development log |

📄 **架構文件 / Architecture Docs** → [`docs/`](./docs/)

---

## 2026-04-10 (四 / Thu)

### Inpatient App

WIP 進行中 — 7 個檔案異動（+141 行），尚未 commit：

- 新增 `GenerateModal.tsx` — AI 病歷生成觸發 UI
- `TPRFlowsheet.tsx` 擴充（+59 行）— 生命徵象顯示強化
- `LabFlowsheet.tsx` — 檢驗資料渲染調整
- `SourceDataPanel.tsx`, `Dialog.tsx` — UI 元件更新
- `prompts/progress-note.ts` — 病程記錄 prompt 調校

### CMIO Log

- Repo 初始化，建立兩週完整歷史紀錄

---

## 2026-04-09 (三 / Wed)

### Inpatient App

- `7157d2e` feat(handoff): v3 docx/pdf overhaul with feedback fixes

### Agent Portal

- `fa62a06` chore: update plans, docs, test config
- fix: handoff v3 feedback — I/O aggregation, CXR missing, lab relevance, med category
- fix: QA-driven improvements — meds 500, pool reconnect, surgeries impl, blood_sugar removal

### SLH-HIS

- `1fbc64c` feat(inpatient): add PDF export endpoint + refactor docx generator

### Portable CC

- chore: update skills submodule (cc-dehydrate/hydrate) + settings

---

## 2026-04-08 (二 / Tue)

### Inpatient App

- feat(handoff-v2): rich DOCX with TPR charts + PDF via Playwright
- feat(handoff-v2): data-driven handoff, UI polish, TPR fixes, navbar rename

---

## 2026-04-07 (一 / Mon)

### Inpatient App — Phase 5 push（7 commits）

- feat(phase5-6): Polish, Word export, DataSourceSelector, TPR enhancements
- feat(phase4): all note types + AI consult + server scripts + PRD update
- feat: DD checklist, patient notes panel, draft storage; expand GenerateModal
- feat(ui): improve contrast, nav font size, AI consult zh-TW text
- fix: add retry with backoff for Anthropic 529 overloaded errors
- Merge feat/phase4-all-note-types into main

### Agent Portal — zh-TW + 新端點（6 commits）

- Add tubes endpoint, O2 vitals fields, nursing NANDA, fix IO/dx/summary
- Dashboard zh-TW 翻譯 + 圖表可讀性改善
- Dashboard: fix charts, add zh-TW endpoint descriptions in popup
- API docs 全面 zh-TW 翻譯；PatientInfo 加入 admission_source 型別
- fix: QA-driven stability improvements across portal
- docs: update api-docs.html; add start-portal.sh launcher

### Portable CC

- chore: update skills submodule, CLAUDE.md, settings, cc-share.bat

---

## 2026-04-06 (日 / Sun) — 最大單日（18 commits）

### Inpatient App — 從零開始，Phase 1-3（9 commits）

- feat(phase1+2): foundation + patient list & workspace shell
- docs: Phase 2.5 PRD，portal encoding fix、orders parsing 提示詞
- feat(phase2.5): role-based routing, sidebar, zh-TW UI, TPR flowsheet
- refactor: simplify Phase 1+2 code after review
- feat(phase3): AI 病歷生成核心 — Progress Note with structured data prompts
- feat(phase3): lab flowsheet body-fluid separation, exams panel, RCW sidebar filter
- Merge branches into main

### Agent Portal — Phase 6 + 新端點（6 commits）

- Phase 6: Fix inpatient labs (LATSDL2) + 5 new CLI commands + recipe enrichment
- feat: PFUDD pharmacy source-of-truth for active orders/meds filtering
- Add patient exams endpoint — 影像/檢查報告含 PACS 連結
- Improve lab response: add test_name, group classification, collected_at ISO date
- Add root redirect, /api-docs route, API Docs link in dashboard header
- Add test accounts, dashboard view-only access, expired key cleanup, API docs

### Portable CC（3 commits）

- feat: add never-kill-node safety rule to CLAUDE.md
- chore: update skills submodule (cc-pull/cc-push add inpatient repo)

---

## 2026-04-05 (六 / Sat)

### Agent Portal

- Direct orders/meds via NSARDL2+M02, fix Codex-flagged regressions (P2a/P2b/P3)

### SLH-HIS

- feat(handoff-cli): refactor docx generator, update query-mrn and types

### Portable CC（3 commits）

- chore: bump skills submodule (cc-push/pull GitHub-only, new cc-share/cc-sync)
- chore: update CLAUDE.md, settings, launch.ps1; bump skills submodule (portal-qa + slh-his)

---

## 2026-04-04 (五 / Fri) — Agent Portal 衝刺（24 commits）

### Agent Portal — 從零到上線（22 commits）

- Initial commit: 專案骨架 + auth POC + 實作計劃
- 完整 PRD 草稿
- PRD v2：整合 Codex review（8 項修正）+ HIS schema 研究（6 個問題解答）
- Phase 4: recipes, prompt formatter, HTTP server（161 tests pass）
- Phase 5: 直連 HIS，migration system，production polish
- 加入帳號登入 + 稽核 IP 記錄 + 隨需稽核 dashboard
- Dashboard: 自動關機、帳號管理、A+/A- 字型調整、請求細節彈窗
- 修正 Big5 encoding、dashboard login cookie、.env 載入
- 將所有可 SQL 端點從 QueryMRNService proxy 遷移至直連 HIS DB
- 修正 parser gap vs handoff-cli，接通 HTTP routes 至直連 HIS

### Portable CC（3 commits）

- feat: add codex launcher, HIS DB write guard hook, npm SSL config
- chore: update skills submodule — agent-portal skill v2

---

## 2026-04-03 (四 / Thu)

### Portable CC（5 commits）

- feat: 加入 LAN 同步腳本（cc-share.bat + cc-pull-lan.bat）
- feat: cc-ops 加入 SMB share/unshare 指令
- fix: bootstrap 在 git clone 前先從 seed 複製備用 launcher
- chore: slh-his skill 移到 directory/SKILL.md 使其可被呼叫

---

## 2026-04-01 (二 / Tue)

### Portable CC（4 commits）

- feat: 更新 ventilator-catastrophic-illness skill，加入 IDS 規則 + QueryMRN
- refactor: 整理 root，utility scripts 移至 scripts/，加入 cc-ops menu
- chore: 加入 codex 至可攜式 runtime（PATH, pack.ps1, skills submodule）
- feat: 加入 Claude Code 自動更新、遠端控制、每日更新檢查

---

## 2026-03-30 (日 / Sun)

### Portable CC — Bootstrap & seed（7 commits）

- feat: 加入最精簡 bootstrap seed，供新電腦初始化
- feat: 改善 bootstrap 磁碟搜尋；加入 cc-seed skill
- feat: cc-push/cc-pull 加入 session 歷史備份/還原
- feat: 加入獨立 session 備份腳本（backup-sessions.bat）

### SLH-HIS — Dashboard + 平台整合（7 commits）

- feat: 統一 sinlau-ai platform，dashboard 強化
- fix: dashboard PACS auth, glucose fallback, med sorting, TPR auto-scroll
- fix: start.bat 自動偵測並啟動 QueryMRNService
- fix: layout state reset, ICU vitals, allergy dedup
- feat: OrderHistoryApp 整合 — 逆向工程 + dashboard 強化

---

## 2026-03-29 (六 / Sat)

### Portable CC

- feat: cc-save skill 改名為 cc-push；加入 cc-pull skill
- feat: runtime archive 改為 zip；加入 cc-save skill + headless script

### SLH-HIS

- feat: QueryMRNService HTTP client, conflict resolution, CRLF docs
- merge: handoff-cli 強化（active orders, RCW filter, PDF, local Chrome）

---

## 2026-03-28 (五 / Fri)

### SLH-HIS（3 commits）

- fix: 改用本機 Chrome（D:\CC\chrome-win64）生成 PDF，跳過 Playwright 安裝
- fix: 還原缺失的 parseStructuredVitals 與 VitalReading 型別
- fix: active order bug, RCW filter, 24hr time, drug lists, PDF output

---

## 總結 / Summary（2026-03-28 ～ 04-10）

| 專案 / Repo | Commits | 活躍天數 / Active Days | 主題 / Theme |
|-------------|---------|----------------------|-------------|
| **Portable CC** | 28 | 10 | Skills submodule、bootstrap/seed、LAN sync、Codex、cc-ops |
| **Agent Portal** | 33 | 5 | 從零到上線 — PRD、Phase 4-6、dashboard、直連 HIS DB、API docs |
| **Inpatient App** | 17 | 4 | 從零到上線 — Phase 1-6、AI 病歷、交班 DOCX/PDF、TPR 圖表 |
| **SLH-HIS** | 14 | 4 | Dashboard 修正、QueryMRNService、handoff-cli、PDF export |
| **CMIO Log** | 1 | 1 | 初始 commit |

**亮點 / Highlights：**
- 兩個新專案（Agent Portal、Inpatient App）各自從零出發，約 5 天內進入可用狀態
- Agent Portal：5 天內 33 commits — 完整 HIS 資料 API，含認證、dashboard、161+ 測試
- Inpatient App：4 天內 17 commits — Next.js AI 病歷生成器，含 handoff DOCX/PDF、TPR 圖表
- Portable CC bootstrap 系統成熟：thin-seed、cc-dehydrate/hydrate、LAN sync
