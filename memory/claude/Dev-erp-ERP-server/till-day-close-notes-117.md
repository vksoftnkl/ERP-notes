---
name: till-day-close-notes-117
description: "Notes 117 built 2026-10-14 — days/close-check · close · reopen (TillDayCloseService), sessions/force-close · reopen · x-report, print purposes TILL_SESSION_Z/X + TILL_DAY_Z with till.* providers; endBilling/count/close split into *In bodies; who is requester vs approver per exception"
metadata:
  node_type: memory
  type: project
  originSessionId: c78c5194-a12d-48dd-a125-6bfccf139357
  modified: 2026-10-14T14:10:43.279Z
---

Migrations `20261014180000_till_day_close_notes_117` (reason category DAY_CLOSE + reasons,
FORCE_CLOSE SHIFT_OVER / DAY_CLOSE, menu 274 EDIT granted to CREATE holders) and
`20261014181000_till_print_purposes_notes_117` (print_purpose rows, src_module ACCOUNTS because
ck_ppo/ck_plg_src_module have no TILL).

Shape decisions (mine; the notes left them open):
- TillSessionService `endBilling/count/close` = lock + `endBillingIn/countIn/closeIn(tx, session, caller)`;
  `countIn(..., {forced})` = final at once, no recount; `closeIn` keeps `tss_z_no` on a re-close.
- Requester vs approver (assertApprover): force close & session reopen → caller approves, cashier
  requests; day reopen → caller approves, the day's closer (`tbd_closed_by`) requests; day-close
  exception → caller requests, `exception.approverId` (default caller → APPROVER_SELF) approves with a
  PIN typed at the desk (channel forced COUNTER when approver ≠ caller).
- Day-close exception over SESSIONS_LIVE needs `exception.counts[]` (one per live session) and
  force-closes inside the close tx with `receiveHandOver: true` (TRcv on the supervisor's count) —
  otherwise CASH_TO_APPROVE would block the same close. A live session with AWAITING drops → refused.
- X number = count of X_REPORT events whose payload has `xNo` (devices may still send X_REPORT).
- Print counts: PrintLogService `PRINT_COUNTERS` bumps tss/tbd_print_count for TILL_SESSION_Z /
  TILL_DAY_Z — the first writer of any `*_print_count`.
- A closed day now refuses sessions/open with TILL_DAY_NOT_OPEN (was SALES_DAY_CLOSED).

**Why:** Qt Business Day / Cockpit buttons were greyed out waiting for these routes (notes 117).

**How to apply:** day-close tests must never run on the live branch — see
[[e2e-suites-run-on-live-branch]] (temporary branch fixture). Safe figures / SAFE_NOT_COUNTED wait for
notes 118 — built 2026-10-15, see [[till-safe-notes-118]] (SAFE_NOT_COUNTED is in `CLOSE_EXCEPTION_CODES`).

Related: [[till-handover-receipt-notes-113]], [[till-variance-decision-notes-105]], [[till-management-schema]].
