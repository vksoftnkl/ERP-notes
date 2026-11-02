---
name: erp-qt-c77f3f1-port
description: "Port of Qt NexERP a449b8c..c77f3f1 (2026-10-15) — till notes 113/114/117: day close/exception/reopen, Approve Cash, force close/reopen/X read, session Z/X + day Z print with branchId, Open Till 15 s wait poll, app-wide 12-hour clock"
metadata:
  node_type: memory
  type: project
  originSessionId: 2fa1028f-7f61-4fb2-8b1d-66e43e34cbd5
  modified: 2026-10-15T05:23:59.981Z
---

Ported 2026-10-15 by four parallel subagents (business-day / sessions / session folders + a 12-hour sweep)
on a shared layer written first. Follows [[erp-qt-6f15977-port]] and the a449b8c port (commit 7b0546a).

- **Business Day** (`features/sales/till/business-day`): `GET /till/days/close-check` drives checklist,
  verdict and buttons (F10 Close day, Close with exception dialog, Reopen day dialog, Print day X / store Z =
  purpose `TILL_DAY_Z`). Qt bug NOT copied: Qt reads `data.tbdZNo` after `/days/close`, but the server returns
  `zNo` at the top level (`day.tbdZNo` inside) — web reads `zNo`.
- **Cockpit** (`features/sales/till/sessions`): Approve Cash (`/sessions/approve-cash`), Force close, Reopen,
  X read (`/sessions/x-report`), Print Z, bag decide (decide-variance with `tcmId` for a RECEIVE-stage
  variance), Re-tender of the picked Bills-tab row via testbill's dialog + `buildRetenderBody` with a stand-in
  draft. Sessions list: variance filter `TO_APPROVE`, `cash_receipt` pill column.
- **Cash Office (menu 286, notes 113)** was retired by notes 114 in Qt itself — deliberately not ported.
- Shared: `features/sales/till/shared/session-badges.ts` (one pill palette for grids 144/150);
  `till-session-api.ts` gained `openCheckQuiet`/`getQuiet` (QUERY reads for timers, so a poll doesn't fire
  the data-freshness bus — see [[erp-data-freshness-bus]]).
- **Print `branchId`**: optional on `DocumentPrintTarget` → preview/print/log; only till prints send it
  (a session/day is printed away from its counter). `/resolve` still gets no branch.
- **12-hour clock**: `lib/time-format.ts` (`clock12Of`, `dateTime12Of`, `clock12FromHhmm`…), display only.
  Physical-stock freeze field now parses AM/PM back (else PM times shift 12 h on blur). TxnHistory
  "When" is formatted by grid 126's SQL — dev already says `HH12:MI:SS AM` (2026-10-14); live via the sync below.
- Live sync: `docs/live-sync/2026-10-15-till-grids.sql` (client repo) upserts 14 grids (126, 133–136, 144–152) +
  5 dropdowns (64–68) BY NAME, dev id kept when free; guarded on the notes 113/117 columns (grid 144/150 SQL
  reads them) so it must run AFTER the server's till migrations are deployed. Dry-run verified on a throwaway
  copy (guard, inserts, id collision, idempotent, content-identical to dev). Prod reads/writes are blocked by
  the auto-mode classifier, so the USER runs it. Print templates for TILL_* purposes are still dev-only.
- Grid/dropdown/ui-table directories now read EVERY page (`store/api/run-all-pages.ts`): 109 Desktop grids
  in dev, so 149–152 had only ever resolved by fallback id.
- Peers staged this port's files mid-task AGAIN; unstaged only ours ([[erp-workspace-layout]]).
- Not browser-driven; success paths of close/reopen/approve were unit-tested only (never posted on dev DB).
