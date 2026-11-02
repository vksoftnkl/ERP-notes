---
name: till-notes-124-open-list
description: "Notes 124 (Qt's till open list, 2026-10-16) — approval gate built (/till/approvals/*, till.approval_gate WARN|REFUSE), decide-variance on CLOSED sessions, day safe belowZero/awaiting, expense refno pre-check, surcharge inside the tender; answers on items 2/3/4"
metadata:
  node_type: memory
  type: project
  originSessionId: d006fa32-cd91-4155-862e-f20e41c74d33
  modified: 2026-10-16T05:09:09.514Z
---

"Notes 124" is TWO documents: the receipt/payment/cheque wording REV 1 (see [[error-codes-notes-119]])
and Qt's "what is still open on the till server" checklist (2026-10-16). This is the second.

Built 2026-10-16 (migration `20261016100000_till_approval_gate_notes_124`, applied on the dev DB):
- §1 user's call: an approver decides, no hand fix. `sessions/decide-variance` on a CLOSED session
  with a drawer (non-SLIPS) CLOSE gap PENDING → APPROVE only, TVar posted at once dated the session's
  business date (`decideAfterClose`). VIJI PC-261012-02 (+1,900 cash; its UPI +50 is a SLIPS gap,
  not decided here) and ZT1-261012-01 (−177) still wait for an approver in Qt.
- §3 user's call: print + flag. days/get `safe.belowZero/awaitingOpening/awaiting`, print
  `safe_below_zero/safe_awaiting_opening/safe_awaiting` = TILL_CASH_AWAITING ledger as of date.
  The real 13-10 −1,426 cause: ZTS safe ledger (01a119d0…) has no opening balance and paid ~6,756
  of ExpV tests (10-09/10-12) — not unapproved hand-overs (awaiting ledger starts 10-14).
- §5 user's call: designed from the schema (TILL_DESIGN §S7 not on disk). See till README "The
  approval gate". Gate = `TillApprovalService.assessGate/settleGate`; wired EXPENSE, CASH_PAYMENT,
  FLOAT_MISMATCH (approval names TILL_COUNTER before a session exists). Setting `till.approval_gate`
  WARN (shipped) / REFUSE, DEVICE max scope. WARN files PENDING till_approval rows → e2e suites that
  post money docs or mismatched opens in a session must delete `till_approval` by session/doc.
- §4 answer: the 254 `bills/create` 500s on 10-15 were `ck_slt_prices` (ZT-ITEM-B negative avg wot),
  fixed 13:01 by [[stock-negative-average-wot]] — that fix was still UNCOMMITTED on 10-16.
- §2 answer: no backup exists (no dumps since 10-08, archive_mode off) — events can't be restored.
- §6 was already built (notes 121 §4). §7: pre-write clash check; the catch also missed partition
  index names. §8: `selfRefusalOf(event)`, code unchanged (catalogue variant `event=FORCE_CLOSE`).
- §9 measured: 754 uncoded sites (876 TS errors − 122 TS2344) when `code` is required; expense's 7
  coded with VCH_*/EXPENSE_NO_LINES; the rest left for the user to scope.

**Why:** the Qt team retests each item live once told it is deployed.
**How to apply:** Part B gaps (NONCASH after slip check on CLOSED sessions, FLOAT_MISMATCH re-treat,
VOID_BILL server enforcement) are listed in the till README "Not in phase 1". Related:
[[till-variance-decision-notes-105]], [[till-safe-notes-118]], [[till-notes-122]].
