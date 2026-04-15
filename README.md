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
| **Drug Interactions** | 藥物交互作用檢查服務（SLH + DynaMed） | Drug interaction checker service (SLH HIS + DynaMed severity) |
| **MediCloud Pro** | 健保雲端藥歷增強 Chrome 擴充功能 | Enhanced NHI MediCloud viewer Chrome extension |
| **Portable CC** | 可攜式 Claude Code 環境，含 skills、bootstrap、LAN 同步 | Portable Claude Code runtime with skills, bootstrap, LAN sync |
| **CMIO Log** | 本 repo — 開發日誌 | This repo — development log |

📄 **架構文件 / Architecture Docs** → [`docs/`](./docs/)

---

## 2026-04-15 (二 / Tue)

### 整體摘要

三大工作：NHI 歸戶 Phase 2 排除條件研究、Drug Interactions DynaMed 爬蟲、MediCloud Pro Chrome 擴充。
1. **NHI 歸戶報表 Phase 2** — 反推排除條件、proxy filter 實作、發現 Q2208PD 權威資料源、rt01 圖表
2. **Drug Interactions — DynaMed 爬蟲 + V3 Excel** — 7 commits，完成自動化爬蟲、品牌名對照、Excel enrichment
3. **MediCloud Pro — Chrome Extension v0.1** — 4 commits，健保雲端藥歷增強顯示擴充功能

---

### NHI 歸戶報表 — Phase 2 排除條件研究與 rt01 圖表

**目標**：解決 dashboard 門診人數 (61,521) 與 rt01 (53,152) 差距 ~15.7% 的問題。

**研究過程**：
- 使用 set difference 比對 Excel ground truth (門診歸戶分析 B.xlsx) vs T01L16 查詢結果
- 探索 200+ 張 AS400 表格，確認 NHI 案件分類不在 T01L16（所有申報欄位為空）
- 發現 T01F16='1' 為最強篩選信號，加上 23 個高排除率診間（洗腎/復健/化療等）
- 交叉驗證 4 個季度，精確度從 75% 提升至 98-100%

**關鍵發現**：
- `SLHLIB.Q2208PD`：NHI 歸戶程式的權威每日累計資料，完全匹配 rt01 Excel
- `SLHLIB.Q2208A`：NHI 已過濾就醫明細（含案件分類 G25TYP），目前存 113Q3 資料
- Excel TN sheet = 全機構（site 11/12/21/22 都歸新樓台南）

**實作**：
- `fetch.ts`：加入 T01F16='1' + 診間排除 + 全院區歸 TN
- `generate-rt01-chart.ts`：rt01 風格圖表，資料源為 Q2208PD + T01L16 即時查詢
- `docs/phase2-progress-report.pdf`：完整研究報告

---

### Drug Interactions — DynaMed 爬蟲 + V3 Excel

#### DynaMed 自動化爬蟲

- `scrape-dynamed.mjs`：用 agent-browser 驅動 Edge 瀏覽器，自動爬取 DynaMed 藥物交互作用資料
- 爬取 225 對藥物組合：171 對成功取得嚴重度、54 對無法配對（台灣特有品牌名）、0 失敗
- 支援 `--resume` 斷點續爬，結果存入 `cache/interactions.json`

#### V3 Excel 產出（700/3906 配對）

- `generate-v3.mjs`：從 V2 格式完整複製，新增欄位 I "DynaMed嚴重度"
- 建立 ~350 筆台灣品牌名→學名對照表（BOKEY→aspirin, PLAVIX→clopidogrel 等）
- 嚴重度分布：595 Major、69 Contraindicated、36 Moderate
- 從前次僅 1 筆配對提升至 700 筆，覆蓋率 18%

#### 文件與快照

- `docs/dynamed-scrape-results.md`：爬蟲結果完整分析報告
- `docs/dynamed-scrape-progress.json`：爬蟲執行稽核軌跡
- `docs/interactions-v3-snapshot.json`：interactions cache 版本快照

<details>
<summary>技術細節</summary>

- NHI: `nhi-aggr-report` c67d206, `skills` 9b657b7, `CC` 507f85b
- DynaMed: 7 commits: e3229e2, 9804e7c, 34789b4, d5b8c72, b64dcb4, 7e27f1f, 23bead7

</details>

---

### MediCloud Pro — Chrome Extension v0.1

#### 健保雲端藥歷增強顯示

- Chrome MV3 擴充功能，掛載於 `medcloud2.nhi.gov.tw`
- 攔截 XHR/fetch 請求，擷取雲端藥歷 API 回應資料
- Sidebar UI 以 timeline 形式重新呈現藥歷資訊

#### CSP 繞過與注入架構

- 使用 MAIN world content script 繞過 MediCloud 的 CSP 限制
- 將 XHR interceptor 與 sidebar mount 分離：`injector.ts`（document_start, MAIN world）+ `index.tsx`（ISOLATED world）
- 收緊 mount guard、pending cap、listener cleanup

<details>
<summary>技術細節</summary>

- `src/content/injector.ts`：MAIN world XHR/fetch 攔截
- `src/content/index.tsx`：ISOLATED world sidebar React 元件
- 4 commits: 7a8d5a0, aab34a5, 895db21, 3bddcfb

</details>

---

## 2026-04-14 (一 / Mon)

### 整體摘要

兩大主軸：Agent Portal 完成 proxy→direct 全面遷移，以及發現並整理院內藥物交互作用資料庫。
1. **Agent Portal — 全面直連 + ER case + OPD orders** — 6 commits，拔掉所有 proxy 依賴，新增 ER 病歷直查、門診醫令
2. **Drug-Drug Interaction 資料庫探勘** — 找到 `SINLAU.MED_INTERACTION` 表 + 藥劑科 Excel，解析成 JSON 存入 slh-his skill
3. **Inpatient App — TPR 修正 + 效能優化** — hydration fix、loading indicators、預設淺色模式

---

### Agent Portal — Proxy→Direct 全面遷移

#### 完成直連遷移 (Phase 5 收尾)

所有 HIS 資料存取從 .NET proxy (QueryMRNService) 遷移到直接 ODBC/SQL Server 查詢：
- 移除所有 `config.direct.*` feature flags — 不再需要 A/B 切換
- 移除 `client-proxy.ts` 的所有 import — 檔案保留但不再使用
- 更新 CLAUDE.md 文件反映 direct-only 架構

#### 新功能：ER Case 直查

- `fetchErCaseDirect()` 取代 ERExportText.dll，直接查 `SLTHIS.EmergencyCase`
- `er-case-codes.ts` 包含從 .NET DLL 反編譯的 CodeToText 對應表
- 保留 DB 欄位 typo: `InitalImpression`, `RevisedDragnosis`, `DrugAllery`

#### 新功能：OPD Orders

- 新增 `portal patient opd-orders` 指令
- 查詢 `OOORDL7` + `M02` 取得門診處方
- 整合進 OPD visit bundle（`--include orders`）

#### ManagedPool 重構

- SQL Server 連線池改用 promise coalescing 模式
- 修復並發查詢在 pool 斷線時的 race condition（null-ref crash）
- 所有 concurrent callers await 同一個 reconnect promise

<details>
<summary>技術細節</summary>

- `src/his/client-direct.ts`：`ManagedPool` class 取代三個獨立的 `getNisPool`/`getSlthisPool`/`getSinlauPool`
- `src/cli/commands/opd/orders.ts`：新增 OPD orders command
- `src/parsers/er-case-codes.ts`：ER case code-to-text mappings
- 6 commits: 1112149, f8d2f3c, 268a3a1, 057b890, 18cacf8, b2ea65f

</details>

---

### Drug-Drug Interaction 資料庫

#### 發現 SINLAU.MED_INTERACTION

- 在 `SINLAU` SQL Server (`192.168.13.36`) 找到 `MED_INTERACTION` 表
- 1,579 筆啟用中的交互作用紀錄，使用院內簡碼（如 `OBOKEY`, `OCOUMA`）

#### 藥劑科 Excel 解析

- 藥劑科提供的 `交互作用.xlsx`：西藥 3,906 對 + 中藥 504 對 = 4,410 對
- 解析存為 `slh-his/data/drug-interactions.json`（1.3MB）
- Excel 比 DB 更完整（含停用藥品和中藥），確認為 authoritative source

#### DB vs Excel 比對

- 重疊 475 對，Excel 獨有 3,262 對，DB 獨有 774 對
- Excel 多出的主要是停用藥品；DB 有些 wildcard 格式碼（`IKE*O`）
- 46 對描述不一致（typo、不同面向描述）

#### 品質驗證

- 隨機抽 10 對用藥理學知識驗證 — 10/10 正確
- 涵蓋 QT prolongation、CYP450 交互、螯合、CNS depression 等機轉

<details>
<summary>技術細節</summary>

- 新增 `slh-his/drug-interactions.md` — 完整 schema、SQL recipes、lookup patterns
- 新增 `slh-his/data/drug-interactions.json` — 4,410 對 DDI 資料
- 更新 `slh-his/SKILL.md` — Quick Reference 加入 DDI 行、SINLAU 描述更新
- 未來計劃：接 DynaMed API 做 enrichment

</details>

---

### Inpatient App — 修正 + 優化

- **TPR hydration fix**：`Date.now()` snap to minute，解決 SSR/CSR 時間不一致導致的 hydration mismatch
- **Loading indicators**：資料載入時顯示 loading 狀態
- **效能優化**：減少不必要的 re-render
- **預設淺色模式**：light theme 改為預設

<details>
<summary>技術細節</summary>

- 2 commits: 3d0b02f, 474fece

</details>

---

### slh-his — ERExportText 反編譯文件

- 新增 `legacy/ERExportText-decompiled.md` — 從 .NET DLL 用 ildasm 反編譯的完整文件
- 作為 `fetchErCaseDirect()` 的 code-to-text mapping 參考依據

---

## 2026-04-12 (日 / Sun)

### 整體摘要

今天是功能密集日，跨三個 repo 做了大量改動：
1. **住院 App — 切帳偵測 + UI 全面 redesign** — 自動偵測健保切帳、合併前次住院資料、UI 整體放大重排
2. **Agent Portal — OPD 直連 + MAR 給藥紀錄 + 住院歷史** — 三個新 endpoint，擺脫 proxy 依賴
3. **Portable CC — Skills 整理 + cc-dehydrate 執行**

---

### Inpatient App — 切帳偵測 + 深色模式 + UI Redesign

#### 切帳 (Billing Split) 自動偵測

健保給付需要切帳時（前一天出院、隔天入院、無經急診/門診），系統自動：
- 偵測並標記「疑似切帳」，PatientBanner 顯示合併住院天數（如 D16）
- 資料來源選擇器自動勾選前次住院，並連帶選上前次的入院來源（ER/OPD）
- AI 生成時納入前次住院的完整資料（透過 `recipes/full-admission`）
- Vitals 自動用前次 CSN 查詢（NIS 護理紀錄仍登記在舊 CSN 下）

#### 資料來源選擇器擴充

原本只能選門診和急診，現在新增「住院」類型：
- 預設拉 30 天，每次可多載 30 天
- 選擇後用 `/api/recipes/full-admission?csn=X&format=prompt` 拉完整住院資料
- 頁面載入即 fetch（不需打開面板），支援自動選擇

#### TPR Flowsheet — 胰島素 MAR

血糖欄位下方新增 RI 行，顯示護理實際施打的胰島素劑量：
- 資料來源：NIS `slh_insulin` 表（MAR 給藥紀錄）
- 對應邏輯：MAR 給藥時間與 vitals 量測時間配對（2hr 窗口）
- 顯示：`3U` + hover 看注射位置和時段

#### UI Redesign

- Base font 15px -> 16px
- Navbar 改為雙行：上排 nav + actions，下排 note buttons
- 明/暗 toggle 改用文字（明/暗），與 A+/A- 統一按鈕組
- PatientBanner 字體加大（text-2xl），admission info 更突出
- 面板標題 font-bold，rounded-xl cards
- 暗色模式 tokens 調整：更深的 surfaces、更亮的 borders
- TPR chart 暗色模式用透明背景
- Labs/Exams category headers 加 dark mode variants
- Anthropic max_tokens 4096 -> 8192 修復 AI 筆記截斷

<details>
<summary>技術細節</summary>

- 新增 `components/BillingSplitBadge.tsx`（client island for server component PatientBanner）
- `lib/types.ts`：新增 `AdmissionRecord`、`InsulinMAR`；Encounter 加 `inpatient` type + `csn`/`dischargeDate`/`admissionSource`
- `lib/encounter-context.tsx`：新增 `billingSplitAdmitDate` state
- `components/DataSourceSelector.tsx`：重寫 — 加住院 fetch、切帳偵測、分組顯示、load more
- `components/SourceDataPanel.tsx`：VitalsPanel 加 CSN fallback fetch + MAR insulin fetch + billing split auto-extend
- `components/TPRFlowsheet.tsx`：新增 RI insulin row with MAR data matching
- `app/api/generate/route.ts`：`fetchPreAdmissionContext` 支援 inpatient type
- `app/globals.css`：dark mode tokens 全面調整
- `components/AuthHeader.tsx`：雙行 layout redesign
- `components/LabFlowsheet.tsx`：GROUP_COLORS 加 dark mode
- 6 commits: 495a806, 3601a4d, eb61713 等

</details>

---

### Agent Portal — OPD 直連 + MAR + 住院歷史

#### OPD 直連 AS400（擺脫 Proxy）

門診資料（visits + SOAP）從 QueryMRN proxy 遷移到直連 AS400 ODBC：
- `fetchOpdVisitsDirect()`：查 T01L16 + PSEMPP（醫師名）+ HIBMSL1（ICD 診斷）
- `fetchOpdSoapDirect()`：查 OOSOAL1，多 SOPSEQ 行自動拼接
- `opd soap --date` 改為 optional，自動查最近一次已發生的門診日期
- Feature flags：`PORTAL_DIRECT_OPD_VISITS=1`, `PORTAL_DIRECT_OPD_SOAP=1`

#### MAR 給藥紀錄（胰島素 + 血糖）

新增 NIS 護理給藥紀錄查詢：
- HTTP: `GET /api/patient/mar?mrn=X&days=7&category=insulin`
- CLI: `portal patient mar --mrn X --days N --category insulin`
- 資料來源：NIS `slh_insulin` 表
- 回傳：給藥時間、血糖值、RI 劑量、注射位置、時段

#### 住院歷史

新增 endpoint 查詢病人的住院紀錄：
- HTTP: `GET /api/patient/admissions?mrn=X&days=N`
- CLI: `portal patient admissions --mrn X --days N`
- 資料來源：AS400 PFIPD 表
- 回傳：CSN、入出院日期、科別、主治醫師、住院天數、入院來源

<details>
<summary>技術細節</summary>

- 3 commits: 58cbb9d (OPD direct), b52c47d (MAR), 7849c65 (admissions CLI)
- NIS slh_insulin schema 完整記錄於 `slh-his/nis-tables.md`

</details>

---

### Portable CC — Skills 整理 + 備份

- **Skills 更新**：NIS insulin MAR schema 補齊、kanfu-homecare 文件更新
- **cc-dehydrate 執行**：spore 651MB + full backup 2.95GB 至 D:\
- **cc-push 新增 Phase 6**：CMIO Log repo 納入批次推送流程

<details>
<summary>技術細節</summary>

- `cc-seed/SKILL.md` 及 `cc-sync/SKILL.md` 已刪除（-287 行）
- `cc-ops/SKILL.md` 新增 SMB 共享與環境說明（+23 行）
- `launch.bat` 重構：調整 PATH 初始化順序，修正 Windows Terminal 啟動參數
- `launch.ps1` 補充環境變數設定邏輯
- `.gitignore` 新增排除規則

</details>

---

## 2026-04-10 (四 / Thu)

### 整體摘要

今天的工作以「準備」為主：住院 App 進行功能開發中（WIP），以及 CMIO Log 這個記錄 repo 本身正式初始化上線。

---

### Inpatient App — UI 功能開發中

住院 App 的 AI 病歷生成功能持續開發，新增了讓醫師點擊觸發「AI 幫我寫病歷」的彈窗介面，以及生命徵象圖表與檢驗資料的顯示強化。尚未正式 commit，屬工作進行中狀態。

- 新增「生成病歷」觸發彈窗介面（`GenerateModal`）
- 生命徵象顯示項目擴充（+59 行）
- 檢驗資料、來源資料面板、對話框等 UI 元件同步更新
- 病程記錄（Progress Note）的 AI 提示詞持續調校

<details>
<summary>技術細節</summary>

WIP — 7 個檔案異動（+141 行），尚未 commit：

- 新增 `GenerateModal.tsx` — AI 病歷生成觸發 UI
- `TPRFlowsheet.tsx` 擴充（+59 行）— 生命徵象顯示強化
- `LabFlowsheet.tsx` — 檢驗資料渲染調整
- `SourceDataPanel.tsx`, `Dialog.tsx` — UI 元件更新
- `prompts/progress-note.ts` — 病程記錄 prompt 調校

</details>

---

### CMIO Log — 正式初始化

這份工作日誌 repo 建立上線，並補齊了過去兩週的完整歷史記錄。

<details>
<summary>技術細節</summary>

- Repo 初始化，建立 2026-03-28 ～ 04-10 兩週完整歷史紀錄（`da22b6e`）

</details>

---

## 2026-04-09 (三 / Wed)

### 整體摘要

今天聚焦在住院 App 交班單的品質提升（v3 版本），以及 Agent Portal 根據實際使用回饋進行的一批 QA 修正。

---

### Inpatient App — 交班單 v3 大改版

交班單（Handoff）是醫師換班時用來交接病情的文件。v3 版本針對前幾天使用後收到的回饋全面修正：I/O（液體出入量）資料彙整方式改善、胸部 X 光漏顯示的問題修正、檢驗值相關性篩選邏輯調整、藥物分類更清晰。

- 液體出入量（I/O）彙整邏輯修正，資料更準確
- 修正胸部 X 光（CXR）在某些情況下不出現的問題
- 檢驗值相關性判斷改善，只顯示真正有意義的數值
- 藥物分類整理，交班單更易閱讀

<details>
<summary>技術細節</summary>

- `7157d2e` feat(handoff): v3 docx/pdf overhaul with feedback fixes
- 修正項目：I/O aggregation、CXR missing、lab relevance、med category

</details>

---

### Agent Portal — QA 回饋修正

Agent Portal 的 QA 過程中發現幾個穩定性問題，今天一併修正。

- 修正藥物查詢偶爾回傳 500 錯誤的問題
- 修正資料庫連線池在長時間運作後重連失敗的問題
- 補齊手術記錄查詢功能
- 移除不應出現的血糖欄位顯示問題

<details>
<summary>技術細節</summary>

- `fa62a06` chore: update plans, docs, test config
- fix: handoff v3 feedback — I/O aggregation, CXR missing, lab relevance, med category
- fix: QA-driven improvements — meds 500 error, pool reconnect, surgeries impl, blood_sugar removal

</details>

---

### SLH-HIS — PDF 匯出端點

HIS 整合層新增了 PDF 匯出 API，並重構了 DOCX 生成器，讓住院 App 可以呼叫它產生正式文件。

<details>
<summary>技術細節</summary>

- `1fbc64c` feat(inpatient): add PDF export endpoint + refactor docx generator

</details>

---

### Portable CC — Skills 更新

<details>
<summary>技術細節</summary>

- chore: update skills submodule (cc-dehydrate/hydrate) + settings

</details>

---

## 2026-04-08 (二 / Tue)

### 整體摘要

今天是住院 App 交班單功能的重大里程碑：從純文字格式升級為含有 TPR 圖表的精美 Word 文件，並支援 PDF 匯出。

---

### Inpatient App — 交班單 v2：Word 文件 + TPR 圖表

原本的交班單只是純文字。今天升級為 Word（.docx）格式，內嵌生命徵象趨勢圖（TPR 圖表），同時也可以匯出 PDF。這讓醫師換班時的書面交接更正式、更易讀。

- 交班單現在可以直接輸出為 Word 文件（.docx）
- 文件內嵌生命徵象趨勢圖（體溫、脈搏、呼吸、血壓）
- 可同步匯出為 PDF 方便列印或傳送
- 介面整體美化，導覽列改名更符合臨床用語

<details>
<summary>技術細節</summary>

- feat(handoff-v2): rich DOCX with TPR charts + PDF via Playwright
- feat(handoff-v2): data-driven handoff, UI polish, TPR fixes, navbar rename
- 使用 Playwright 的 `page.pdf()` 將 HTML 渲染轉為 PDF（避免中文亂碼）

</details>

---

## 2026-04-07 (一 / Mon)

### 整體摘要

週一是住院 App 和 Agent Portal 雙線並行的密集開發日，共推送 13 個 commit。住院 App 完成了所有病歷類型的 AI 生成功能（Phase 4-5），Agent Portal 完成了中文化和多個新資料端點。

---

### Inpatient App — Phase 4-5：完整病歷類型 + UI 精修

AI 病歷生成的核心功能全面上線：住院病歷、病程記錄、會診回覆、出院摘要等所有主要病歷類型都可以用 AI 輔助生成。同時新增了鑑別診斷清單（DD checklist）和病人備忘便條功能。

- 所有主要病歷類型（住院記錄、病程記錄、會診、出院摘要）完成 AI 生成
- 新增鑑別診斷清單，輔助臨床思考
- 新增病人備忘便條面板，草稿自動儲存
- Anthropic API 過載時自動重試，避免服務中斷
- 介面對比度、字型大小改善，中文 UI 文字優化

<details>
<summary>技術細節（7 commits）</summary>

- feat(phase5-6): Polish, Word export, DataSourceSelector, TPR enhancements
- feat(phase4): all note types + AI consult + server scripts + PRD update
- feat: DD checklist, patient notes panel, draft storage; expand GenerateModal
- feat(ui): improve contrast, nav font size, AI consult zh-TW text
- fix: add retry with backoff for Anthropic 529 overloaded errors
- Merge feat/phase4-all-note-types into main

</details>

---

### Agent Portal — 全面中文化 + 新端點（6 commits）

Agent Portal 的儀表板和 API 文件全面翻譯為繁體中文，讓臨床人員更容易理解。同時新增管路（tubes）、氧氣、護理 NANDA 等臨床資料端點。

- 儀表板介面完整中文化
- 新增管路、氧氣濃度等 ICU 常用資料端點
- 新增護理 NANDA 診斷資料查詢
- 修正液體出入量、診斷、摘要等資料顯示問題
- API 說明文件全面中文翻譯

<details>
<summary>技術細節（6 commits）</summary>

- Add tubes endpoint, O2 vitals fields, nursing NANDA, fix IO/dx/summary
- Dashboard zh-TW 翻譯 + 圖表可讀性改善
- Dashboard: fix charts, add zh-TW endpoint descriptions in popup
- API docs 全面 zh-TW 翻譯；PatientInfo 加入 admission_source 型別
- fix: QA-driven stability improvements across portal
- docs: update api-docs.html; add start-portal.sh launcher

</details>

---

### Portable CC

<details>
<summary>技術細節</summary>

- chore: update skills submodule, CLAUDE.md, settings, cc-share.bat

</details>

---

## 2026-04-06 (日 / Sun) — 最大單日（18 commits）

### 整體摘要

這是迄今為止最密集的一天，共推送 18 個 commit。住院 App 從零開始建立了 Phase 1-3 的完整框架，Agent Portal 完成 Phase 6 並新增多個重要端點。

---

### Inpatient App — 從零到可用，Phase 1-3（9 commits）

住院 App 在這一天從空白 repo 建立起完整的框架：病人列表、工作空間介面、角色權限路由、生命徵象圖表（TPR）、AI 病程記錄生成核心，以及影像檢查面板。

- 建立病人列表頁和個人工作空間介面（Phase 1+2）
- 角色權限路由：主治醫師 vs 住院醫師不同視角
- 側欄 + 中文 UI，生命徵象 TPR 圖表首次上線
- AI 病程記錄（Progress Note）生成核心功能完成（Phase 3）
- 檢驗值圖表支援體液分類，影像檢查面板整合 PACS

<details>
<summary>技術細節（9 commits）</summary>

- feat(phase1+2): foundation + patient list & workspace shell
- docs: Phase 2.5 PRD，portal encoding fix、orders parsing 提示詞
- feat(phase2.5): role-based routing, sidebar, zh-TW UI, TPR flowsheet
- refactor: simplify Phase 1+2 code after review
- feat(phase3): AI 病歷生成核心 — Progress Note with structured data prompts
- feat(phase3): lab flowsheet body-fluid separation, exams panel, RCW sidebar filter
- Merge branches into main

</details>

---

### Agent Portal — Phase 6 + 新端點（6 commits）

Agent Portal 補齊了住院檢驗查詢的資料來源（LATSDL2），新增 5 個 CLI 指令，並整合了藥局的正式用藥資料來源（PFUDD）作為判斷「目前有效藥單」的依據。

- 修正住院檢驗資料來源，補齊之前遺漏的表（LATSDL2）
- 新增 5 個 CLI 指令，豐富命令列操作
- 以藥局 PFUDD 資料庫作為有效藥物的唯一依據，避免資料混亂
- 新增影像/檢查報告端點，含 PACS 連結
- 稽核 dashboard 加入測試帳號、唯讀存取、過期 key 清理

<details>
<summary>技術細節（6 commits）</summary>

- Phase 6: Fix inpatient labs (LATSDL2) + 5 new CLI commands + recipe enrichment
- feat: PFUDD pharmacy source-of-truth for active orders/meds filtering
- Add patient exams endpoint — 影像/檢查報告含 PACS 連結
- Improve lab response: add test_name, group classification, collected_at ISO date
- Add root redirect, /api-docs route, API Docs link in dashboard header
- Add test accounts, dashboard view-only access, expired key cleanup, API docs

</details>

---

### Portable CC（3 commits）

新增了安全規則：禁止用指令一次殺掉所有 Node.js 程序（那樣會把 Claude Code 自己也殺掉）。同時更新 skills 讓 cc-push 支援住院 App repo。

<details>
<summary>技術細節</summary>

- feat: add never-kill-node safety rule to CLAUDE.md
- chore: update skills submodule (cc-pull/cc-push add inpatient repo)

</details>

---

## 2026-04-05 (六 / Sat)

### 整體摘要

週六繼續修正 Agent Portal 的資料查詢問題，並整理了 SLH-HIS 的 DOCX 產生器。

---

### Agent Portal — 藥物和醫囑查詢修正

修正了 Agent Portal 直接查詢 HIS 藥物和醫囑的問題：改用 NSARDL2+M02 這兩張表作為資料來源，並修正了 Codex 程式碼審查中找出的三個回歸問題。

- 藥物和醫囑現在直接從 HIS 資料庫讀取，不再透過中介服務
- 修正三個已知回歸問題（P2a、P2b、P3）

<details>
<summary>技術細節</summary>

- Direct orders/meds via NSARDL2+M02 tables
- fix: Codex-flagged regressions P2a/P2b/P3

</details>

---

### SLH-HIS — 交班單 CLI 重構

交班單 CLI 的 DOCX 產生器進行了重構，以配合住院 App 的新需求。

<details>
<summary>技術細節</summary>

- feat(handoff-cli): refactor docx generator, update query-mrn and types

</details>

---

### Portable CC

<details>
<summary>技術細節（3 commits）</summary>

- chore: bump skills submodule (cc-push/pull GitHub-only, new cc-share/cc-sync)
- chore: update CLAUDE.md, settings, launch.ps1; bump skills submodule (portal-qa + slh-his)

</details>

---

## 2026-04-04 (五 / Fri) — Agent Portal 誕生（24 commits）

### 整體摘要

這一天幾乎只做一件事：從零建立 Agent Portal。這是一個統一存取新樓醫院 HIS 資料的 API 閘道，讓住院 App 和其他 AI 工具可以安全、方便地取得病人資料。一天之內完成了從設計文件（PRD）到生產環境的全程。

---

### Agent Portal — 從設計到上線（22 commits）

Agent Portal 解決了一個核心問題：各個 AI 工具都需要從 HIS 取得病人資料，但 HIS 的資料格式複雜、欄位命名不直觀，每個工具都要自己處理一遍。Agent Portal 作為統一閘道，把這些複雜性集中處理，對外提供乾淨的 API。

- 當天完成 PRD 設計 → Codex 程式碼審查 → HIS schema 研究 → 實作 → 上線
- 161 個測試全數通過（Phase 4）
- 直連 HIS 資料庫（不再透過 QueryMRNService 中介），效能更好
- 加入帳號登入、稽核 IP 記錄、自動關機等安全機制
- 稽核 dashboard 提供管理員視角

<details>
<summary>技術細節（22 commits）</summary>

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

</details>

---

### Portable CC（2 commits）

新增了 Codex 啟動器，以及一個防護鉤子（hook）避免 AI 工具意外寫入 HIS 資料庫（HIS DB 應為唯讀）。

<details>
<summary>技術細節</summary>

- feat: add codex launcher, HIS DB write guard hook, npm SSL config
- chore: update skills submodule — agent-portal skill v2

</details>

---

## 2026-04-03 (四 / Thu)

### 整體摘要

今天聚焦在 Portable CC 的 LAN 同步功能：讓醫院內的其他電腦可以透過區域網路直接同步 Claude Code 環境，不需要走公網 GitHub。

---

### Portable CC — LAN 同步上線（5 commits）

醫院內有多台工作站，以往只能透過 GitHub 同步 Claude Code 環境。今天新增了院內 LAN 同步方式：一台電腦開啟 SMB 分享，其他電腦可以直接複製整個環境，速度更快，也不受防火牆影響。

- 新增 `cc-share.bat`：一鍵把 D:\ 分享給院內網路
- 新增 `cc-pull-lan.bat`：從院內其他電腦拉取環境
- cc-ops 選單新增 SMB 分享/取消分享的快捷操作
- 修正 bootstrap 流程：先備份 launcher 再執行 git clone，避免意外
- SLH-HIS skill 修正路徑，使其可以被正確呼叫

<details>
<summary>技術細節（5 commits）</summary>

- feat: 加入 LAN 同步腳本（cc-share.bat + cc-pull-lan.bat）
- feat: cc-ops 加入 SMB share/unshare 指令
- fix: bootstrap 在 git clone 前先從 seed 複製備用 launcher
- chore: slh-his skill 移到 directory/SKILL.md 使其可被呼叫（符合 skill 命名規範）

</details>

---

## 2026-04-01 (二 / Tue)

### 整體摘要

四月初的整理日：Portable CC 環境的多項基礎建設更新，包含呼吸器重症特殊材料 skill 更新、Codex 整合、以及 Claude Code 自動更新機制。

---

### Portable CC — 基礎建設完善（4 commits）

- 更新呼吸器重症特殊材料 skill，加入最新的 IDS 規則和 QueryMRN 整合，讓 AI 在申請特材時可以查詢病人資料
- 整理根目錄結構，utility scripts 移至 `scripts/` 資料夾，新增 cc-ops 選單作為統一操作入口
- 將 Codex CLI 加入可攜式執行環境（PATH 設定、pack.ps1 打包腳本、skills 整合）
- 加入每日自動更新機制，Claude Code 會在每天首次啟動時檢查並更新

<details>
<summary>技術細節（4 commits）</summary>

- feat: 更新 ventilator-catastrophic-illness skill，加入 IDS 規則 + QueryMRN
- refactor: 整理 root，utility scripts 移至 scripts/，加入 cc-ops menu
- chore: 加入 codex 至可攜式 runtime（PATH, pack.ps1, skills submodule）
- feat: 加入 Claude Code 自動更新、遠端控制、每日更新檢查

</details>

---

## 2026-03-30 (日 / Sun)

### 整體摘要

週日雙線並行：Portable CC 完善了「薄種子（thin seed）」的 bootstrap 流程，HIS Dashboard 則完成了多項穩定性修正和 OrderHistoryApp 整合。

---

### Portable CC — Bootstrap 系統成熟（7 commits）

「Portable CC」要能在一台新電腦上快速部署。今天完善了這個流程：建立一個極小的 `cc-spore.zip` 作為種子，解壓後執行 bootstrap 就能自動完成所有初始化，包含 session 歷史備份/還原。

- 建立最精簡 bootstrap seed（`cc-spore.zip`），方便帶到新電腦
- 改善磁碟搜尋邏輯，bootstrap 可以自動找到正確的安裝位置
- cc-push / cc-pull 加入 session 歷史備份和還原功能
- 新增獨立的 session 備份腳本（`backup-sessions.bat`）

<details>
<summary>技術細節（7 commits）</summary>

- feat: 加入最精簡 bootstrap seed，供新電腦初始化
- feat: 改善 bootstrap 磁碟搜尋；加入 cc-seed skill
- feat: cc-push/cc-pull 加入 session 歷史備份/還原
- feat: 加入獨立 session 備份腳本（backup-sessions.bat）

</details>

---

### SLH-HIS Dashboard — 穩定性修正 + OrderHistoryApp 整合（7 commits）

HIS Dashboard 是醫師查看病人資料的主要介面。今天修正了多個使用上發現的問題，並整合了 OrderHistoryApp（醫院的醫囑查詢系統），讓資料來源更完整。

- 修正 PACS 影像系統的認證問題，讓 X 光連結可以正常開啟
- 血糖資料顯示修正，藥物排序優化，TPR 圖表自動捲動至最新
- 修正 ICU 生命徵象顯示和過敏紀錄重複問題
- 整合 OrderHistoryApp，逆向工程取得 API 介接方式

<details>
<summary>技術細節（7 commits）</summary>

- feat: 統一 sinlau-ai platform，dashboard 強化
- fix: dashboard PACS auth, glucose fallback, med sorting, TPR auto-scroll
- fix: start.bat 自動偵測並啟動 QueryMRNService
- fix: layout state reset, ICU vitals, allergy dedup
- feat: OrderHistoryApp 整合 — 逆向工程 + dashboard 強化

</details>

---

## 2026-03-29 (六 / Sat)

### 整體摘要

今天主要調整工具命名與打包格式，同時 SLH-HIS 的 handoff-cli 新增了多項功能。

---

### Portable CC — 工具命名整理

將舊有的 `cc-save` 技能正式更名為 `cc-push`，並新增對應的 `cc-pull`，讓推送和拉取更對稱直觀。打包格式也從 tar.gz 改為 zip，在 Windows 環境操作更方便。

<details>
<summary>技術細節</summary>

- feat: cc-save skill 改名為 cc-push；加入 cc-pull skill
- feat: runtime archive 改為 zip；加入 cc-save skill + headless script

</details>

---

### SLH-HIS — Handoff CLI 強化

交班單 CLI 新增了多項功能：主動醫囑顯示、RCW 病房篩選、PDF 輸出，以及改用本機 Chrome 產生 PDF（避免 CDN 連線問題）。

<details>
<summary>技術細節</summary>

- feat: QueryMRNService HTTP client, conflict resolution, CRLF docs
- merge: handoff-cli 強化（active orders, RCW filter, PDF, local Chrome）

</details>

---

## 2026-03-28 (五 / Fri)

### 整體摘要

交班單 CLI 的 bug 修正日：修正了 PDF 產生、資料型別、醫囑顯示等多個問題。

---

### SLH-HIS — 交班單 CLI 修正（3 commits）

- 改用本機安裝的 Chrome（`D:\CC\chrome-win64`）產生 PDF，繞過醫院防火牆對外部 CDN 的封鎖
- 還原被意外刪除的生命徵象型別定義（`parseStructuredVitals`、`VitalReading`）
- 修正有效醫囑顯示、RCW 病房篩選、24 小時制時間、藥物清單、PDF 輸出等多個問題

<details>
<summary>技術細節（3 commits）</summary>

- fix: 改用本機 Chrome（D:\CC\chrome-win64）生成 PDF，跳過 Playwright install
- fix: 還原缺失的 parseStructuredVitals 與 VitalReading 型別
- fix: active order bug, RCW filter, 24hr time, drug lists, PDF output

</details>

---

## 總結 / Summary（2026-03-28 ～ 04-12）

| 專案 / Repo | Commits | 活躍天數 / Active Days | 主題 / Theme |
|-------------|---------|----------------------|-------------|
| **Portable CC** | 30+ | 11 | Skills submodule、bootstrap/seed、LAN sync、Codex、cc-ops、skills 整理 |
| **Agent Portal** | 35+ | 6 | 從零到上線 — PRD、Phase 4-6、dashboard、直連 HIS DB、row-mappers 重構 |
| **Inpatient App** | 20+ | 5 | 從零到上線 — Phase 1-6、AI 病歷、交班 DOCX/PDF、TPR 圖表、深色模式 |
| **SLH-HIS** | 14 | 4 | Dashboard 修正、QueryMRNService、handoff-cli、PDF export |
| **CMIO Log** | 3 | 2 | 初始化、補齊歷史、格式統一 |

**亮點 / Highlights：**
- 兩個新專案（Agent Portal、Inpatient App）各自從零出發，約 5 天內進入可用狀態
- Agent Portal：直連 HIS + 統一 row-mappers，成為 AI 工具取得病人資料的唯一閘道
- Inpatient App：涵蓋所有主要病歷類型，深色模式讓夜班更友善
- Portable CC bootstrap 系統成熟：thin-seed、cc-dehydrate/hydrate、LAN sync
