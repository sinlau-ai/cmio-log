# Inpatient LLM Documentation App — Implementation Plan

## Context

Li-yang wants to build an A+-like (奇美醫院) inpatient clinical documentation webapp for Sin-lau Hospital, powered by LLM AI generation. The app reuses `medical-ai-playground` UI patterns, connects exclusively through `agent-portal` (the HIS data gateway), and lives inside `slh-his/apps/inpatient` (existing folder). Both apps deploy on the same Windows PC on hospital LAN.

**Key insight**: Unlike the current `medical-ai-playground` (copy-paste raw text → AI), this app pulls structured clinical data directly from HIS via agent-portal, then uses it as context for LLM note generation. This is the clinical AI pipeline pattern used by DAX Copilot, Abridge, and Epic AI.

---

## Note Types & Data Requirements Matrix

Each note type needs different clinical data. This matrix drives which agent-portal endpoints to call per note type.

| Data Source | Admission | Progress | Transfer/Accept | Weekly Summary | Treatment Course | Working Dx | Handover (multi-pt) |
|---|---|---|---|---|---|---|---|
| **info** | full | brief | brief | - | full | - | one-liner |
| **vitals** | latest | 24-48h detail | latest | 7d trends | - | - | latest only |
| **labs** | recent 7d | 48h detail + flags | recent 3d | 7d trends | key abnormals | relevant | key recent |
| **io** | - | 24h detail | recent 24h | 7d daily totals | - | - | latest balance |
| **meds** | home + active | active (abx/vaso highlight) | active | all + changes this week | full timeline | - | key active |
| **orders** | admission orders | active | active + pending | changes this week | major only | - | - |
| **notes** | ER notes if available | last progress note | recent notes | all notes this week | admission + key milestones | recent assessment | - |
| **dx** | full ICD list | full list | full list | full list | full list | full + evidence | brief |
| **allergies** | full | - | full | - | full | - | - |
| **surgeries** | - | - | if applicable | if occurred | full history | - | - |

### Agent-Portal Endpoint Mapping Per Note Type

```
Admission Note:
  → GET /api/patient/summary?mrn=X (full bundle)
  → GET /api/patient/notes?mrn=X&type=all&days=3 (ER notes)

Progress Note:
  → GET /api/recipes/progress-note?mrn=X (pre-bundled: vitals 1d, labs 2d, io 1d, meds active, prev note, dx)

Transfer / Acceptance Note:
  → GET /api/patient/summary?mrn=X
  → GET /api/patient/notes?mrn=X&days=7

Weekly Summary:
  → GET /api/patient/vitals?mrn=X&days=7
  → GET /api/patient/labs?mrn=X&days=7
  → GET /api/patient/io?mrn=X&days=7
  → GET /api/patient/meds?mrn=X&status=all
  → GET /api/patient/notes?mrn=X&days=7
  → GET /api/patient/dx?mrn=X
  → GET /api/patient/orders?mrn=X&days=7

Treatment Course:
  → GET /api/patient/summary?mrn=X
  → GET /api/patient/notes?mrn=X (all notes, no day limit)
  → GET /api/patient/meds?mrn=X&status=all
  → GET /api/patient/orders?mrn=X&status=all

Working Diagnosis:
  → GET /api/patient/dx?mrn=X
  → GET /api/patient/labs?mrn=X&days=7
  → GET /api/patient/notes?mrn=X&days=3&type=progress

Handover Sheet (multi-patient):
  → GET /api/list/inpatients?dr=DRCODE
  → For each patient: GET /api/patient/summary?mrn=X (condensed)
  (or use /api/recipes/handoff?dr=DRCODE for batch)
```

---

## Architecture

### Project Location

The slh-his monorepo uses npm workspaces with pattern `apps/*/web`. The existing `apps/inpatient/` has placeholder docs (dashboard/, notes/, nursing/). The new Next.js app goes in `apps/inpatient/web/` to match the workspace convention.

```
slh-his/apps/inpatient/
├── README.md                    # Existing (update)
├── dashboard/                   # Existing legacy (keep for now)
├── notes/                       # Existing legacy (keep for now)
├── nursing/                     # Existing legacy (keep for now)
└── web/                         # NEW: Next.js inpatient app
    ├── app/
    │   ├── layout.tsx               # Root layout (Tailwind, fonts)
    │   ├── page.tsx                 # Login page
    │   ├── (auth)/                  # Route group: all auth-required pages
    │   │   ├── layout.tsx           # Session check middleware
    │   │   ├── patients/
    │   │   │   └── page.tsx         # Patient list
    │   │   └── workspace/
    │   │       └── [mrn]/
    │   │           ├── page.tsx     # Patient workspace (note selection + generation)
    │   │           └── loading.tsx  # Loading skeleton
    │   └── api/
    │       ├── auth/
    │       │   ├── login/route.ts   # POST: proxy to agent-portal /auth/login
    │       │   └── logout/route.ts  # POST: revoke + destroy session
    │       ├── portal/
    │       │   └── [...path]/route.ts  # Catch-all proxy to agent-portal /api/*
    │       └── generate/route.ts    # LLM generation endpoint
    ├── lib/
    │   ├── session.ts               # iron-session config & helpers
    │   ├── portal-client.ts         # Server-side agent-portal HTTP client
    │   ├── cache.ts                 # LRU cache for clinical data
    │   ├── ai-client.ts             # Multi-model AI client (reuse from playground)
    │   ├── ai-models.ts             # Model definitions (reuse from playground)
    │   ├── types.ts                 # TypeScript types
    │   └── note-configs.ts          # Note type → data requirements mapping
    ├── prompts/
    │   ├── admission-note.ts
    │   ├── progress-note.ts
    │   ├── transfer-note.ts
    │   ├── weekly-summary.ts
    │   ├── treatment-course.ts
    │   ├── working-diagnosis.ts
    │   └── handover-sheet.ts
    ├── components/
    │   ├── LoginForm.tsx
    │   ├── PatientList.tsx
    │   ├── PatientBanner.tsx         # Top banner: name, bed, dx, admission date
    │   ├── NoteTypeSelector.tsx      # Left panel: note type cards
    │   ├── SourceDataPanel.tsx       # Collapsible source data sections
    │   ├── ManualInputPanel.tsx      # Editable fields (PE, assessment)
    │   ├── GenerateButton.tsx        # AI generation trigger
    │   ├── DraftEditor.tsx           # Editable generated note
    │   └── CopyExportBar.tsx         # Copy/Word export actions
    ├── package.json
    ├── next.config.ts
    ├── tailwind.config.ts
    └── tsconfig.json
```

### Tech Stack

| Concern | Choice | Why |
|---|---|---|
| Framework | Next.js 15 App Router | Same as playground, React 19 |
| Session | iron-session | Encrypted HttpOnly cookies, zero deps, works on LAN |
| Cache | lru-cache | Memory-safe, perfect for single-server |
| UI | Tailwind CSS + custom components | Same as playground, no heavy UI lib |
| AI Client | Multi-model (Claude, GPT, Gemini) | Reuse from playground |
| Export | docx library | Reuse from playground |
| Proxy | fetch to agent-portal | Server-side only, token never in browser |

### Session & Auth Flow

```
User → [Login Page]
  ↓ POST /api/auth/login { employee: "D4303", password: "xxx" }
  ↓ Server: fetch("http://localhost:3000/auth/login", { employee, password })
  ↓ Agent-portal: AS400 ODBC auth → returns { token: "slhp_...", employee, expires_at }
  ↓ Server: iron-session.save({ portalToken, employee, expiresAt })
  ↓ Browser: gets encrypted HttpOnly cookie "__Host-inpatient-session"
  ↓ Redirect to /patients

Browser → [Any /api/* request]
  ↓ Server: reads iron-session → extracts portalToken
  ↓ Server: fetch("http://localhost:3000/api/...", { Authorization: "Bearer slhp_..." })
  ↓ Response proxied back to browser (no token exposure)

User → [Logout]
  ↓ POST /api/auth/logout
  ↓ Server: fetch("http://localhost:3000/auth/logout", { Bearer token })
  ↓ Server: session.destroy()
  ↓ Redirect to /
```

### Data Flow for Note Generation

```
1. User selects patient → /workspace/[mrn]
2. Page loads: GET /api/portal/patient/summary?mrn=X → cached in LRU
3. User selects note type (e.g., "Progress Note")
4. UI shows relevant source data panels (vitals, labs, io, meds)
   - Auto-fetched based on note-configs.ts mapping
   - Displayed in collapsible sections
5. User optionally fills manual inputs (today's PE, assessment)
6. User clicks "Generate"
7. POST /api/generate { noteType: "progress-note", mrn, manualInputs }
8. Server:
   a. Fetches required data from agent-portal (based on noteType config)
   b. Builds prompt using prompts/progress-note.ts
   c. Calls AI via ai-client.ts
   d. Returns { result, inputTokens, outputTokens, costUSD, model }
9. Generated note appears in DraftEditor (editable)
10. User reviews, edits, copies to clipboard or exports to Word
```

---

## Implementation Phases

### Phase 1: Foundation (4 files)
**Goal**: App shell with login, session, and basic agent-portal connectivity.

1. `package.json` — dependencies: next, react, iron-session, lru-cache, tailwindcss
2. `lib/session.ts` — iron-session config, getSession helper
3. `app/page.tsx` — Login page (AS400 employee ID + password)
4. `app/api/auth/login/route.ts` — Proxy login to agent-portal, create session
5. `app/api/auth/logout/route.ts` — Revoke token, destroy session
6. `app/(auth)/layout.tsx` — Session check, redirect if not authenticated
7. `app/layout.tsx` — Root layout with Tailwind
8. `next.config.ts`, `tailwind.config.ts`, `tsconfig.json`

### Phase 2: Patient List & Workspace Shell (3 files)
**Goal**: Browse patients, navigate to workspace.

1. `lib/portal-client.ts` — Server-side agent-portal HTTP client with auth header injection
2. `lib/cache.ts` — LRU cache wrapper (TTL-based, key = mrn+endpoint)
3. `app/api/portal/[...path]/route.ts` — Catch-all proxy
4. `app/(auth)/patients/page.tsx` — Patient list (from /api/list/inpatients)
5. `components/PatientList.tsx` — Patient list component
6. `components/PatientBanner.tsx` — Patient info banner

### Phase 3: Note Generation Core (most critical)
**Goal**: Full generate flow for Progress Note (the most common type).

1. `lib/note-configs.ts` — Note type registry: which endpoints, which UI panels
2. `lib/types.ts` — TypeScript types for all data shapes
3. `prompts/progress-note.ts` — Progress note prompt (adapted from playground)
4. `lib/ai-client.ts` — Reuse from playground
5. `app/api/generate/route.ts` — Generation endpoint
6. `app/(auth)/workspace/[mrn]/page.tsx` — Patient workspace
7. `components/NoteTypeSelector.tsx` — Note type selection cards
8. `components/SourceDataPanel.tsx` — Collapsible data display
9. `components/ManualInputPanel.tsx` — Today's assessment inputs
10. `components/DraftEditor.tsx` — Editable generated draft
11. `components/CopyExportBar.tsx` — Copy/export actions

### Phase 4: All Note Types
**Goal**: Add remaining 6 note type prompts and their specific data configs.

1. `prompts/admission-note.ts`
2. `prompts/transfer-note.ts`
3. `prompts/weekly-summary.ts`
4. `prompts/treatment-course.ts`
5. `prompts/working-diagnosis.ts`
6. `prompts/handover-sheet.ts`
7. Update `lib/note-configs.ts` with all type mappings

### Phase 5: Polish & Hardening
**Goal**: Production readiness.

1. Session timeout handling (8hr from agent-portal key expiry)
2. Error states and loading skeletons
3. Word export integration
4. Mobile-responsive layout
5. CSP headers

---

## Files to Reuse (copy & adapt)

| From | File | Reuse |
|---|---|---|
| medical-ai-playground | `lib/ai-client.ts` | Multi-model AI caller (as-is) |
| medical-ai-playground | `lib/ai-models.ts` | Model definitions + cost calc |
| medical-ai-playground | `lib/docx-generator.ts` | Word export |
| medical-ai-playground | `lib/token-tracker.ts` | Token/cost tracking |
| medical-ai-playground | `prompts/progress-note.ts` | Prompt template pattern |
| medical-ai-playground | Login page UI pattern | Visual design only |

## Files to Create New

| File | Purpose |
|---|---|
| `lib/session.ts` | iron-session encrypted cookie management |
| `lib/portal-client.ts` | Agent-portal HTTP client with Bearer auth |
| `lib/cache.ts` | LRU cache for clinical data (5min TTL vitals, 15min labs) |
| `lib/note-configs.ts` | Registry mapping note types → required data → UI panels |
| All `prompts/*.ts` | 7 note type prompt builders |
| All `components/*.tsx` | UI components for workspace |

---

## Verification Plan

1. **Login flow**: Enter AS400 creds → verify session cookie set → verify /patients loads
2. **Patient list**: Verify patients load from agent-portal → click patient → workspace
3. **Data proxy**: Verify agent-portal data appears in workspace panels
4. **Note generation**: Select note type → generate → verify output quality
5. **Session security**: Verify portal token is NOT in browser (check DevTools cookies, localStorage, network tab)
6. **Cache**: Verify second load is faster (LRU cache hit)
7. **Logout**: Verify session destroyed, portal token revoked

---

## Key Architectural Decisions

1. **iron-session over custom HMAC**: The playground uses custom HMAC cookie signing. iron-session provides AES-256 encryption (not just signing), session data storage in cookie (no DB needed), and is the recommended pattern for Next.js App Router.

2. **Catch-all proxy over individual routes**: A single `app/api/portal/[...path]/route.ts` proxies all agent-portal requests. This avoids duplicating route definitions and lets agent-portal's own auth/permission system handle access control.

3. **Server-side prompt building only**: Prompts never leave the server. The frontend sends `{ noteType, mrn, manualInputs }` and gets back the generated text. This prevents prompt leakage and keeps PHI server-side.

4. **Note config registry pattern**: `lib/note-configs.ts` declares for each note type: which agent-portal endpoints to fetch, which manual input fields to show, and which source data panels to display. Adding a new note type = adding one config entry + one prompt file.
