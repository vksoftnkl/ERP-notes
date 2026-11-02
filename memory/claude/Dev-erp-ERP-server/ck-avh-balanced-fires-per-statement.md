---
name: ck-avh-balanced-fires-per-statement
description: Removing ALL legs of a voucher in one UPDATE passes ck_avh_balanced (0=0); only a one-sided subset is refused
metadata: 
  node_type: memory
  type: project
  originSessionId: 8806e8bc-d8c9-4a60-b458-3ba8bf781fae
  modified: 2026-09-17T14:03:02.665Z
---

`tr_av_refresh_totals` is an **AFTER ... FOR EACH ROW** trigger, and PostgreSQL
queues after-row triggers to the **end of the statement**. So a single
`updateMany` that soft-deletes every leg of a voucher lets the trigger see only
the final state — 0 debit, 0 credit — which **balances**, and
`ck_avh_balanced` passes even while the header is POSTED.

Verified 2026-09-17 against rct00039 on localhost, with the trigger installed
inside a rolled-back transaction (it is absent by default — see
[[voucher-totals-trigger-missing]]):

- `UPDATE ... SET av_is_deleted=true WHERE av_voucher_id=<v>` (all 5 legs) →
  **allowed** while POSTED, totals land at 0.00 / 0.00.
- the same restricted to `av_dr_cr='CR'` → **refused**, `ck_avh_balanced`.

**Why it matters:** the natural assumption — "you cannot strip legs off a
POSTED voucher" — is false as usually written, so a guard justified only by
that constraint is not doing the work you think. `/receipts/amend`'s unwind
drops the header to DRAFT for a reason that actually holds: `saveInTransaction`
and `assertStatusMayPost` both demand a DRAFT, which is what lets amend call
§5.1 and §5.2 unmodified. The constraint is the second reason, and it only
bites if the leg removal ever stops being one statement.

**How to apply:** do not rely on `ck_avh_balanced` to catch a partial leg
removal, and do not rely on "we happen to do it in one statement" to stay
inside it — a per-voucher loop or a batch reintroduces the one-sided
intermediate state. Check the status gate you actually need. Related:
[[receipt-amend-unwind-relies-on-partial-indexes]].
