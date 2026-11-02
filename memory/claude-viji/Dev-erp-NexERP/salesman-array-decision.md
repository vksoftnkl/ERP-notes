---
name: salesman-array-decision
description: salesman is uuid[] on all five sales documents (not salesman2 columns); the FK that costs, the GIN indexes, and element-1-is-primary
metadata:
  node_type: memory
  type: project
---

**Decided 2026-09-21 by the user, after I recommended two columns and they
overruled it.** Their reasons, both good: *"no keep it in array for 5% ppl why
we need to change in case in future if it comes 3 means what to do"* — two
columns cap at two and a third salesman would be a migration — and separately
*"but loadman definitely need array"*.

**salesman = `uuid[]` on the HEADER of all five documents.** `sale_bill`,
`sale_order` and `sale_dc` already were; `sale_return` (29) and
`sale_quotation` (30 §1c `ALTER … TYPE`) were converted.
**Line-level stays a single uuid** (`sbi_`/`soi_`/`sdi_`/`sri_salesman_id`):
the DOCUMENT is credited to a team, a line to one person.
**loadman stays `uuid[]`** — open-ended crew, no primary, nothing joins it.
**driver stays single** — a vehicle has one.

**THE COST, and it is load-bearing: PostgreSQL cannot put a foreign key on the
elements of an array.** That is why sale_bill / sale_order never had one, and
why `fk_sq_salesman` had to be DROPPED and `fk_sr_salesman` deleted from 29.
Nothing in the database will ever catch a salesman id that does not exist, is
soft-deleted, belongs to another company or is a typo. Replaced by
`assertSalesmen()` in `sales.guards.ts` (plan-backend §3.7): resolves in
`employee_master`, this company's, not deleted, no duplicates, null rather
than `{}`, refuse `SALES_SALESMAN_INVALID`.

**Element 1 is the PRIMARY — a convention the schema cannot enforce**, so it is
written in three places (30 §1d, plan-backend §3.7, the client that builds the
array). `acc_bill_balance.abl_salesman_id` stays a single uuid with its btree
`ix_abl_salesman` and takes `sb_salesman_id[1]`, so outstanding-by-salesman
stays fast and credits ONLY the primary. A second/third salesman is on the
document, not in that report. If that must change, abl becomes uuid[] + GIN.

**Lookups need GIN**: `ix_sb_/ix_so_/ix_sq_salesman_gin` (30 §1c). Query shape
`WHERE sb_salesman_id @> ARRAY[:salesman]::uuid[]` — a btree cannot answer
containment; do not let a report fall back to an unindexed `= ANY(...)`.

**Context that shaped it:** 95% of companies run a single salesman, so the
array is carried for the 5%. Test box had 30 bills, 24 with a salesman, none
with more than one element. Verified on `erp_dry`: all five headers ARRAY.

**The five crew roles, settled 2026-09-21** — identical on all four
goods-moving documents (bill, sale return, challan, DC return):
`salesman_id uuid[]`, `supervisor_id uuid`, `driver_id uuid`,
`loadman_id uuid[]`, `vehicle_id uuid`. `supervisor_id` existed on the BILL
ONLY and was added to the other three; `sdr_salesman_id` was added so a
delivery and its reversal can sit side by side in a salesman report.
**`sale_order` and `sale_quotation` carry NONE of them on purpose** — the rule
is already written in `schema/06` line ~90: *"Driver / loadman / supervisor /
vehicle are NOT carried: despatch is the bill's fact."*
The SINGULAR roles now have FKs to `employee_master` / `vehicle_master`
(`fk_*_driver`, `fk_*_supervisor`, `fk_*_vehicle`) — 29 for the three new
documents, 30 §1b2 for `sale_bill`, which never had any. Safe: 0 test-box rows
have a driver or supervisor set, no orphans. The two ARRAY roles cannot carry
an FK and rely on the service check.

**That gap is CLOSED (2026-09-21, user: "add transport band to sale_return and
dc_return also").** Both returns now carry the challan's band with the
direction reversed: a PICKUP block (`*_pickup_*` — the e-way FROM, not the same
as the bill-to snapshot), RECEIVE (`*_receive_godown_id` + `*_receive_branch_id`)
and the full transport set incl. `*_vehicle_id`, `*_driver_id`,
`*_loadman_id uuid[]`. `sr_receive_godown_id` is NULLABLE (header default only —
`sri_godown_id` on the line is authoritative and a return may split across
godowns); `sdr_receive_godown_id` stays NOT NULL (one challan, one place).
`fk_*_transporter`/`fk_*_vehicle` for both added to 29's deferred ALTER block,
plus `ck_*_transport_mode`. Verified on erp_dry: all three documents carry an
identical band. plan-qt §5 and §6 updated.
