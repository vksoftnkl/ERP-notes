# `stock.stock_ledger` — every column, and what it is for

Companion to `16_stock.sql` §5 (lines 770-1015). One row = **one movement of one lot, in one
godown, in one direction, caused by one document line**.

Three rules govern the whole table, and most of the column choices below follow from them:

1. **Append-only.** Never updated in place, never deleted — `tr_sml_immutable` (§17) raises on
   both. A cancellation is a *reversing row*, not an edit.
2. **Quantities are magnitudes.** The sign lives in `sml_direction` alone. `ck_sml_qty_sign`
   refuses a negative quantity, because a negative qty with a negative direction is a silent
   double negation.
3. **There is no running-balance column.** Deliberately — see the bottom of this file.

Legend for **Written by**: *caller* = whatever posts the movement (the sales service, a stock
voucher, the migration script); *trigger* = the engine fills it; *generated* = Postgres computes
it and nothing may write it.

---

## Tenant and location — where this happened

| Column | Type | Written by | What it is / why |
|---|---|---|---|
| `sml_id` | `uuid` `uuidv7()` | default | Row key. v7 is time-ordered and minted at the counter, so an offline till can generate one that will not collide. PK is `(sml_id, sml_acc_year)` — the table is partitioned. |
| `sml_company_id` | `uuid` NOT NULL | caller | Tenant. FK → `public.companys`. In every index's leading position. |
| `sml_branch_id` | `uuid` NOT NULL | caller | Which shop. FK → `public.branch_master`. |
| `sml_tenant_id` | `uuid` | caller | Reserved for the multi-tenant layer; nullable and unused today. |
| `sml_acc_year` | `char(9)` NOT NULL | caller | `'2026-2027'`. **This is the partition key** — `PARTITION BY LIST (sml_acc_year)`. `ck_sml_acc_year` enforces the shape *and* that the second year is the first + 1. A missing partition is not a warning, it is every insert for that year failing, which is why §0 ships `fn_create_stock_partitions`. |
| `sml_godown_id` | `uuid` NOT NULL | caller | Where the stock physically is. FK → `inventory.godown_locations` (a tree: WAREHOUSE/ZONE/AISLE/RACK/SHELF/BIN — hold stock at whatever level the branch operates at). **Never nullable**: an unlocated movement cannot be reversed, counted or transferred. `branch_master.br_default_godown_id` resolves it for screens with no godown picker. |

---

## What moved

| Column | Type | Written by | What it is / why |
|---|---|---|---|
| `sml_item_id` | `uuid` NOT NULL | caller | FK → `inventory.item_master`. |
| `sml_lot_id` | `uuid` NOT NULL | caller | FK → `stock.stock_lot`. **Always set, even for an item tracked by nothing** — such an item gets exactly one lot, for ever, with every identity column NULL. That is what removes all the branching: there is no "no lot" code path. Obtain it from `fn_slt_resolve()`, never by inserting into `stock_lot` yourself. |
| `sml_uom_id` | `uuid` NOT NULL | caller | The unit the **document** was keyed in. FK → `inventory.item_unit_conversion(iuc_id)`. **`iuc_id`, not `item_unit_master.unit_id`** — this is the mistake the file warns about twice, and it matters because `sale_bill_item.fk_sbi_item_unit` points at `iuc_id`; stock a bill consumes must be keyed the way the bill keys it. |
| `sml_base_uom_id` | `uuid` NOT NULL | caller | The item's base unit, also an `iuc_id`. Stored so a report can show "3 cases" and still sum pieces. |
| `sml_to_base_factor` | `numeric(18,6)` NOT NULL, dflt 1 | caller | Conversion from keyed unit to base. `ck_sml_to_base_factor` requires `> 0`. Storing it per row is what makes history interpretable after a unit definition changes. |

> ⚠️ **Known hole.** Neither `ux_slt_identity` nor `ux_sbl_scope` includes the base unit, so
> changing an item's base unit while stock exists makes new movements convert to a *different*
> base and be **added to the old total** — 500 PIECE + 3 BOX = 503. `fn_sbl_rebuild()` cannot
> detect it: it sums `sml_signed_base_qty` and takes the base unit as
> `(array_agg(sml_base_uom_id))[1]`, mixing them the same way. The ledger itself stays
> recoverable because these two columns are per row. Freeze the base unit once stock exists.

---

## Which document caused it

The `(module, doc_type, doc_id)` triple is the same one `acc_tender_detail`, `txn_status_log`,
`txn_hold` and `acc_voucher_doc_register` use. A new document type that moves stock costs one
CHECK value here and nothing else.

| Column | Type | Written by | What it is / why |
|---|---|---|---|
| `sml_src_module` | `varchar(20)` NOT NULL | caller | `SALES` · `PURCHASE` · `STOCK` · `LOYALTY` · `PRODUCTION` · `MIGRATION` (`ck_sml_src_module`). |
| `sml_src_doc_type` | `varchar(30)` NOT NULL | caller | e.g. `SALE_BILL`, `GRN`, `STOCK_VOUCHER`. Free-form within the module. |
| `sml_src_doc_id` | `uuid` NOT NULL | caller | The document's id. **Polymorphic — no FK**, because it points into several tables. |
| `sml_src_acc_year` | `char(9)` | caller | The **source document's** year, which is not always this row's year: an April sale return against a March bill posts into the new year and points back into the old one. |
| `sml_src_refno` | `varchar(100)` | caller | The document number as printed, so a stock report reads without a join. |
| `sml_line_no` | `integer` NOT NULL, dflt 1 | caller | The document line. `ck_sml_line_no` ≥ 1. |
| `sml_split_no` | `integer` NOT NULL, dflt 1 | caller | Which split of that line — one bill line drawing from three lots is three rows. `ck_sml_split_no` ≥ 1. |

**Why the grain is exactly this:** `(doc, line, split, godown, lot, direction)` is precisely what
`sale_bill_item` already emits — `sbi_line_no × sbi_split_no × sbi_godown_id × sbi_stock_id` — so
posting a bill is a straight `INSERT … SELECT` with no reshaping and no aggregation.

**Idempotency.** `ux_sml_source` is unique over
`(company, branch, acc_year, module, doc_type, doc_id, line_no, split_no, godown, lot, direction)`
where `NOT deleted AND NOT reversal`. The same bill posted twice — a retry after a timeout, a sync
replaying a day — cannot move stock twice. Direction is in the key because a transfer legitimately
writes an OUT and an IN for one source line.

---

## The movement itself

| Column | Type | Written by | What it is / why |
|---|---|---|---|
| `sml_txn_type` | `varchar(30)` NOT NULL | caller | 20 values (`ck_sml_txn_type`): `OPENING` `PURCHASE` `PURCHASE_RETURN` `SALE` `SALE_RETURN` `TRANSFER_OUT` `TRANSFER_IN` `ADJUST_PLUS` `ADJUST_MINUS` `PHYSICAL_PLUS` `PHYSICAL_MINUS` `DAMAGE` `EXPIRY_WRITEOFF` `REPACK_IN` `REPACK_OUT` `BUCKET_IN` `BUCKET_OUT` `GIFT_ISSUE` `SAMPLE_ISSUE`. |
| `sml_direction` | `smallint` NOT NULL | caller | `+1` in, `-1` out (`ck_sml_direction`). **Redundant against `sml_txn_type` on purpose**, so a report that sums need not know that `EXPIRY_WRITEOFF` is an outward. |
| `sml_bucket` | `varchar(20)` NOT NULL, dflt `SALEABLE` | caller | `SALEABLE` · `DAMAGED` · `QUARANTINE` · `EXPIRED` · `SAMPLE`. Saleable and damaged stock are both stock and must never be added together by accident — each bucket gets its own balance row. Moving between buckets is a `BUCKET_OUT` + `BUCKET_IN` pair. |
| `sml_doc_date` | `date` NOT NULL | caller | The document's date. **What every report and every as-of query filters on**, and what `fn_stp_effective` / `fn_sml_cost_default` resolve the policy against. |
| `sml_doc_datetime` | `timestamptz` NOT NULL, dflt `now()` | caller | Time of day, for same-day ordering at a till. |
| `sml_posted_on` | `timestamptz` NOT NULL, dflt `now()` | default | When the row physically arrived. Differs from `doc_datetime` for anything posted offline and synced later — that gap is the audit trail for a back-dated entry. |

---

## Quantity

| Column | Type | Written by | What it is / why |
|---|---|---|---|
| `sml_qty` | `numeric(18,6)` | caller | As keyed, in `sml_uom_id`. |
| `sml_base_qty` | `numeric(18,6)` | caller | The same, in base. `qty × to_base_factor`. |
| `sml_free_qty` | `numeric(18,6)` | caller | Free goods as keyed. |
| `sml_free_base_qty` | `numeric(18,6)` | caller | Free goods in base. |
| `sml_weight_qty` | `numeric(18,6)` | caller | Weight, for loose/weighed lines. Informational; not summed into stock. |
| **`sml_signed_base_qty`** | `numeric(18,6)` | **generated** | `direction × (base_qty + free_base_qty)`. **The one column every quantity report sums and the only one `stock_balance` reads.** Nothing may write it. |

**Free goods are stock.** They sit on the shelf and they get sold, so they count in on-hand. They
are kept in their own columns only so margin reporting can separate what was paid for from what
was not — which is why `signed_base_qty` adds both.

Two CHECKs worth knowing: `ck_sml_qty_sign` (all four quantities ≥ 0) and `ck_sml_qty_nonzero`
(`base_qty + free_base_qty > 0`) — a movement of nothing is not a movement.

---

## Value

On an **inward** these are what it cost. On an **outward** they are what it cost *us* — the COGS,
taken from the lot or the moving average per `stp_valuation_method`. **Never the selling price**,
which is `sml_doc_rate`.

| Column | Type | Written by | What it is / why |
|---|---|---|---|
| `sml_cost_rate` | `numeric(18,6)` | caller **or** `fn_sml_cost_default` | Per base unit. If the caller leaves it 0 on an outward, the trigger fills it: FIFO/LOT_ACTUAL → the lot's `slt_cost_rate`; WAVG → `stock_item_cost`. Set it yourself only when you know better than the policy. |
| `sml_cost_value` | `numeric(18,2)` | same | `cost_rate × qty`. |
| `sml_cost_rate_wot` | `numeric(18,6)` | same | Cost without tax. |
| `sml_cost_value_wot` | `numeric(18,2)` | same | Value without tax — what `fn_sdd_rebuild` uses for margin. |
| `sml_landed_rate` | `numeric(18,6)` | caller | Cost after freight, duty and other landed charges. |
| `sml_landed_value` | `numeric(18,2)` | caller | The same, extended. |
| `sml_doc_rate` | `numeric(18,6)` | caller | What the **document** said — the selling price on a sale. Kept beside cost so margin needs no join back to `sale_bill_item`. |
| `sml_doc_rate_wot` | `numeric(18,6)` | caller | Document rate without tax. |
| `sml_doc_amount_wot` | `numeric(18,2)` | caller | Document amount without tax. Indexed in `ix_sml_demand` for the demand rebuild. |

`sml_cost_*` and `sml_doc_*` sitting side by side is the whole point: gross margin is one row's
arithmetic, with no join and no repricing.

---

## Lot snapshots

| Column | Type | Written by | What it is / why |
|---|---|---|---|
| `sml_mrp` | `numeric(18,6)` | caller | Copied from the lot. |
| `sml_batch_no` | `varchar(100)` | caller | Copied from the lot. |
| `sml_expiry_date` | `date` | caller | Copied from the lot. |

Denormalised on purpose. They turn "stock movement by batch" and "what expired this month" into
single-table scans instead of a join to `stock_lot` for every row of a fifty-branch report.
`stock_lot` **owns** these values; nothing else may write them, and they must match the lot.

---

## Reversal

| Column | Type | Written by | What it is / why |
|---|---|---|---|
| `sml_is_reversal` | `boolean` NOT NULL, dflt false | caller | This row undoes another. |
| `sml_reverses_id` | `uuid` | caller | Which row. `ck_sml_reversal` binds the pair: both set, or neither. |

A cancelled bill writes reversing rows; it does not remove the originals. Two reasons, and the
second is the business one: *a stock figure that can change retrospectively is a stock figure
nobody can reconcile* — and in an offline chain the cancellation may arrive after the day was
closed. Note `ux_sml_source` excludes reversals, so a reversal never collides with the row it
undoes — and `ux_sml_reversal` is unique over `(sml_reverses_id, sml_acc_year)`, so a row can
be reversed only once: a concurrent or replayed cancellation fails instead of double-reversing.

---

## Context and audit

| Column | Type | Written by | What it is / why |
|---|---|---|---|
| `sml_reason_id` | `uuid` | caller | FK → `stock.stock_reason_master`. *Why* it moved — damage, theft, sample, correction. Required by policy on adjustments, not by the schema. |
| `sml_party_id` | `uuid` | caller | Customer or supplier, depending on the module. **No FK** — it points into two tables. |
| `sml_narration` | `text` | caller | Free note. |
| `sml_is_deleted` | `boolean` NOT NULL, dflt false | — | Soft-delete flag. **Every index on this table is filtered on `= false`.** In practice it stays false: the table is append-only and a mistake is corrected by a reversal, not a delete. |
| `sml_sync_date` | `timestamptz` | sync | When this row reached the server. NULL = still local. |
| `sml_created_on` | `timestamptz` NOT NULL, dflt `now()` | default | |
| `sml_created_by` | `uuid` | caller | FK → `public.user_master`. |

---

## The column that is deliberately missing

There is **no** "balance after this movement" column. That is the classic stock-ledger trap:

- two tills posting concurrently would have to serialise on it,
- a back-dated entry invalidates every row after it,
- and an offline chain has no global order to compute it in.

On-hand is `stock_balance`'s job and nothing else's. If you need a balance as of a date, that is
`fn_stock_asof()`.

---

## The five indexes, and the question each answers

| Index | Answers |
|---|---|
| `ux_sml_source` (unique) | "has this document line already posted?" — idempotency |
| `ux_sml_reversal` (unique) | "has this row already been reversed?" — one reversal per ledger row |
| `ix_sml_item_date` | "what has moved for this item lately?" — the item ledger screen |
| `ix_sml_balance_scope` | "rebuild the balance for this godown/lot/bucket" — `fn_sbl_rebuild` |
| `ix_sml_lot` | "everything that ever happened to this lot" — recall, ageing |
| `ix_sml_source_doc` | "what did this bill move?" — drill-down from a document |
| `ix_sml_demand` | the `SALE`/`SALE_RETURN` slice by day — `fn_sdd_rebuild` |

---

## Posting one row: the checklist

1. `fn_slt_resolve(...)` → `sml_lot_id`. Never insert into `stock_lot` directly.
2. Fill the tenant/location five, the document triple, `line_no` / `split_no`.
3. `txn_type` + `direction` + `bucket`. Quantities as **magnitudes**.
4. `uom_id` / `base_uom_id` as `iuc_id`s, and `to_base_factor` between them.
5. Cost: leave `sml_cost_rate` at 0 on an outward and let `fn_sml_cost_default` apply the
   valuation policy, unless you genuinely know the rate.
6. `INSERT`. Four derived phases then run off that row — `stock_balance`, the moving
   weighted average and its distribution onto the holdings, the negative-stock policy, and
   the lot's total and status. **That is `fn_sml_apply`, and it has its own file:
   [`17_sml_apply_contract.md`](17_sml_apply_contract.md).** Read it before writing anything
   that posts stock; the summary in this line is not enough to implement against, and the
   engine that actually runs is the NestJS one, not the trigger.
7. Never `UPDATE` or `DELETE` — write a reversal.
