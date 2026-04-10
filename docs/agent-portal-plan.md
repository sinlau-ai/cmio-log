# Agent Portal — Implementation Plan

## Context

資訊室發現現有 HIS 連線權限過於開放，需要一個統一入口來管理所有 agentic app 對 HIS 的存取。Agent Portal **取代並重寫** 現有的 QueryMRNService，成為唯一的 HIS 資料存取層，提供：登入驗證、群組權限控制、存取稽核紀錄、server-side data cleaning、cache、以及預先清理好的臨床資料 recipes。

**為什麼重寫而非 wrap**：現有 QueryMRNService 有多個限制——查詢天數/內容 hardcode、回傳的 raw data 需要大量 client-side cleaning（參考 handoff-cli 的 300+ 行 parsing）、完全沒有認證機制。Gateway 模式只是在問題上面加一層，不解決根本問題，還多一跳延遲和一個需要維運的 process。

**專案位置**: `D:\CC\repos\slh-agent-portal` (獨立 repo)
**儲存**: SQLite (better-sqlite3)
**資料來源**: Sinlau HIS DLLs (直接呼叫) + AS400 SLLIB (透過 DLLs)

---

## Architecture Overview

```
                 ┌─────────────┐
  CLI (AI agent) │ portal cmd  │──┐
                 └─────────────┘  │    ┌──────────────────────────────────────┐
                                  ├───>│  Agent Portal (5112)                  │
                 ┌─────────────┐  │    │  Hono + SQLite                        │
  Future Web UI  │ browser     │──┘    │                                       │
                 └─────────────┘       │  ┌─────────┐  ┌────────┐  ┌───────┐  │    ┌──────────────┐
                                       │  │  Auth    │  │ Cache  │  │ Audit │  │    │ Sinlau DLLs  │
                                       │  │ API Key  │  │ LRU    │  │  Log  │  │───>│ + AS400 HIS  │
                                       │  └─────────┘  └────────┘  └───────┘  │    └──────────────┘
                                       │  ┌──────────────────────────────────┐ │
                                       │  │ Recipes (data cleaning + cache)  │ │
                                       │  └──────────────────────────────────┘ │
                                       └──────────────────────────────────────┘
```

**注意**：過渡期間 portal 仍可 proxy 到舊 QueryMRNService (5111)，逐步將端點遷移到直接 DLL 呼叫。這樣可以漸進式替換，不用一次全部重寫。

---

## Project Structure

```
D:\CC\repos\slh-agent-portal\
├── src/
│   ├── index.ts                    # Server entry (Hono on port 5112)
│   ├── cli.ts                      # CLI entry (subcommand dispatcher)
│   ├── config.ts                   # ENV loading
│   │
│   ├── server/
│   │   ├── app.ts                  # Hono app factory + middleware chain
│   │   ├── middleware/
│   │   │   ├── auth.ts             # API key validation -> identity on ctx
│   │   │   ├── audit.ts            # Post-response audit log insert
│   │   │   ├── permissions.ts      # Group permission check
│   │   │   └── error-handler.ts    # Structured JSON error responses
│   │   └── routes/
│   │       ├── health.ts           # GET /health (portal + upstream)
│   │       ├── patient.ts          # GET /api/patient, /api/patients, /api/discharged
│   │       ├── notes.ts            # GET /api/notes
│   │       ├── recipes.ts          # GET /api/recipes/:name
│   │       └── admin.ts            # API key & group CRUD, audit query
│   │
│   ├── his/
│   │   ├── client.ts               # Unified HIS client (proxy to 5111 during migration)
│   │   ├── client-direct.ts        # Direct DLL calls via child_process (replaces QueryMRNService)
│   │   ├── types.ts                # ClinicalData, PatientSummary interfaces
│   │   └── cache.ts                # LRU cache layer (cleaned data, TTL-based)
│   │
│   │
│   ├── recipes/
│   │   ├── registry.ts             # Recipe name -> handler map
│   │   ├── base.ts                 # Recipe interface + date range + classifyVisitType
│   │   ├── formatters.ts           # --format prompt 輸出格式化 (LLM-optimized text)
│   │   ├── # -- 住院 --
│   │   ├── admission-vitals.ts     # Vitals + I/O + glucose + ICD dx + allergies
│   │   ├── weekly-orders.ts        # Active orders + drug categories + changes
│   │   ├── lab-summary.ts          # Lab 摘要 + abnormal flags + delta
│   │   ├── progress-notes.ts       # Progress/consult notes by date range
│   │   ├── discharge-summary.ts    # 出院病摘 (I01) parsed by section type
│   │   ├── nursing-notes.ts        # 住院護理交班 + 護理評估 (sections 16-17)
│   │   ├── # -- 急診 --
│   │   ├── er-nursing-3d.ts        # 三日內急診護理紀錄
│   │   ├── er-visit-summary.ts     # 完整急診摘要 (incl. erCaseExport)
│   │   ├── # -- 門診 --
│   │   ├── opd-visit-summary.ts    # 門診就診摘要
│   │   ├── opd-medication.ts       # 門診用藥 (Phase 3b)
│   │   ├── # -- Task recipes (AI agent 任務導向組合) --
│   │   ├── task-progress-note.ts   # 寫 progress note 所需全部資料
│   │   ├── task-discharge.ts       # 寫出院病摘所需全部資料
│   │   ├── task-handoff.ts         # 交班所需全部資料
│   │   └── task-consult-reply.ts   # 回覆會診所需全部資料
│   │
│   ├── db/
│   │   ├── connection.ts           # SQLite singleton (better-sqlite3)
│   │   ├── schema.ts               # DDL + seed default groups
│   │   ├── api-keys.ts             # CRUD for api_keys
│   │   ├── groups.ts               # CRUD for groups
│   │   └── audit.ts                # Insert + query audit_logs
│   │
│   ├── auth/
│   │   ├── api-key.ts              # Key generation (slhp_ prefix), SHA-256 hashing
│   │   └── oauth-stub.ts           # /oauth/* routes -> 501 placeholder
│   │
│   └── cli/
│       ├── commands/
│       │   ├── patient.ts          # portal patient --mrn X [--csn Y]
│       │   ├── patients.ts         # portal patients --dr X
│       │   ├── vitals.ts           # portal vitals --mrn X [--days N]
│       │   ├── labs.ts             # portal labs --mrn X [--days N]
│       │   ├── orders.ts           # portal orders --mrn X [--active|--week]
│       │   ├── notes.ts            # portal notes --mrn X [--days N] [--type X]
│       │   ├── recipe.ts           # portal recipe <name> --mrn X [--from --to]
│       │   └── admin.ts            # portal admin create-key|list-keys|...
│       ├── parser.ts               # Minimal arg parser (no external dep)
│       └── output.ts               # JSON / text / table formatters
│
├── data/                           # SQLite DB file (gitignored)
├── portal.bat                      # CLI launcher (calls init-env.bat)
├── portal-server.bat               # Server launcher
├── package.json
├── tsconfig.json
├── esbuild.config.ts               # Two entrypoints: server + cli
├── .gitignore
└── CLAUDE.md
```

---

## Key Design Decisions

### 1. Auth: API Key (per-person，OAuth 結構保留但不啟用)

- Key 格式: `slhp_` + 32 bytes hex (64 chars) = 69 chars total
- 只存 SHA-256 hash，raw key 只在建立時顯示一次
- CLI 讀 `PORTAL_API_KEY` env var 或 `~/.portal/config.json`
- Header: `Authorization: Bearer slhp_...`
- OAuth routes (`/oauth/*`) 回傳 501，結構預留給未來 LDAP/AD 整合

**Per-person 規範（強制）**：
- 每把 key 必須綁定一個人，`label` 格式: `<員工編號>-<用途>`（如 `4303-ward-ai`、`N001-nursing-agent`）
- 禁止共用 key — `portal admin create-key` 時 `--label` 為必填
- 這樣 audit log 可追溯到個人，滿足 PHI 合規需求

**現狀對比**：slh-his inpatient app 的 `auth.ts` 有 HMAC session cookie + TOTP 機制，但實際上**從未啟用**（所有 import 被註解掉，沒有登入頁面）。QueryMRNService 完全沒有 auth。Portal 的 API key 是首次引入的真正認證機制。

### 2. Permission Model

群組 → 權限字串陣列，支援 wildcard：

| Group | Permissions | 用途 |
|-------|------------|------|
| `admin` | `*` | 系統管理 |
| `attending` | `patient:*`, `notes:*`, `recipe:*` | 主治醫師 |
| `resident` | `patient:read`, `patient:list`, `notes:read`, `recipe:*` | 住院醫師 |
| `nurse` | `patient:read`, `notes:read`, `recipe:er-nursing-3d`, `recipe:admission-vitals`, `recipe:nursing-notes` | 護理師 |
| `pharmacist` | `patient:read`, `patient:list`, `recipe:weekly-orders`, `recipe:lab-summary`, `recipe:admission-vitals` | 藥師 |
| `ai-agent` | `patient:read`, `patient:list`, `notes:read`, `recipe:*` | AI agent (通用) |

Permission format: `<resource>:<action>` (e.g., `patient:read`, `recipe:weekly-orders`)

**最小權限原則**：預設的 `ai-agent` group 有 `recipe:*`，但生產環境中**不應使用通用 ai-agent group**。應按任務建立專用 group（用 `portal admin create-group`），只給需要的 recipe：

- `ai-notes`: `patient:read`, `recipe:admission-vitals`, `recipe:lab-summary`, `recipe:weekly-orders`, `recipe:progress-notes`, `recipe:task-progress-note`
- `ai-discharge`: `patient:read`, `notes:read`, `recipe:discharge-summary`, `recipe:lab-summary`, `recipe:weekly-orders`, `recipe:task-discharge`
- `ai-handoff`: `patient:read`, `patient:list`, `recipe:admission-vitals`, `recipe:weekly-orders`, `recipe:task-handoff`

`ai-agent` group 僅用於開發/測試。`CLAUDE.md` 中應明確記載此規範。

### 3. Audit Log

記錄每次 API 呼叫：who (key_id + label + group), what (endpoint + MRN), when (timestamp), result (status code + duration)。**絕不記錄臨床資料內容**（PHI 合規）。

### 4. Cache 層

Portal server 在記憶體中 cache 已清理好的資料（不是 raw upstream response），減少重複查詢 AS400。

**設計**：
- LRU cache（Map-based，無外部依賴），key = `mrn:dataType:dateRange`
- Cache 的是 **recipe output**（已經過 data cleaning 的結構化 JSON），不是 raw HIS response
- 每筆 cache entry 含 `cachedAt` timestamp，API response 一律帶 `_meta.cachedAt` 欄位

**TTL 策略**：

| 資料類型 | TTL | 理由 |
|----------|-----|------|
| 住院 vitals / I/O | 15 分鐘 | 頻繁變動 |
| Labs | 30 分鐘 | 新結果出來才變 |
| Active orders | 30 分鐘 | 醫囑異動頻率中等 |
| Progress notes | 1 小時 | 一天寫 1-2 次 |
| 門診/出院紀錄 | 4 小時 | 近乎靜態 |
| 病人清單 (patients --dr) | 5 分鐘 | 入出院會改變 |

**Cache 控制**：
- `?nocache=1` query param 強制 bypass cache（開發/調試用）
- `portal admin cache-clear` CLI 命令清空全部 cache
- Server 啟動時 cache 為空（不持久化）
- Cache 上限：1000 entries，超過 LRU 淘汰

**Batch mode 的 cache 效益**：`portal recipe lab-summary --dr 4303` 會先查病人清單（1 次），再對每位病人查 labs。如果 5 分鐘內再跑一次，病人清單 cache hit，個別 labs 如果在 30 分鐘內也 cache hit。

### 5. Error Response Contract

所有 API error 回傳統一格式，讓 AI agent 能自動判斷是否 retry：

```json
{
  "error": {
    "code": "UPSTREAM_TIMEOUT",
    "message": "QueryMRNService did not respond within 20s",
    "retryable": true
  }
}
```

**Error codes**：

| Code | HTTP Status | Retryable | 說明 |
|------|------------|-----------|------|
| `AUTH_MISSING` | 401 | false | 沒有 API key |
| `AUTH_INVALID` | 401 | false | Key 不存在或已撤銷 |
| `PERMISSION_DENIED` | 403 | false | 權限不足 |
| `PATIENT_NOT_FOUND` | 404 | false | MRN 不存在 |
| `UPSTREAM_TIMEOUT` | 504 | true | HIS 查詢逾時 |
| `UPSTREAM_ERROR` | 502 | true | HIS 回傳錯誤 |
| `VISIT_TYPE_UNKNOWN` | 422 | false | 無法判別就診類型 |
| `RECIPE_PARTIAL` | 207 | false | Task recipe 部分成功（見下方） |

### 6. Recipes (資料清理 + 聚合)

每個 recipe 是一個 pure function，接收 `RecipeParams` + `HisClient`，回傳結構化 JSON：

**住院 Recipes:**

| Recipe | 說明 | 資料來源 |
|--------|------|----------|
| `admission-vitals` | 本次住院 vitals + I/O + glucose + **ICD dx + allergies** | tprLast24hr + tprFull + ioData + icdDiagnosis + allergies |
| `weekly-orders` | 這週 active orders + 藥物分類 + 異動 | orderHistoryJson (parsed + categorized) |
| `lab-summary` | Lab 摘要 + 異常標記 + delta 趨勢 | labResults (parsed, grouped by test type) |
| `progress-notes` | Progress/consult notes 依日期篩選 | /notes endpoint with date filter |
| `discharge-summary` | 出院病摘 (I01) 依 section type 分段 | dischargeSummary + icdDiagnosis + allergies |
| `nursing-notes` | 住院護理紀錄 (護理交班 + 護理評估) | nursingShift + nursingCareNotes (72hr) |

**急診 Recipes:**

| Recipe | 說明 | 資料來源 |
|--------|------|----------|
| `er-nursing-3d` | 三日內急診護理紀錄 | erNursingNotes (filtered to 3 days) |
| `er-visit-summary` | 急診就診完整摘要 | erTriage + erNursingNotes + erPresentIllness + **erCaseExport** + labResults + icdDiagnosis + allergies |

**門診 Recipes:**

| Recipe | 說明 | 資料來源 | 狀態 |
|--------|------|----------|------|
| `opd-visit-summary` | 門診就診摘要 (主訴、診斷、處置) | opdSessionRecord + icdDiagnosis + labResults | 可用 (現有資料) |
| `opd-medication` | 門診用藥清單 + 慢箋 | 需 QueryMRNService 新增 OPD 處方查詢 | Phase 3b |

**Task Recipes (AI agent 任務導向 — 組合多個 base recipes):**

| Task Recipe | 打包資料 | 用途 |
|-------------|----------|------|
| `task-progress-note` | vitals(24hr) + labs(48hr) + active orders + previous progress note + I/O | AI 寫 daily progress note |
| `task-discharge-summary` | admission info + discharge-summary draft + all labs + order history + ICD dx + consult notes | AI 寫出院病摘 |
| `task-handoff` | vitals(24hr) + order changes(48hr) + I/O + nursing shift + active meds | AI 做交班 |
| `task-consult-reply` | admission info + labs + progress notes + orders + icdDiagnosis | AI 回覆會診 |

Task recipes 是 base recipes 的組合，不是新的資料來源。它們為 AI agent 的特定任務打包最佳資料組合，減少 agent 自己要判斷「該查什麼」的負擔。

**Task recipe 的 partial failure 處理**：

Task recipe 內部呼叫多個 base recipe，使用 `Promise.allSettled`（不是 `Promise.all`），確保單一 sub-recipe 失敗不會導致整個 task recipe 失敗。回傳格式包含 `_meta.completeness`：

```json
{
  "_meta": {
    "completeness": {
      "vitals": "ok",
      "labs": "ok", 
      "orders": "error:UPSTREAM_TIMEOUT",
      "previousNote": "ok",
      "io": "ok"
    },
    "complete": false,
    "cachedAt": "2026-04-01T10:30:00Z"
  },
  "vitals": { ... },
  "labs": { ... },
  "orders": null,
  "previousNote": { ... },
  "io": { ... }
}
```

HTTP status 為 207 (Multi-Status)，error code 為 `RECIPE_PARTIAL`。AI agent 可根據 `completeness` 決定是否繼續生成（例如 vitals + labs OK 但 orders 失敗時，可能仍能寫 progress note，但需要標註資料不完整）。

**Sub-recipe 並行呼叫**：Task recipe 內部的 sub-recipe 使用 `Promise.allSettled` 並行執行，搭配 cache 命中時幾乎 instant。預估 latency：
- 全部 cache hit: < 50ms
- 全部 cache miss: 3-8 秒（並行，取決於最慢的上游查詢）
- 混合: 1-3 秒

**Batch mode concurrency limit**：`--dr 4303` 查全部病人時，並行上限 3 個病人（避免 flood 上游）。

**急診/門診辨識邏輯 (重要 — 已修正時間陷阱):**

在 HIS 系統中，急診和門診的掛號結構類似 — 都使用 `erVdt`/`erVtm`（就診日期時間）作為查詢 key。

**陷阱**: `slh_er_triage` 儲存所有歷史 ER 紀錄。QueryMRNService 用 `TOP 1 ORDER BY vdt DESC` 查詢，三年前的急診紀錄也會被撈出。必須加上時間匹配：

```typescript
function classifyVisitType(data: ClinicalData, visitDate?: string): "er" | "opd" | "inpatient" | "unknown" {
  // 1. 有住院 CSN → 住院
  if (data.admissionInfo && !data.admissionInfo.includes("No current admission")) return "inpatient";
  
  // 2. erTriage 存在且日期匹配本次就診 → 急診
  if (data.erTriage && data.erTriage !== "(no rows)") {
    const triageDate = extractDateFromTriage(data.erTriage);
    if (!visitDate || triageDate === visitDate) return "er";
    // Triage 存在但日期不匹配 → 是舊的急診紀錄，不算
  }
  
  // 3. 有 opdSessionRecord → 門診
  if (data.opdSessionRecord && data.opdSessionRecord !== "") return "opd";
  
  return "unknown"; // 無法判別 — recipe 層應拒絕處理，回傳 VISIT_TYPE_UNKNOWN error
}
```

**重要**：fallback 不再是 `"inpatient"`。當 visit type 為 `"unknown"` 時，recipe 層回傳 `VISIT_TYPE_UNKNOWN` error（HTTP 422），而非靜默猜測並產出垃圾資料。AI agent 收到此 error 後可要求使用者手動指定 visit type。

共用資料（不分急診/門診）：labResults, icdDiagnosis, allergies — 都用同一個 `erVdt`/`erVtm` 查詢。

Recipes 根據 visit type 自動選擇正確的資料欄位組合，CLI 輸出也會標示就診類型。

**門診資料源擴充需求 (Phase 3b — 需修改 QueryMRNService.cs):**

QueryMRNService 目前主要為住院流程設計。門診需要以下新端點/資料：

| 新端點 | SQL 來源 (AS400 SLLIB) | 說明 |
|--------|------------------------|------|
| `GET /opd-visits?mrn=X&days=N` | `T01L12`/`T01L13`/`T01L14`/`T01L16` — 門診掛號表 | 近期門診就診紀錄清單 (T01F04 NOT '2A' = 門診, '2A' = 急診) |
| `GET /opd-prescriptions?mrn=X&visitDate=X` | `OOORDL7` — 門診醫囑/處方表 | 門診處方 (ORDCHT=MRN, ORDDLT='', ORDDSP='Y') |
| `GET /opd-soap?mrn=X&visitDate=X` | `OOSOAL1` — SOAP 紀錄 | 門診 SOAP notes (SOPSTS='', SOPTYP='S') |

**已確認的門診用藥相關 AS400 tables:**

| Table | 用途 | Key Columns |
|-------|------|-------------|
| `SLLIB.OOORDL7` | **門診醫囑/處方** (核心) | ORDCHT(MRN), ORDDAT, ORDTIM, ORDDOC(醫師), ORDCDE(藥品碼), ORDDES(藥名), ORDUNT(單位), ORDUSE(用法) |
| `SLLIB.OOSOAL1` | 門診 SOAP notes | SOPCHT(MRN), SOPDAT, SOPTIM, SOPTXT(內容) |
| `SLLIB.OOSOAP` | 門診 SOAP (被 OutpatientRecordExport.dll 使用) | — |
| `SLLIB.ORRGTL1` | 醫囑掛號 | RGTCHT(MRN), RGTDAT, RGTTIM |
| `SLLIB.M02` | **藥品/項目主檔** | M02F01(藥品碼), M02NAM(藥名), M02F33(途徑), M02DRT(劑型) |
| `SLLIB.NSARDL9` | 藥品醫囑 (住院+門診共用) | 被 OHExportText.dll 使用 |
| `SLLIB.SLDRGL1` | 藥品清單 | 被 OrderHistory 使用 |
| `SLLIB.T01L16` | **門診掛號主檔** | T01F11(MRN), T01DAT(就診日), T01F14(時間), T01F15(醫師碼), T01F04(類別: NOT '2A'=門診) |

**注意:** 慢箋沒有獨立 table — 以 `OOORDL7` 中的 duration/frequency 欄位區分（需與資訊室確認具體欄位名）。

Portal 的 recipe 架構已預留擴充點 — 新增 recipe 只需：
1. 建立 `src/recipes/opd-xxx.ts` 實作 `Recipe` interface
2. 在 `registry.ts` 註冊
3. 對應的 `his/client.ts` 加新的 fetch function

Recipe 共同特性：支援 `--from`/`--to` 日期範圍或 `--days N` 快捷參數。

---

## SQLite Schema

```sql
CREATE TABLE groups (
  id          TEXT PRIMARY KEY,
  name        TEXT NOT NULL UNIQUE,
  description TEXT,
  permissions TEXT NOT NULL,  -- JSON array: ["patient:*", "notes:*"]
  created_at  TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ','now')),
  updated_at  TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ','now'))
);

CREATE TABLE api_keys (
  id          TEXT PRIMARY KEY,
  key_hash    TEXT NOT NULL UNIQUE,
  label       TEXT NOT NULL,
  group_id    TEXT NOT NULL REFERENCES groups(id),
  is_active   INTEGER NOT NULL DEFAULT 1,
  created_at  TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ','now')),
  last_used   TEXT,
  expires_at  TEXT
);

CREATE TABLE audit_logs (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  timestamp   TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ','now')),
  key_id      TEXT NOT NULL,
  key_label   TEXT NOT NULL,
  group_name  TEXT NOT NULL,
  method      TEXT NOT NULL,
  path        TEXT NOT NULL,
  mrn         TEXT,
  csn         TEXT,
  recipe_name TEXT,
  status_code INTEGER NOT NULL,
  duration_ms INTEGER NOT NULL,
  ip_address  TEXT,
  user_agent  TEXT
);
-- Indexes on timestamp, mrn, key_id
```

Seed 6 個預設 groups (admin, attending, resident, nurse, pharmacist, ai-agent)。
首次啟動自動建立 admin API key 並印到 stdout。

---

## API Endpoints

| Method | Path | Permission | Description |
|--------|------|------------|-------------|
| GET | `/health` | (none) | Portal + upstream health |
| GET | `/api/patient?mrn=X&csn=Y` | `patient:read` | 完整臨床資料 (proxy) |
| GET | `/api/patients?dr=X` | `patient:list` | 醫師在院病人清單 |
| GET | `/api/discharged?dr=X&days=N` | `patient:list` | 近期出院清單 |
| GET | `/api/notes?mrn=X&csn=Y&days=N&type=X` | `notes:read` | Progress/consult notes |
| GET | `/api/recipes/:name?mrn=X&days=N&from=X&to=X` | `recipe:<name>` | 執行 recipe |
| GET | `/api/recipes` | (authenticated) | 列出可用 recipes |
| POST | `/api/admin/keys` | `admin:keys` | 建立 API key |
| GET | `/api/admin/keys` | `admin:keys` | 列出 API keys |
| DELETE | `/api/admin/keys/:id` | `admin:keys` | 撤銷 API key |
| POST | `/api/admin/groups` | `admin:groups` | 建立/更新 group |
| GET | `/api/admin/groups` | `admin:groups` | 列出 groups |
| GET | `/api/admin/audit?since=X&mrn=X&key_id=X` | `admin:audit` | 查詢 audit log |

---

## CLI Commands

```bash
# 基本查詢（proxy 到 QueryMRNService，回傳原始資料）
portal patient --mrn 12345 [--csn 67890]
portal patients --dr 4303

# Recipes（清理 + 結構化資料，建議 AI agent 優先用這些）
# 住院
portal recipe admission-vitals --mrn 12345 [--days 3]
portal recipe weekly-orders --mrn 12345
portal recipe lab-summary --mrn 12345 [--days 7]
portal recipe progress-notes --mrn 12345 [--from 2026-03-25 --to 2026-04-01]
portal recipe discharge-summary --mrn 12345
portal recipe nursing-notes --mrn 12345 [--days 3]

# 急診
portal recipe er-nursing-3d --mrn 12345
portal recipe er-visit-summary --mrn 12345

# 門診
portal recipe opd-visit-summary --mrn 12345 [--days 30]

# Task recipes（AI agent 任務導向 — 一次取得任務所需全部資料）
portal recipe task-progress-note --mrn 12345
portal recipe task-discharge --mrn 12345
portal recipe task-handoff --mrn 12345
portal recipe task-consult-reply --mrn 12345

# Batch mode（一次查醫師所有病人）
portal recipe lab-summary --dr 4303              # 所有在院病人的 lab
portal recipe admission-vitals --mrn 12345,67890 # 多個 MRN

# Output format
portal recipe lab-summary --mrn 12345 --format json    # default, structured JSON
portal recipe lab-summary --mrn 12345 --format text    # human readable table
portal recipe lab-summary --mrn 12345 --format prompt  # LLM-optimized text with section headers

# 管理（需 admin group 的 key）
portal admin create-key --label "ward-ai-agent" --group ai-agent
portal admin list-keys
portal admin revoke-key --id <key-id>
portal admin create-group --name custom --permissions "patient:read,recipe:*"
portal admin list-groups
portal admin audit [--since 2026-03-01] [--mrn X] [--limit 100]

# Output format
portal patient --mrn 12345 --format json   # default, for AI agents
portal patient --mrn 12345 --format text   # human readable
```

---

## Implementation Phases

### Phase 1: Foundation + HIS Client
1. Init repo: `package.json`, `tsconfig.json`, `esbuild.config.ts`, `.gitignore`, `CLAUDE.md`
2. `src/config.ts` — env loading (PORTAL_PORT, PORTAL_DB, HIS_SERVICE_URL)
3. `src/db/connection.ts` + `src/db/schema.ts` — SQLite init + DDL + seed groups
4. `src/his/client.ts` — **過渡期 proxy client**（async HTTP 到舊 QueryMRNService 5111）
   - Port from: `slh-his/apps/inpatient/app/lib/his-client.ts` (remove `server-only`, add configurability)
   - 所有查詢參數化（days、content type 等不再 hardcode）
   - Reuse types: `ClinicalData`, `PatientSummary`, `NotesResult`
5. `src/his/cache.ts` — LRU cache layer（TTL-based，見 Design Decisions #4）
6. `src/auth/api-key.ts` — key generation + hashing
7. `src/his/types.ts` — 統一 type definitions

### Phase 2: Server + Auth + Audit + Error Handling
1. `src/db/api-keys.ts`, `src/db/groups.ts`, `src/db/audit.ts` — CRUD
2. `src/server/middleware/auth.ts` — Bearer token -> identity resolution
3. `src/server/middleware/permissions.ts` — group permission check with wildcard
4. `src/server/middleware/audit.ts` — post-response logging
5. `src/server/middleware/error-handler.ts` — **統一 error response format**（見 Design Decisions #5）
6. `src/server/routes/health.ts`, `patient.ts`, `notes.ts`, `admin.ts`
7. `src/server/app.ts` + `src/index.ts` — wire up Hono
8. `src/auth/oauth-stub.ts` — 501 placeholder
9. `portal.bat` + `portal-server.bat` — Windows launchers with `init-env.bat`（提前到 Phase 2，方便測試）

### Phase 3: Recipes — Data Cleaning on Server
1. `src/recipes/base.ts` — Recipe interface, date range helpers, `classifyVisitType()` (fallback = `"unknown"`)
2. `src/recipes/registry.ts` — Recipe registry + list
3. `src/recipes/formatters.ts` — `--format prompt` 輸出格式化（port from handoff-cli prompt builder）
4. **住院:**
   - `admission-vitals.ts` — TPR + I/O + glucose + ICD dx + allergies (port from query-mrn.ts)
   - `weekly-orders.ts` — orderHistoryJson parsed (port extractOrderChanges + parseOrderHistoryMeds + drug-categories.json)
   - `lab-summary.ts` — labResults parsed, grouped by test type, abnormal flags, delta
   - `progress-notes.ts` — /notes endpoint with date filter, clean formatting
   - `discharge-summary.ts` — I01 parsed by section type (200/300/320/340/360/380/400/500/800)
   - `nursing-notes.ts` — nursingShift + nursingCareNotes (sections 16-17, 72hr)
5. **急診:**
   - `er-nursing-3d.ts` — erNursingNotes filtered to 3 days
   - `er-visit-summary.ts` — triage + nursing + PI + erCaseExport + labs + dx + allergies
6. **門診:**
   - `opd-visit-summary.ts` — opdSessionRecord + labs + dx (自動辨識非急診)
7. **Task recipes** (組合 base recipes，用 `Promise.allSettled` + `completeness`):
   - `task-progress-note.ts` — vitals(24hr) + labs(48hr) + orders + prev note + I/O
   - `task-discharge.ts` — admission + discharge-summary + labs + orders + consult notes
   - `task-handoff.ts` — vitals(24hr) + order changes(48hr) + I/O + nursing + meds
   - `task-consult-reply.ts` — admission + labs + progress notes + orders + dx
8. `src/server/routes/recipes.ts` — route handler with `--dr` batch support（concurrency limit 3）

### Phase 4: CLI + Polish
1. `src/cli/parser.ts` — minimal arg parser (hand-rolled, no deps)
2. `src/cli/output.ts` — JSON / text / table formatters
3. `src/cli/commands/*.ts` — one file per command (patient, patients, vitals, labs, orders, notes, recipe, admin)
4. `src/cli.ts` — dispatcher entry point
5. First-run: auto-create admin key, print to stdout
6. `CLAUDE.md` for the portal repo
7. Health endpoint checks portal DB + upstream connectivity

### Phase 5: QueryMRNService 重寫 (漸進式替換)

**策略**：不一次重寫，而是逐步將端點從「proxy 到 5111」遷移到「直接 DLL 呼叫」。

1. `src/his/client-direct.ts` — 直接呼叫 Sinlau DLLs (via child_process + .NET helper)
   - 從 QueryMRNService.cs 的 SQL/DLL 邏輯 port 過來
   - 查詢參數全部可配（days、content type、date range）
   - 不再 hardcode 查詢天數
2. **逐端點遷移**（每個端點獨立切換，不影響其他端點）：
   - `client.ts` 加入 feature flag：`USE_DIRECT_<endpoint>` env var
   - 遷移順序建議：`/patients` → `/patient` → `/notes` → 其他
   - 每遷移一個端點，用 A/B 比對驗證結果一致
3. **門診資料直接查 AS400**（不需等資訊室改 QueryMRNService）：
   - `fetchOpdVisits()` — 查 `SLLIB.T01L16` (T01F04 NOT '2A' = 門診)
   - `fetchOpdPrescriptions()` — 查 `SLLIB.OOORDL7`
   - `fetchOpdSoap()` — 查 `SLLIB.OOSOAL1`
   - 藥品名稱 join `SLLIB.M02`
4. `src/recipes/opd-medication.ts` — 門診用藥清單 + 慢箋辨識
5. 完成後可停用 QueryMRNService (5111)

---

## Key Files to Reference During Implementation

| Purpose | File | What to port/reuse |
|---------|------|--------------------|
| HIS HTTP client (Phase 1 proxy) | `slh-his/apps/inpatient/app/lib/his-client.ts` | fetchJson, fetchPatientList, fetchClinicalData, fetchNotes |
| HIS DLL 呼叫邏輯 (Phase 5 rewrite) | `QueryMRNService.cs` (資訊室) | SQL queries, DLL call patterns, AS400 table schema |
| Clinical data types | `slh-his/legacy/handoff-cli/src/types.ts` | VitalReading, OrderChange, Medication, MedCategory, Patient |
| Order parsing + data cleaning | `slh-his/legacy/handoff-cli/src/query-mrn.ts` | extractOrderChanges(), parseOrderHistoryMeds() — 300+ 行 parsing 邏輯 |
| Drug classification | `slh-his/legacy/handoff-cli/src/drug-categories.json` | keyword -> MedCategory mapping |
| Auth reference (HMAC pattern) | `slh-his/apps/inpatient/app/lib/auth.ts` | timing-safe compare pattern（注意：此 auth 目前未啟用） |
| esbuild config | `slh-his/legacy/handoff-cli/package.json` | Build script pattern (platform=node, target=node18) |
| CLI arg pattern | `slh-his/legacy/handoff-cli/src/index.ts` | process.argv parsing pattern |

---

## Verification Plan

1. **Build**: `npm run build` produces `dist/index.js` (server) + `dist/cli.js` (CLI)
2. **Server start**: `portal-server.bat` starts on 5112, `GET /health` returns ok
3. **Admin key**: First run prints admin API key to stdout
4. **Auth test**: Request without key -> 401; with valid key -> 200; with revoked key -> 401
5. **Permission test**: pharmacist key on admin endpoint -> 403; on recipe:weekly-orders -> 200
6. **Patient list test**: `GET /api/patients?dr=4303` with valid key returns patient list
7. **Recipe test**: `GET /api/recipes/admission-vitals?mrn=<test>` returns structured JSON with vitals + ICD dx + allergies
8. **Task recipe test**: `GET /api/recipes/task-progress-note?mrn=<test>` returns bundled data with `_meta.completeness`
9. **Partial failure test**: Mock upstream timeout on one sub-recipe → task recipe returns 207 with partial data + completeness metadata
10. **Cache test**: 同一 recipe 查兩次，第二次 `_meta.cachedAt` 有值且 response time < 50ms
11. **Format test**: `--format prompt` returns LLM-optimized text with section headers
12. **Batch test**: `portal.bat recipe lab-summary --dr 4303` returns array for all patients
13. **ER classification**: Test with known ER patient → `er-visit-summary` works; OPD patient → `opd-visit-summary`; unknown → 422 VISIT_TYPE_UNKNOWN
14. **Error format test**: 各種錯誤情境回傳統一的 `{ error: { code, message, retryable } }` 格式
15. **Audit test**: `GET /api/admin/audit` shows all previous requests with MRN + recipe name
16. **CLI test**: `portal.bat recipe task-handoff --mrn <test> --format prompt` outputs LLM-ready text

---

## Testing Strategy

**Unit tests**（Phase 3 同步進行）：
- Recipe parsing 邏輯是 pure function，非常適合 unit test
- **優先**：`extractOrderChanges()` 和 `parseOrderHistoryMeds()` 有大量 edge case（單一藥品 vs. 陣列、`@Del` 旗標、EndTime 解析失敗）
- Port 之前先用現有 raw data 建立 test fixture（從 QueryMRNService 截取真實回傳，移除 PHI）
- `classifyVisitType()` 的 edge case（空資料、日期不匹配、同時有 ER + OPD 紀錄）
- Cache TTL 和 LRU 淘汰邏輯

**Integration tests**（Phase 4）：
- Mock HIS client（回傳 fixture data），測 recipe 端對端
- Auth + permission middleware chain（401/403/200 scenarios）
- Error handler（upstream timeout、partial failure 的 HTTP status 和 response format）

**不做**：E2E browser test（沒有 web UI），QueryMRNService live test（依賴院內環境）。

---

## Schema Migration

初期（Phase 1-4）使用簡易方案：

- `src/db/schema.ts` 中加 `schema_version` table（單一 row，記錄目前版本號）
- 每次啟動時比對版本號，版本不匹配時依序執行 migration SQL
- Migration SQL 寫在 `src/db/migrations/` 目錄下，檔名格式 `002-add-response-hash.sql`
- **不用 migration library**（過度複雜），手寫 `ALTER TABLE` 即可
- SQLite 的 `ALTER TABLE` 限制：不能 DROP COLUMN、不能改 type。需要時用 create-copy-drop-rename pattern。

---

## Deployment & Monitoring

**啟動方式**：
- `portal-server.bat` 透過 Task Scheduler 設為開機自動啟動
- Task Scheduler 設定：失敗時自動重啟（最多 3 次，間隔 30 秒）
- stdout/stderr 導向 `data/portal.log`（日期 rotate）

**Health check**：
- `/health` endpoint 在 Phase 2 就實作（不等到 Phase 4）
- 回傳 portal DB status + upstream HIS connectivity
- 可用 Task Scheduler 另設一個每 5 分鐘跑的 health check script，失敗時寫 log 告警

**監控（初期簡易方案）**：
- Portal crash → Task Scheduler 自動重啟 + 寫 Windows Event Log
- Audit log 本身就是 monitoring 資料來源（查 status_code 5xx 的頻率）
- 未來可加 `/metrics` endpoint 給 Grafana 用

---

## Future Expansion (Out of Scope for Phase 1-5)

- **OAuth + LDAP/AD**: 當 user database 建立後，啟用 oauth-stub 中的 routes（`slh-his/packages/core/auth` 已有規劃但尚未開發）
- **Web UI**: 管理介面 + 資料瀏覽器
- **Rate limiting**: 防止 AI agent 過度查詢（目前 batch mode 的 concurrency limit 3 是初步保護）
- **Circuit breaker**: 連續 N 次 upstream failure 後 fast-fail M 秒（簡易計數器，~50 行 TS）
- **Audit log retention**: 超過 50 萬筆自動 archive 到 `audit_logs_YYYYMM.db`
- **Response hash**: audit log 加 `response_sha256` 欄位（不記錄 PHI 但可驗證 reproducibility）
- **WebSocket**: 即時通知（新入院等）
- **Row-level security**: 限制特定 key 只能查特定醫師的病人（`key → dr_id` 綁定）
- **門診轉診/會診紀錄**: 需確認 NIS/SLTHIS 轉介表結構
