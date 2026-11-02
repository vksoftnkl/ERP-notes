---
name: notes-127-status
description: "Notes 127 (Qt's leftovers from notes 124, 2026-10-16) — all of 1–6 built the same day across three sessions; `code` now REQUIRED on every shared error detail; §1 hide-SUPER-ADMIN-from-list not done (grid 62)"
metadata:
  node_type: memory
  type: project
  originSessionId: 25542c84-46e2-48bb-8a14-e562612f5fcc
  modified: 2026-10-16T10:24:04.961Z
---

Notes 127 = what was still open from Qt's notes 124 till list. Built 2026-10-16 (sessions d006fa32,
0ccc8f3c; checked + §4 spec added by 25542c84), all uncommitted when checked at 15:50:
- §1 / §2: were already notes 125 ([[user-role-super-admin-notes-125]]). Optional "hide SUPER ADMIN
  users from the list" NOT done: the list is grid 62 and the grid runner has no trusted caller.
- §3: `approveCash` lets the session's own cashier approve when the CASH_VARIANCE rule has
  `tar_allow_self` — always with PIN (only single_operator may skip it). Journal `selfApproval`
  'SINGLE_OPERATOR' | 'RULE'; session `receivedBySelf`; Z prints `received_by_self`.
- §4: was already built (receipt-lines.ts, perc/(100+perc)); unit spec
  `receipt-lines.surcharge.spec.ts`.
- §5: `code: string` REQUIRED on `ModuleErrorDetail`, `ModuleApiErrorDetail` and `throwNotFound`;
  tsc clean. A plain Nest 4xx (no house body) gets `HTTP_<status>` from AllExceptionsFilter.
- §6: force-close by the session's own cashier → `TILL_FORCE_CLOSE_SELF`, its own sentence.
- §7 answers: no backup exists; −1,426 → −1,346 is the ZT safe with no opening balance (+80 = 14-10
  net), print-and-flag already built; 10-15 500s were `ck_slt_prices` ([[stock-negative-average-wot]]).

Notes 129 §1/§2 (Qt's recheck) only saw stale Swagger: approve-cash description + ApproveTillCashDto PIN text
and the receipt tdSurchargeAmt docs were updated 10-16; a gap decided in approve-cash now files tap_self_allowed
= true when self-approved.

**Why:** Qt rechecks each item live once Viji is told it is deployed (:3011).
**How to apply:** a new refusal must carry a code or it won't compile. Related: [[till-notes-124-open-list]],
[[error-codes-notes-119]].
