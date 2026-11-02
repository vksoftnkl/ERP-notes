---
name: erp-qt-1d3e203-port
description: "Port of Qt NexERP 425fa80..1d3e203 (2026-10-17) — Till Reports menu 288 (grids 159–169, dev-only → live-sync), Receipt/Payment Voucher money line = tender list, Open Till Force close; server grid 166/167 tautology binds; dry-run sequence trap"
metadata:
  node_type: memory
  type: project
  originSessionId: efd50c0a-e938-4cce-9b50-123aba8a2507
  modified: 2026-10-17T08:21:11.610Z
---

Ported 2026-10-17 by two parallel subagents (reports / vouchers) on a shared layer written first. Follows [[erp-qt-425fa80-port]].
Not committed by me. At the end all 25 port files were staged by someone else (only my hunks, nothing else staged) — left as is, told the user ([[erp-workspace-layout]]).

- **Till Reports** (menu 288, server migration 20261017110000) at `/sales/till/reports`, `features/sales/till/reports`
  (`till-reports.ts` domain + `use-till-reports.ts` + `report-table.tsx`). Registry keys `tillRpt*` (159–169). Every token is
  QUOTED — always sent, `""` = all ([[erp-configured-grid-registry]]). Cockpit is a route: the view is saved to
  sessionStorage `till-reports:back:<company>` and restored once on Back. Ctrl+H = `sessions/journal-dialog.tsx`
  (grid 145, what Qt's `tillSessions().history` is), NOT `TxnHistoryDialog` (grid 126). Print via
  `features/printing/print-delivery.ts#sendToPrinter`; Export = CSV download (BOM + CRLF).
- **Grids 159–169 exist only in the dev DB** (Qt's registration script, not a server seed) →
  `docs/live-sync/2026-10-17-till-report-grids.sql`; USER runs it on live ([[erp-live-config-drift]]).
  Guarded on tcm_remit_status / tsf_insured_limit / till_approval(+_authority).
- SERVER: grid 166 binds `iapprover_id` only as `('iapprover_id' = 'iapprover_id')` → the Approver filter never
  narrows the not-decided band; 167 binds counter/operator the same way. Grid 157 has no NULLIF on `ibranch_id`
  ("Every branch" → pick a branch, as Qt).
- **Voucher money line** (RcpV Dr / PmtV Cr): `features/accounts/vouchers/domain/tender-menu.ts` generalises the
  Contra picker (`isMoneyRow`, `tenderMenu`, `tenderLabel`, `tenderNeedsDetails`, `applyMoneyTender`,
  `settleMoneyLine`). Instrument popup runs as a queue (Cancel stops; money line stays). Qt's new
  typedDifference / ensureTrailingLine-after-instrument were already true on the web (`handDifference`, `setInstrument`).
  Validate accepted the payload live for both types.
- **Open Till Force close** (another cashier's session on this device): `forceCloseRequestOf` renames the
  open-check session to cockpit keys; ForceCloseDialog now takes `ForceCloseRequest` (`SessionFactsSource`).
- **Dry-run trap:** running a live-sync script on dev inside BEGIN…ROLLBACK still advances sequences
  (`nextval`/`setval` are not transactional) — reset with `setval(seq, max(id))` afterwards.
