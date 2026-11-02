---
name: till-variance-decision-notes-105
description: "Notes 105 Part A built 2026-10-12 — POST /till/sessions/decide-variance (APPROVE/RECOUNT) with the approver check and a till_approval row; Part B (the /till/approvals gate + inbox) built 2026-10-16 as notes 124 §5; approver role order and the SUPER ADMIN implicit grant are user decisions"
metadata:
  node_type: memory
  type: project
  originSessionId: e7d8bb81-5dab-4765-ad1d-61d5d53fdc96
  modified: 2026-10-12T06:23:25.202Z
---

Part A of notes 105 is live (2026-10-12): `POST /till/sessions/decide-variance` on a
PENDING_APPROVAL session, menu 273 OVERRIDE. `TillApprovalService.assertApprover` judges the
caller (self → `TILL_APPROVER_SELF`; a `till_approval_authority` grant at the rule's level
covering Σ|counted − expected| → `_NO_GRANT` / `_LIMIT`; channel COUNTER = `approverPin` against
`usr_pin_hash`, REMOTE = `taa_can_remote`, EITHER = either → `_PIN`, all 403), then `record()`
writes `till_approval` (APPROVED, or REJECTED for a RECOUNT) — the first writer of that table.
APPROVE sets the PENDING variances' treatment and sends the session back to COUNTING so `/close`
posts the TVar (RECOVER → `emp_loan_ledger_id` of the cashier, SUSPENSE → TENDER_SUSPENSE);
RECOUNT bumps `tss_recounts_granted` (migration 20261012120000) and the next final count takes the
final flag (`ux_tct_final`) and soft-deletes the old variances (`ux_tvr_live`).
Notes 112 (2026-10-14): RECOUNT also clears `tss_close_count_id` / counted_on/_by / witness and
zeroes cash/noncash counted, and `/close` refuses (409 TILL_NOT_COUNTED, `tenderTypeIds`) while a
non-SLIPS CLOSE variance is PENDING (`tvr_slip_count IS NULL` = drawer row). Before that fix two
sessions closed undecided: VIJI PC-261012-02 (cash +1,900, repair asked for, not yet run as of
2026-10-14) and ZT1-261012-01 (harness, cash −177).

**Why:** the user decided 2026-10-12 that a SUPER ADMIN (`usr_type`) is an implicit HO_FINANCE
grant on every event and branch with no ceiling, that ADMIN and everyone else need a 276 grant,
and that self and channel rules still bind a super admin. The approver level order is TILL_DESIGN.md
§5.1's — SUPERVISOR < CASH_OFFICE < STORE_MANAGER < AREA_MANAGER < HO_FINANCE — which the Qt 276
screen previews; `TILL_APPROVER_ROLES` was reordered to match (it had STORE_MANAGER before
CASH_OFFICE). Two-person rules are refused as "Part B".

**How to apply:** Part B (`/till/approvals/request · decide · get · cancel`, the inbox, expiry,
two-person, FORCE_CLOSE / FLOAT_MISMATCH / EXPENSE enforcement) should grow out of
`assertApprover` + `record`, and A1 then becomes a thin wrapper over `decide`. A wrong PIN is
journalled `PIN_FAIL` outside the transaction (the check runs before it). The e2e suite
`test/till-variance-decide-http.e2e-spec.ts` sets a PIN on user prathap and clears it after.
Related: [[till-management-schema]], [[user-menu-transaction-rights]].

Part B built 2026-10-16 — see [[till-notes-124-open-list]]; decide-variance stays its own route (the inbox names it).
