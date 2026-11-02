---
name: erp-qt-3b5069d-port
description: "Port of Qt NexERP 1d3e203..3b5069d (2026-10-19) — Cash Position 289 + HO review (grids 170–174), Till Reports tabs 5–7 (175–180), TILL_MOVEMENT/TILL_REMIT prints, Open Till held session, ?branch= drills, Textile Tags screen built before its server routes exist"
metadata:
  node_type: memory
  type: project
  originSessionId: f90cbf20-649c-4d23-8b1b-48f225b32dd7
  modified: 2026-10-19T10:31:26.020Z
---

Ported 2026-10-19 by four parallel subagents on a shared layer written first. Follows [[erp-qt-1d3e203-port]].
Not committed. A peer staged all 66 port files mid-task (index == worktree, only our hunks); I unstaged exactly
those paths ([[erp-workspace-layout]]).

- **Cash Position** (menu 289, notes 134) at `/sales/till/cash-position`, `features/sales/till/cash-position`.
  Grids 170–174 (`tillCashPosition*`), every token QUOTED; 174 needs a branch. Cash in tills = Σ
  `/till/sessions/expected` cash tender per live session (today only; a 403 counts as hidden). HO review dialog
  posts `/till/reviews/create` (client `reviewCreate`/`reviewList`). Menu 289 verbs are VIEW·CREATE·EXPORT — no
  PRINT, so Print may be disabled there (Qt gates it the same way).
- **Drill contract** (Qt `openOn`/`setBranch`/`setScope`): `/sales/till/business-day?branch=&tbdId=&accYear=`,
  `/sales/till/sessions?branch=`, `/sales/till/reports?branch=&branchName=&from=&to=`. Read via `useSearchParams`,
  NEVER `window.location` in a lazy initial state: on a forward client navigation Next pushes the URL after the
  first render, so `window.location` is still the page being left (both pages had it; fixed).
- **Till Reports** tabs `performance`/`ledger`/`tender` (grids 175–180). On ledger/tender the MAIN row names the
  session (Open cockpit / Ctrl+H fall back to it, Qt `currentSessionRow`).
- **Prints (notes 135)**: `PURPOSE_CODE.TILL_MOVEMENT` (Cockpit Print slip, Cash Movement post→"Print the slip
  now?") and `TILL_REMIT` (Cash Office Print memo, post remittance→question). docId = tcm_id/md_id.
- **Open Till**: a held session (`deviceSession` ?? `userSessionElsewhere`) greys the summary to "—";
  `counterDisplay` shows "<code>  (in use by <name>)" first.
- **Textile Tags** at `/stock/textile-tags` (`features/stocks/textile-tags`, pure reducer + tests) — no menu row and
  NO `/textile-tags/*` server routes yet (notes 136 steps B-1/B-2); reserve/suggest answers were faked in tests and
  a CDP smoke run. ui_table 53 is dev-only; list dropdown ids are 0 (names resolve at reserve via the save route).
  The dev DB keeps a soft-deleted copy of every ui_table 53 column; the route filters it out.
- Live sync: `docs/live-sync/2026-10-19-till-cash-position-grids.sql` (grids 170–180 by NAME; guarded on menu 289 +
  till_session_review). Dry-run on dev in BEGIN…ROLLBACK, sequence reset after. USER runs it on live
  ([[erp-live-config-drift]]).
