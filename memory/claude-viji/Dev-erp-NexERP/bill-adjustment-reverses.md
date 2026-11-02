---
name: bill-adjustment-reverses
description: "acc_bill_adjustment retracts by NEGATIVE counter-row, never by is_deleted — and the vouchers.sql spec still describes the old soft-delete design"
metadata:
  type: project
---

**`accounts.acc_bill_adjustment` never sets `abj_is_deleted` on a cancel.** It
retracts by inserting a **negative counter-row** pointing at the original. The
schema enforces it:

```
ck_abj_reversal_sign  CHECK (abj_reversal_of_id IS NULL     AND abj_amount > 0
                          OR abj_reversal_of_id IS NOT NULL AND abj_amount < 0)
ux_abj_reversal       UNIQUE (abj_reversal_of_id, abj_acc_year)  -- one reversal per original
ck_abj_no_self_reversal
ck_abj_cheque_mode    ... OR abj_reversal_of_id IS NOT NULL      -- carve-out FOR reversal rows
```

So a receipt cancel / amend / cheque bounce all **add** a row. `abj_is_deleted`
is a hard retraction that removes the row from every index (they are all
partial `WHERE abj_is_deleted = false`) — a cancel must stay visible, because
it is a thing that happened. **Seeing zero `is_deleted = true` is the design
working, not a missing step.** Verified live 2026-09-19: 332 cancelled
receipts, 452 originals under them, 452 reversals, zero unreversed.

`abj_reversal_reason` carries WHY — the cancel reason, or `"Amended: …"`, or
`"Cheque 55492 bounced: …"`. It is the field to group by when asking what
reversed something. (Grouping with `min(reason)` hides the cancels behind the
alphabetically-earlier `"Amended:"` ones — that misled a first pass.)

**A HELD (post-dated) cheque writes its adjustment row but does NOT settle the
bill.** `abl_alloc_amount` excludes it until the cheque clears. Any
reconciliation of live rows against `acc_bill_balance` MUST exclude
`apd_status = 'HELD'`, or exactly the PDC bills look like drift. With that
exclusion all 20 bills on the box reconciled to the paisa.

**THE SPEC FILE IS STALE AND SAYS THE OPPOSITE.**
`/home/vk/Shared/Prathap/vouchers.sql` still describes the ORIGINAL design:
`aba_`-prefixed columns, no reversal column at all, and every index
`WHERE aba_is_deleted = false` — i.e. soft-delete as the retraction. The
DEPLOYED table is `abj_`-prefixed with the reversal columns above, is
partitioned by `abj_acc_year`, and carries tender/cheque links the spec has
no trace of. Read the live `\d accounts.acc_bill_adjustment`, not the .sql.
Same class of divergence as [[stock-engine-not-deployed]].
