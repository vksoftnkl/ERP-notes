---
name: promotion-schema-design
description: "15_promotion_scheme.sql — the chain-store promotion master: two-column rule model, prm_/prb_/prp_/pri_/prs_/pru_ family, composite-FK benefit enforcement, EXCLUDE tie-break"
metadata:
  type: project
---

Promotion scheme master written 2026-08-19 as `15_promotion_scheme.sql` in the share
(see [[db-schema-share-location]]). Verified on a throwaway PG18 cluster restored from
`erp.sql`, with 12→13→14→15 applied on top: 6 tables, 3 partitions, 32 FKs, 45 CHECKs,
1 EXCLUDE, **0 FKs without a covering index**, idempotent on re-run.

**User's locked decisions** (do not re-derive):
- **Legacy is reference only.** `public.promotion_schemes/_items/_offers` are NOT migrated,
  read or referenced. Clean-room rewrite. User said so explicitly after pasting the old DDL.
- **Chain store is the design centre**, not a single shop.
- Deliverable was first scoped to the SQL file alone; the user then asked for the entry-screen
  mockup too, so `promotion_scheme_ui_mockup.py` → `.png` also exists (PIL, not matplotlib —
  same helpers as `loyalty_scheme_ui_mockup.py`). Still no schema-map .py and no `fixed.*` seed.
  The mockup is 3 stacked windows: Scheme tab, Items & bands tab, Branches tab.

**The rule model is two orthogonal columns, not the five legacy scheme types:**
`prm_apply_on` BILL_AMOUNT|BILL_QTY|ITEM_AMOUNT|ITEM_QTY × `prm_benefit`
FREE_ITEM|DISC_PERC|DISC_AMT|FIXED_PRICE. All five 3.0 types are combinations;
FIXED_PRICE is new and free. `prm_apply_on` reuses loyalty's `lsc_apply_on` vocabulary.

**Prefixes `psc_`/`psb_` are TAKEN** by `stock.physical_stock_*`. The family is
`promotion_scheme` (prm_) + `_branch` (prb_) + `_party` (prp_) + `_item` (pri_) +
`_slab` (prs_) | `promotion_usage` (pru_, partitioned by acc_year).

**Scope rows are KIND + SCOPE_ID, with the FKs kept by GENERATED columns.** The user asked for
one id column picked by kind (the 3.0 shape), because the entry grid is two columns wide. Plain
polymorphic ids take no FK — the flaw the loyalty rewrite removed — so the resolution is:

```sql
prp_kind     varchar(20) NOT NULL,   -- CUSTOMER | CUSTOMER_GROUP | AREA | CITY
prp_scope_id uuid NOT NULL,          -- what the form writes; no FK of its own
prp_cust_id  uuid GENERATED ALWAYS AS
    (CASE WHEN prp_kind = 'CUSTOMER' THEN prp_scope_id END) STORED
    REFERENCES sales.customers(cus_id) ON UPDATE NO ACTION ON DELETE RESTRICT,
-- ...one per kind. pri_ has five (ITEM|ITEM_GROUP|ITEM_CATEGORY|ITEM_BRAND|ITEM_SECTION,
-- the live ls_item_type vocabulary). 8 generated columns across the two tables.
```

**CITY scope (`sales.city_master.ctm_id`) is the level above AREA.** Priority seed is now
narrowest-first: CUSTOMER 4, AREA 3, **CITY 2**, CUSTOMER_GROUP 1 — CITY ranks *below* AREA
because one city holds many areas. **A CITY rule reaches a customer only through
`customers.cus_area_id -> area_master.arm_city_id`.** `customers.cus_city` EXISTS but is free
text with no FK (live values include 'NAMAKKAL', 'test2', 'SINGARATHOPE') — never match on it.
A customer with no area is in no city; live data currently has none such.

**`ck_pri_unit` is a BICONDITIONAL**: `(pri_kind='ITEM') = (pri_unit_id IS NOT NULL)` — the unit is
REQUIRED on an ITEM row and forbidden on the other four kinds. User asked for this directly.
Reason: "10 of item X" is not a quantity until you say ten of what, and a unit row belongs to one
item. `pri_unit_id` FKs `inventory.item_unit_conversion(iuc_id)`, the same target a bill line uses.
NOT enforced: that the unit belongs to the chosen item — would need a composite FK to
`(iuc_id, iuc_item_id)`, and that table has only a PK on `iuc_id`, so it would mean adding a
UNIQUE to a LIVE table. Left as a write-path check.

**Tested, not assumed** — the db still refuses: a scope_id in no master; kind=CUSTOMER holding an
area id; an UPDATE of kind that orphans the id; deleting a master a live campaign points at
(isolated against an area no customer referenced, so `fk_prp_area` was provably the objector);
and any direct write to a generated column.

**`ON UPDATE NO ACTION`, not CASCADE** — PostgreSQL *forbids* a cascading update on a key
containing a generated column ("invalid ON UPDATE action for foreign key constraint containing
generated column"). The only house-rule departure in the file. Harmless: master keys are uuidv7
and never reissued.

**Prisma caveat for the backend dev:** generated columns must be mapped read-only
(`@default(dbgenerated())` / `@ignore`); an ORM that INSERTs into `prp_cust_id` is refused.

This replaced an earlier draft that had typed columns + a stored kind + `ck_*_kind_target` CHECKs.
The generated-column form is strictly better: the agreement is *derived*, so no CHECK is needed
and it cannot be violated. Don't reintroduce the CHECKs.
**12 WAS CONVERTED TO MATCH on 2026-08-19** (user: "change loyalty table scripts too based on
this, i havent run yet"). `loyalty_scheme_party` is now `lsp_kind` + `lsp_scope_id`
(CUSTOMER|CUSTOMER_GROUP) and `loyalty_scheme_item` is `lsi_kind` + `lsi_scope_id`
(the five ITEM_* values), both with generated FK carriers and `ON UPDATE NO ACTION`.
`ck_lsp_one_target` / `ck_lsi_one_target` are GONE — derived, not tested.
Loyalty deliberately has **no AREA or CITY** kind: a wallet follows the person, not the route.
Adding them would be one generated column + one FK each.
13 and 14 needed no change (no references to the dropped columns). Whole chain 12→13→14→15
applies clean and idempotently; 135 FKs across the four files, 0 without a covering index.
(The 4 uncovered FKs the audit reports are on the OLD live `loyalty_sch_*` tables, not ours.)

**Two enforcement tricks worth remembering:**
1. `promotion_scheme` carries a redundant `UNIQUE (prm_id, prm_benefit)` purely so the slab
   can hold a **composite FK** `(prs_prm_id, prs_benefit) → (prm_id, prm_benefit)`. A
   percentage scheme therefore physically cannot hold a free-item band — header/detail
   agreement without a trigger. It also carries the ON DELETE CASCADE, so the slab needs no
   second plain FK.
2. `ex_prm_exclusive_clash` — an `EXCLUDE USING gist` refusing two EXCLUSIVE campaigns with
   the same company/branch/trigger/priority whose **date ranges overlap**. Replaced a first
   attempt at a plain `ux_prm_primary` unique index, which wrongly collided with campaigns
   in different months. **btree_gist 1.8 is already installed on the live db** (along with
   pgcrypto), so the required `CREATE EXTENSION IF NOT EXISTS btree_gist` is a no-op there.

**Discount columns are the trio perc/qty/amt on both `pri_` and `prs_`**, deliberately
matching `sbi_sch_disc_perc/_qty/_amt` — the master must not force the till to convert,
because a converted figure is the one that later disagrees with the printed bill.
`pri_disc_qty` / `prs_disc_qty` mean **rupees off PER UNIT**.

**Nothing on the document side needed changing** — `sale_bill_item` already has
`sbi_is_promo/_is_free/_free_type='SCHEME'/_src_free_qty/_scheme_id/_scheme_name/
_sch_disc_*/_bill_sch_*` and `sale_bill` has `sb_has_promo/_sch_disc/_bill_sch_disc`.
`sbi_scheme_id` is deliberately left **without** an FK to `prm_id`: the bill is partitioned
and offline-first, so a till must be able to save against a since-closed scheme; the name is
snapshotted for that reason. `fixed.menu_master` already has **menu_id 66 "Promotion Scheme"**.

**`promotion_usage` exists only because a chain cannot count from the bill** — the bill that
spent the last of the budget may have been raised offline at another branch. Caps
(`prm_max_uses_*`, `prm_budget_amount`) are ADVISORY at an offline till and authoritative
server-side after sync. Reversal rows, never deletes, so the running total is a plain SUM.

**Why:** chain-store ERP, tables cannot be altered after go-live.
**How to apply:** run order 12→13→14→15; deliver as a recorded Prisma migration, not raw SQL.
See [[loyalty-schema-design]], [[sales-backend-schema-design]], [[charge-distribution-model]].
