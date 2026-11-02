---
name: loyalty-schema-design
description: "Loyalty/coupon schema 12-14 in the Prathap share — user's locked decisions (statistical-only, FIFO lots, cooling period, member=cus_id) and the ck_/chk_ convention split"
metadata: 
  node_type: memory
  type: project
  originSessionId: cbe9dc79-0d2e-480a-8d58-bd08edfd2fbc
  modified: 2026-08-18T05:06:14.456Z
---

Loyalty program schema written 2026-08-18 as `12_loyalty_scheme.sql`, `13_loyalty_ledger.sql`, `14_loyalty_coupon.sql` in the share (see [[db-schema-share-location]]). Applied and functionally verified against a throwaway PG18 cluster.

**User's locked decisions** (asked directly, so do not re-derive or "improve" these):
- **Statistical only** — earning points posts *nothing* to the GL. No liability accrual, no expense provision. The GL is touched only at redemption, by the existing `acc_tender_detail` type-10 row.
- **Redemption is a tender line** (points type 10, coupons type 11). A *gift* redemption is NOT a tender — no money crosses the counter, stock does; it gets `sales.loyalty_gift_issue`.
- **Member identity is `sales.customers.cus_id`.** Card number and mobile are lookup handles only, never identity. One wallet per (company, customer) — never per branch, which is what makes points portable across a chain.
- **FIFO by expiry, then earn date, never-expiring last.** Declared once in `sales.fn_loyalty_lots()` and nowhere else.
- **Claw-back defence is a cooling period** (`ls_activation_days`), not a negative balance and not blocking the return. Residual case (return after cooling, points already spent) → cap at live balance, record shortfall in remarks. *This last bit was my assumption, flagged to the user, not yet confirmed.*
- Earn on bill save, offline included; cancel/return writes a reversal row, never a delete.
- Coupons: **sold gift vouchers (reloadable) and bulk campaign codes only.** No milestone-issued, no points→coupon conversion.
- **No tier master** — tiers explicitly out of scope.

**Convention split, unresolved:** the pre-existing `loyalty_sch_*` tables use `chk_` / `<table>_pkey` / `_updated_on` / `created_by uuid` / `CURRENT_TIMESTAMP`; the `00`–`11` chain uses `ck_` / `pk_` / `_modified_on` / `created_by varchar(50)` / `now()`. File 12 follows the former, 13 and 14 the latter (their txn tables must be partitioned with composite PKs anyway). Flagged to the user for a decision with the backend dev.

**Pre-existing bug found:** `04_tenders.sql:208` creates `ix_tnd_tally_guid` on `tnd_tally_guid`, a column absent from the `CREATE TABLE`. It aborts a fresh chain run at file 04 and everything downstream that needs `acc_tender_detail`. Not mine to fix; reported.

**Why:** chain-store ERP, tables cannot be altered after go-live, so the decisions above were settled before writing DDL.
**How to apply:** the wallet/ledger pair copies `09_acc_bill_balance.sql` exactly — unpartitioned parent with trigger-maintained roll-up caches, partitioned per-event child, write-through CHECK guards, reversal rows never deletes. Any further loyalty table must do the same. See [[sales-backend-schema-design]], [[party-adjustment-design]].

## Corrected 2026-08-18 after read-only access to the live ERP db

Connected as a read-only role (`claude`, `pg_read_all_data`, verified zero write privilege on all 151 tables). Files were then fixed against reality and re-verified on a restored replica. Five things guessing had got wrong:

1. **Vocabularies.** Real values: `ls_type` = BOTH/REDEEM (not POINTS/GIFT/COMBO); `ls_apply_on` = BILL_AMOUNT/BILL_QTY/ITEM_AMOUNT/ITEM_QTY; `ls_cust_type` = CUSTOMER/CUSTOMER_GROUP; `ls_item_type` = ITEM/ITEM_GROUP/ITEM_CATEGORY/ITEM_BRAND/ITEM_SECTION; `ls_expiry_basis` = NONE/EARN_DATE/MONTH_END/YEAR_END/SCHEME_END_DATE; `ls_valid_weekdays` holds **day names** (MON,TUE), not ISO numbers.
2. **`lspt_item_id` is polymorphic on `ls_item_type`** (ITEM→item_master, ITEM_GROUP→item_group_master, ITEM_CATEGORY→item_category_master; verified against every row). It must NEVER get an FK. Same for `lps_scope_id` on `ls_cust_type`. This is why the planned `loyalty_sch_item` table was dropped — the existing design already does per-item/group/brand scoping.
3. **`public.fn_create_year_partitions` does not exist on the live db** even though 00_init.sql defines it. Files 13/14 now create their own partitions in a self-contained DO loop.
4. **Overnight time windows are real data** (22:22→04:44), so there is deliberately no `to_time >= from_time` CHECK.
5. Master PKs: `cust_groups.cgr_id`, `item_group_master.itg_id`, `item_category_master.category_id`, `item_brand_master.brand_id`, `item_section_master.sec_id`, `item_price_master.ipm_id` (this is where `ipm_loyalty_points` lives, NOT item_price_levels).

**The live db is managed by Prisma** (`public._prisma_migrations`, 277 rows) — the middleware team's tool. Raw SQL applied outside Prisma reads as drift, and `migrate reset`/`db push` could drop the triggers, CHECKs, partial indexes and partitions, none of which Prisma can represent. Deliver 12/13/14 as a recorded migration.

Live db has diverged from the share files: `public.txn_charge_detail` (not `sales.sale_doc_charge`), `sale_quotation`, `acc_pdc_register`, `acc_opening_balance` exist; **`sale_return` does not** (so 13's ALTER for it no-ops). `accounts.acc_voucher_seq` exists despite the files claiming no sequences. `fixed.*` is the config-driven UI layer (menu_master, grid_details/columns, dropdown_details/columns, form_section/field) — a Loyalty Scheme master screen needs rows there.

## Superseded again 2026-08-18 (later): standalone tables, real FKs

User decided the old `loyalty_sch_*` tables are NOT to be considered. The files no longer amend them — they define a standalone family, left alongside the old tables which are untouched:
`loyalty_scheme` (lsc_) + `_branch` (lsb_) + `_party` (lsp_) + `_item` (lsi_) + `_slab` (lss_) + `_gift` (lsg_) | `loyalty_member` (lmb_) + `loyalty_ledger` (lld_) + `loyalty_gift_issue` (lgi_) | `loyalty_coupon_batch` (lcb_) + `loyalty_coupon` (lcp_) + `loyalty_coupon_txn` (lct_).

**Polymorphic scope ids are gone.** Every master was confirmed in the catalog, so scope is now TYPED columns each with a real FK plus a CHECK that exactly one is set — `loyalty_scheme_item` has item/group/category/brand/section, `loyalty_scheme_party` has cust/cust_group. Referential integrity is real.

FK targets verified against the live catalog (all `ON UPDATE CASCADE ON DELETE RESTRICT`; children of a scheme use `ON DELETE CASCADE`):
`companys(comp_id)` `branch_master(br_id)` `user_master(usr_id)` `customers(cus_id)` `cust_groups(cgr_id)` `item_master(item_id)` `item_group_master(itg_id)` `item_category_master(category_id)` `item_brand_master(brand_id)` `item_section_master(sec_id)` `item_price_levels(ipl_id)` **integer** `item_unit_conversion(iuc_id)` `acc_ledger_master(led_id)` `acc_tender_master(tnd_id)` `acc_tender_detail(td_id, td_acc_year)` `device_master(dev_id)`.

TWO EASY MISTAKES, both corrected: a document-line UNIT is `inventory.item_unit_conversion(iuc_id)`, NOT `item_unit_master(unit_id)` (that is what `sale_bill_item.fk_sbi_item_unit` uses). A PRICE LEVEL is `inventory.item_price_levels(ipl_id)`, NOT `fixed.price_levels(price_lvl_id)` — two different tables.

Audit block taken from the live tables (`acc_pdc_register`, `sale_bill`): `_is_active, _is_deleted, _sync_date, _created_on, _created_by varchar(50), _modified_on, _modified_by`. NOT `_updated_on`, NOT `created_by uuid`.

Verified: 12 tables (3 partitioned), 74 declared FKs, 117 CHECKs, 13 functions; applies clean and idempotently; **every FK has a covering index** (deliberately, since 133 FKs elsewhere in the live db do not).
