---
name: company-scope-nullable
description: "NULL company_id on ledgers/groups/customers means SHARED BY EVERY COMPANY — never \"fix\" it to NOT NULL"
metadata: 
  node_type: memory
  type: feedback
  originSessionId: 710f791a-1472-4ec1-9cd3-20c6a06e4ac6
  modified: 2026-09-17T12:35:18.937Z
---

**`led_company_id`, `acc_group_company_id`, `cus_company_id`, `sup_company_id`
are nullable ON PURPOSE. NULL = the row is shared by every company.** One
customer can be billed in all four companies off ONE ledger.

**Why:** the user corrected me on 2026-09-17 after I recommended making them
NOT NULL to close a duplicate-name hole. Forcing a scope would mean four copies
of the same party, four ids, and four sets of bills to reconcile.

Both modes are in live use: 50 of 61 ledgers are global, 11 are scoped (9 Acme
Foods, 1 CHANDRA SAW MILL, 1 HAP Solutions). All 48 groups are global.

**How to apply:**

1. Uniqueness is "unique within what one company can SEE" — its own rows plus
   the globals. Three rules; an index covers only two (global-vs-global,
   same-company); **global-vs-scoped needs a trigger**. The live
   `VAPI BATTERY & AUTO ELECTRICALS` pair (a global supplier ledger + an
   Acme-scoped customer ledger, same name) is that third case, and the API's
   "already exists for this company" 409 does not catch it.
2. **A balance can never live on a shared row.** `acc_opening_balance`
   (`op_company_id + op_branch_id + op_acc_year`) is the correct home and is
   already built. `led_ob_amount`/`led_ob_type`/`led_ob_as_on`/`led_total_dr`/
   `led_total_cr`/`led_total_balance` on the ledger row are single-valued and
   should come out of the save DTO — they are 0 everywhere today.
3. For the Tally export: a Tally company is its own file, so exporting company X
   emits the globals PLUS X's own rows, each carrying only X's balance. Two
   `<LEDGER>` entries with the same NAME get merged by Tally.

See [[ledger-master-gap-analysis]], [[acc-group-master-contract]].
