# Notes 105: no way to approve a till variance; the approval gate is still missing (2026-10-12)

**STATUS:** Part A DONE + verified live 2026-10-12: decide-variance through the Qt Session Cockpit popup. Self → TILL_APPROVER_SELF, no grant → TILL_APPROVER_NO_GRANT, wrong PIN → TILL_APPROVER_PIN, grant + PIN + RECOUNT → ZT1-261012-01 back to COUNTING. APPROVE (posting via /close) not yet run live. Note: an approver also needs menu 273 Override (TILL_RIGHT_OVERRIDE). Part B open.

**Order:** Part A now, because a live session is stuck on it. Part B after A.

## What happens

The user asked: *"why admin / super admin can't approve cash variance"*. The answer is that **nobody can**. There is no route to approve a variance.

- `/till/sessions/count` moves a session that is still out of tolerance after the last recount to **PENDING_APPROVAL**. Its variances are left with `tvr_treatment = 'PENDING'`.
- `/till/sessions/close` on that session answers **428 `TILL_APPROVAL_REQUIRED`** (`till-session.service.ts` ~951). Its own comment says *"the approval gate is phase 3; until then a supervisor decides it there"*. But there is no "there": TILL_DESIGN.md lists `POST /till/approvals/request · decide · get · cancel`, and none of these routes exist.
- `accounts.till_approval` has **0 rows**. `accounts.till_approval_authority` (the grants made in Approval Setup, menu 276) has **0 rows**. The rules exist: 23 shipped rows, e.g. `CASH_VARIANCE OVER_AMOUNT 10.00 → SUPERVISOR, allow_self false, blocks_till true`.

**Stuck right now** (Acme / Coimbatore, test counter ZT1):

| session | status | variance |
|---|---|---|
| ZT1-261012-01 | PENDING_APPROVAL | CLOSE stage, cash: expected 2,177.00, counted 2,000.00, **−177.00** (tolerance 10), treatment PENDING |

It cannot close, it cannot be approved, and it will also block Day Close for the branch.

## Part A: the minimum that unblocks (please do first)

### A1. `POST /till/sessions/decide-variance`

Body: the session key (`companyId, branchId, accYear, tssId`), plus:

```json
{
  "decision": "APPROVE | RECOUNT",
  "treatment": "EXPENSE | RECOVER | SUSPENSE",    // APPROVE only
  "reasonId": "<till_reason, category CASH_VARIANCE / NONCASH_VARIANCE>",  // APPROVE only
  "note": "free text, required",
  "approverPin": "...."                            // when the rule's channel is COUNTER (see A2)
}
```

What it does, in one transaction:
- **Only on a PENDING_APPROVAL session**; anything else gets 409.
- **APPROVE:**
  - On every `PENDING` variance of the session: set `tvr_treatment` (the chosen one), `tvr_reason_id`, `tvr_decided_by`, `tvr_decided_on`, and `tvr_approval_id` (pointing at the A3 row).
  - For `RECOVER`, `tvr_recovery_ledger_id` = the cashier's staff ledger (notes 95). If the cashier has none, refuse with `field: treatment`.
  - Session goes back to **COUNTING**, so the existing `/close` runs unchanged and posts the TVar with the chosen treatment.
  - `tss_variance_status` becomes `ACCEPTED` / `RECOVER` / `INVESTIGATE` (SUSPENSE).
- **RECOUNT:** the session goes back to COUNTING with one more count attempt allowed. The variances stay as they are, and the next `/count` replaces them as it does today.
- Write a `till_event` row (`VARIANCE_DECIDED`), as the other session actions do.

### A2. Who may decide (this is the user's actual question)

Apply the rule (`TillApprovalService.ruleFor`: branch > company > shipped) to the event (CASH_VARIANCE if the cash variance is among the pending ones, else NONCASH_VARIANCE) and to the amount (the sum of |counted − expected|, as the 428 already computes). The caller passes if **all** of these hold:

1. **Not the session's cashier** (`tar_allow_self = false`). This is the blind rule `assertApprover` already enforces.
2. Has a **till_approval_authority grant** for this company (and branch, or every branch) that is active today: event = this event or ALL, `taa_role` ≥ the rule's `tar_min_role`, `taa_max_amount` ≥ the amount.
   - Level order, as the 276 screen previews it: SUPERVISOR < CASH_OFFICE < STORE_MANAGER < AREA_MANAGER < HO_FINANCE. Please confirm, or tell us the order you use.
3. Channel: COUNTER means the approver's till PIN is checked (`approverPin`). REMOTE means `taa_can_remote`. EITHER accepts either.

**Admin / super admin: decided by the user, 2026-10-12.** A **super admin counts as an implicit HO_FINANCE grant** on every event, every branch, with no amount limit. The rules still apply to them: never on their own session (`allow_self`), and the channel. **Admin and everyone else need a 276 grant.**

Refusals, with `field` set, as the till module does elsewhere:
- `TILL_APPROVER_SELF`
- `TILL_APPROVER_NO_GRANT` (name the event and the role needed)
- `TILL_APPROVER_LIMIT` (name the max)
- `TILL_APPROVER_PIN`

### A3. Record it in `till_approval`

One row per decision. The table already has every column needed:
- `tap_event_code`, `tap_session_id`, `tap_amount`
- `tap_rule_id` + `tap_rule_snapshot`
- `tap_required_role`, `tap_requested_by` (the cashier, the moment it went PENDING_APPROVAL)
- `tap_status` = APPROVED / REJECTED
- `tap_decided_by`, `tap_decided_on`, `tap_decided_device_id`, `tap_decision_note`

That way Part B can start from the same table.

## Part B: the gate itself (after A)

As in TILL_DESIGN.md §S7: `POST /till/approvals/request · decide · get · cancel` plus a pending list for an inbox. Then the other events that today are "reported only" or disabled can use it:
- FORCE_CLOSE (the Open Till button is disabled today)
- FLOAT_MISMATCH at open
- NONCASH_VARIANCE after a slip check
- paid-out and expense over limit (EXPENSE 500 is reported, not enforced)
- SAFE_VARIANCE (two-person)
- the parked void bill / void line

A1 can then become a thin wrapper over `decide` for CASH_VARIANCE / NONCASH_VARIANCE.

## Client side (we do this once A is live)

- **Session Cockpit (273 → Enter):** the Approve button is disabled today with "comes with the approval gate". It becomes a popup: treatment, reason, note, PIN when the rule wants one, and Approve / Recount. The 428's details (event, amount, required role, channel) fill its header.
- **Business Day (274):** the "%n session(s) wait for an approver" line gets a jump to the cockpit.
- **Approval Setup (276):** no change. Its grants are what A2 reads.
