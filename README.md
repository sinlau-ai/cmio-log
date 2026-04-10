# CMIO Log

Daily work log for Li-yang Chen (陳禮揚), CMIO / AI Team, Tainan Sin-lau Hospital.

Tracks AI-powered clinical workflow development across multiple repos at `D:\CC`.

---

## 2026-04-10 (Thu)

### Inpatient App

Active WIP — 7 files changed (141 insertions), not yet committed:

- New `GenerateModal.tsx` — UI for triggering AI note generation
- `TPRFlowsheet.tsx` enhancements (+59 lines) — expanded vital signs display
- `LabFlowsheet.tsx` — lab data rendering refinements
- `SourceDataPanel.tsx`, `Dialog.tsx` — UI component updates
- `prompts/progress-note.ts` — prompt tuning for daily progress notes

### CMIO Log

- Repo initialized, daily log created with full 2-week history

---

## 2026-04-09 (Wed)

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

## 2026-04-08 (Tue)

### Inpatient App

- feat(handoff-v2): rich DOCX with TPR charts + PDF via Playwright
- feat(handoff-v2): data-driven handoff, UI polish, TPR fixes, navbar rename

---

## 2026-04-07 (Mon)

### Inpatient App — Phase 5 push (7 commits)

- feat(phase5-6): Polish, Word export, DataSourceSelector, TPR enhancements
- feat(phase4): all note types + AI consult + server scripts + PRD update
- feat: DD checklist, patient notes panel, draft storage; expand GenerateModal
- feat(ui): improve contrast, nav font size, AI consult zh-TW text
- fix: add retry with backoff for Anthropic 529 overloaded errors
- Merge feat/phase4-all-note-types into main

### Agent Portal — zh-TW + new endpoints (6 commits)

- Add tubes endpoint, O2 vitals fields, nursing NANDA, fix IO/dx/summary
- Dashboard zh-TW translation + improved chart/table readability
- Dashboard: fix charts, add zh-TW endpoint descriptions in popup
- API docs full zh-TW translation; add admission_source to PatientInfo type
- fix: QA-driven stability improvements across portal
- docs: update api-docs.html; add start-portal.sh launcher

### Portable CC

- chore: update skills submodule, CLAUDE.md, settings, cc-share.bat

---

## 2026-04-06 (Sun) — Biggest day (18 commits)

### Inpatient App — Built from scratch, phases 1-3 (9 commits)

- feat(phase1+2): foundation + patient list & workspace shell
- docs: Phase 2.5 PRD, prompts for portal encoding fix, orders parsing
- feat(phase2.5): role-based routing, sidebar, zh-TW UI, TPR flowsheet
- refactor: simplify Phase 1+2 code after review
- feat(phase3): AI note generation core — Progress Note with structured data prompts
- feat(phase3): lab flowsheet body-fluid separation, exams panel, RCW sidebar filter
- Merge branches into main

### Agent Portal — Phase 6 + new endpoints (6 commits)

- Phase 6: Fix inpatient labs (LATSDL2) + 5 new CLI commands + recipe enrichment
- feat: PFUDD pharmacy source-of-truth for active orders/meds filtering
- Add patient exams endpoint — imaging/exam reports with PACS links
- Improve lab response: add test_name, group classification, collected_at ISO date
- Add root redirect, /api-docs route, API Docs link in dashboard header
- Add test accounts, dashboard view-only access, expired key cleanup, API docs

### Portable CC (3 commits)

- feat: add never-kill-node safety rule to CLAUDE.md
- chore: update skills submodule (cc-pull/cc-push add inpatient repo)

---

## 2026-04-05 (Sat)

### Agent Portal

- Direct orders/meds via NSARDL2+M02, fix Codex-flagged regressions (P2a/P2b/P3)

### SLH-HIS

- feat(handoff-cli): refactor docx generator, update query-mrn and types

### Portable CC (3 commits)

- chore: bump skills submodule (cc-push/pull GitHub-only, new cc-share/cc-sync)
- chore: update CLAUDE.md, settings, launch.ps1; bump skills submodule (portal-qa + slh-his)

---

## 2026-04-04 (Fri) — Agent Portal sprint (24 commits)

### Agent Portal — From zero to production (22 commits)

- Initial commit: project scaffold + auth POC + implementation plan
- Add comprehensive PRD for Agent Portal CLI
- PRD v2: integrate Codex review (8 fixes) + HIS schema research (6 answers)
- Phase 4: recipes, prompt formatter, HTTP server (161 tests pass)
- Phase 5: Direct HIS access, migration system, production polish
- Add credential-based login + audit IP tracking + on-demand audit dashboard
- Dashboard: auto-shutdown, admin management, A+/A- font controls, request detail popup
- Fix Big5 encoding, dashboard login cookie, .env loading
- Migrate all SQL-queryable endpoints off QueryMRNService proxy to direct HIS DB queries
- Fix parser gaps vs handoff-cli and wire HTTP routes to direct HIS queries

### Portable CC (3 commits)

- feat: add codex launcher, HIS DB write guard hook, npm SSL config
- chore: update skills submodule — agent-portal skill v2

---

## 2026-04-03 (Thu)

### Portable CC (5 commits)

- feat: add LAN sync scripts (cc-share.bat + cc-pull-lan.bat)
- feat: add SMB share/unshare commands to cc-ops
- fix: bootstrap copies fallback launchers from seed before git clone
- chore: update skills submodule — slh-his moved to directory/SKILL.md for invocability

---

## 2026-04-01 (Tue)

### Portable CC (4 commits)

- feat: update ventilator-catastrophic-illness skill with IDS rules + QueryMRN
- refactor: tidy root — move utility scripts to scripts/, add cc-ops menu
- chore: add codex to portable runtime (PATH, pack.ps1, skills submodule)
- feat: add Claude Code auto-update, remote control, and daily update check

---

## 2026-03-30 (Sun)

### Portable CC — Bootstrap & seed (7 commits)

- feat: add minimal bootstrap seed for new-PC setup
- feat: improve bootstrap drive search; add cc-seed skill
- feat: add session history backup/restore to cc-push/cc-pull skills
- feat: add standalone session backup script (backup-sessions.bat)

### SLH-HIS — Dashboard + platform (7 commits)

- feat: unified sinlau-ai platform with dashboard enhancements
- fix: dashboard PACS auth, glucose fallback, med sorting, TPR auto-scroll
- fix: start.bat auto-detects and launches QueryMRNService
- fix: layout state reset, ICU vitals, allergy dedup
- feat: OrderHistoryApp integration — reverse engineer + dashboard enhancements

---

## 2026-03-29 (Sat)

### Portable CC

- feat: rename cc-save skill to cc-push; add cc-pull skill
- feat: switch runtime archive to zip; add cc-save skill + headless script

### SLH-HIS

- feat: QueryMRNService HTTP client, conflict resolution, CRLF docs
- merge: handoff-cli enhancements (active orders, RCW filter, PDF, local Chrome)

---

## 2026-03-28 (Fri)

### SLH-HIS (3 commits)

- fix: use local Chrome (D:\CC\chrome-win64) for PDF, skip Playwright install
- fix: restore missing parseStructuredVitals and VitalReading type
- fix: active order bug, RCW filter, 24hr time, drug lists, PDF output

---

## Summary (2026-03-28 ~ 04-10)

| Repo | Commits | Active Days | Theme |
|------|---------|-------------|-------|
| **Portable CC** | 28 | 10 | Skills submodule, bootstrap/seed, LAN sync, Codex, cc-ops |
| **Agent Portal** | 33 | 5 | Built from scratch — PRD, phases 4-6, dashboard, direct HIS DB, API docs |
| **Inpatient App** | 17 | 4 | Built from scratch — phases 1-6, AI notes, handoff DOCX/PDF, TPR charts |
| **SLH-HIS** | 14 | 4 | Dashboard fixes, QueryMRNService, handoff-cli, PDF export |
| **CMIO Log** | 1 | 1 | Initial commit |

**Highlights:**
- Two new projects (Agent Portal, Inpatient App) went from zero to production in ~5 days each
- Agent Portal: 33 commits in 5 days — full HIS data API with auth, dashboard, 161+ tests
- Inpatient App: 17 commits in 4 days — Next.js clinical note generator with AI integration
- Portable CC bootstrap system matured: thin-seed, cc-dehydrate/hydrate, LAN sync
