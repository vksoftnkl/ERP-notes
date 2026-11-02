# NestJS — ONE price table: `item_price_master` becomes the MRP-bucket table

Written 2026-09-30. A new file; it edits no existing plan. It **replaces Phase 3 of
`plan-nestjs-stock-engine-port.md`** and retires `stock.stock_mrp_price` before it was ever
created.

| It overrides | Section | What changes |
|---|---|---|
| `16_stock.sql` | §20 PART 2 — PRICING (16:2992-3344) | the table, its indexes and `fn_smp_effective` are **not built**; the reasoning in the §20 header stays true and moves here |
| `16q_stock_grid_queries.sql` | Q23-Q27, S1-S3 | re-targeted at `inventory.item_price_master` (§5 below) |
| `PRICING_AND_FORMS_PLAN.md` | Part 1 §1, §4 ("keep it… smp is an override") | one table, not two |
| `change_selling_screen_plan.md` | "Untracked items … route to the existing `item_price_master` endpoint" | there is no fan-out; every row is the same table |
| `plan-nestjs-stock-engine-port.md` | PHASE 3, Appendix A row 20 | this file |
| repo `plan/plan-nestjs-change-selling-price.md` | §0.1, §0.2, §1, §6 | the blockers are gone; §6's fan-out is deleted |
| `19q_opening_item_lookup.sql` Q6 price join | the smp join | reads the headline / dearest bucket row of `item_price_master` |

**Mockup:** `change_selling_one_table_mockup.png` (menu 30 redrawn for this design; supersedes
`change_selling_ui_mockup.png` of 09-04). The screen plan `change_selling_screen_plan.md` still holds
for keys, grid columns and the four-number panel.

**The whole change in one line:** a price row gets two nullable bucket columns (`ipm_bucket_mrp`,
`ipm_bucket_sp`); a row with both NULL is the headline the item has always had, a row with one set is
"what stock at THIS MRP sells for", and one resolver picks the exact bucket before the headline —
which is 3.0's `UPDATE stocks … WHERE max_price = …` on the price master instead of on stock.

---

## 0. Why one table, and why this one

The §20 header gives a good reason for pricing PER MRP (Hamam at MRP 40 and MRP 45 on the same shelf
sell for different money) and a good reason for NOT pricing per lot (two batches at one MRP sell for
the same money; per-lot prices drift). Both stand. The only reason it then became a **second table**
is written in the same header: "inventory.item_price_master (Prisma-owned, never ALTERed)". That was
an ownership rule of 2026-08, not a design reason, and it no longer holds — this repo owns both sides
and already ships raw-SQL migrations for the stock schema.

What the two-table design cost, verified 2026-09-30:

- `fn_smp_effective` is a UNION of two tables with a NOT EXISTS between them. Its TypeScript port
  (`PriceResolverService`, port plan §3) would be the same double read.
- Menu 30 needs a "headline fan-out" (`isHeadlineRow` → `ItemsPriceMasterService.save(tx)`) because a
  `(NULL, NULL)` bucket was forbidden by `ck_smp_identity`. With one table that whole branch is gone.
- `stock_mrp_price` carried only prices. The sale lookup takes cess, loading, freight, loyalty points,
  min price, cost and the default godown off the price row too. A bucket row in a separate table had
  none of them, so a bucket-priced line would have lost its cess.
- `item_price_master` already has sync triggers (`zz_sync_mark_dirty`), an audit projection and a
  60 s read cache. The new table would have needed all three again.

Why the LOT is still the wrong row (the 3.0 `stocks` row was one receipt line per branch and
godown; `stock.stock_lot` is not): the lot is **company-scoped identity shared by every branch**, so a
branch price has no lot to live on; batch+MRP items hold several lots at one MRP; and goods not yet
received have no lot at all. 3.0 met the last one by silently matching zero rows (the fault §20
refuses to copy). The lot keeps `slt_cost_rate`; it never carries a selling price.

**Live facts this plan is sized against** (192.168.0.106, 2026-09-30):

| Fact | Value |
|---|---|
| Active `item_price_master` rows / items with one | 48 / 30 (of 10,241 items) |
| Rows with `ipm_max_price > 0` | 18 |
| Rows whose A-D price exceeds their MRP | **5** — see §1.3 |
| Policies tracking MRP / sale price | 9 / 1 of 29 |
| Lots with an MRP or sale price | 0 of 143 (every lot is signature `N`) |
| `sales.cust_item_rates` rows | 0 |
| `btree_gist`, `ex_stp_overlap` | both present |

Nothing to migrate: no bucket stock exists, and the 48 rows become headline rows by default.

---

## 1. The migration (one Prisma migration, raw SQL, expand-only)

Named after `20260930150000_group_policy_shared`. Every statement is `IF NOT EXISTS` / re-runnable.

### 1.1 Columns

```sql
ALTER TABLE inventory.item_price_master
  -- The bucket: the price-defining half of the lot's identity. NULL = headline.
  ADD COLUMN IF NOT EXISTS ipm_bucket_mrp  numeric(18,6),
  ADD COLUMN IF NOT EXISTS ipm_bucket_sp   numeric(18,6),
  -- Null-normalised companions, the same sentinel stock_lot uses (slt_key_mrp / slt_key_sp),
  -- so the EXCLUDE below cannot be fooled by NULLs and the two tables agree by construction.
  ADD COLUMN IF NOT EXISTS ipm_key_mrp numeric(18,6)
      GENERATED ALWAYS AS (COALESCE(ipm_bucket_mrp, -1)) STORED,
  ADD COLUMN IF NOT EXISTS ipm_key_sp  numeric(18,6)
      GENERATED ALWAYS AS (COALESCE(ipm_bucket_sp,  -1)) STORED,
  -- Dormant effective dating, exactly as §20 had it: the defaults mean "always", a far-past start
  -- so a back-dated bill still finds its row. No screen writes these in v1.
  ADD COLUMN IF NOT EXISTS ipm_effective_from date NOT NULL DEFAULT DATE '1900-01-01',
  ADD COLUMN IF NOT EXISTS ipm_effective_to   date NOT NULL DEFAULT DATE '9999-12-31';
```

Not added, because the table already has them under another name — do not duplicate:

| §20 column | already in `item_price_master` |
|---|---|
| `smp_priced_on_cost` / `_wot` | `ipm_cost_price` / `ipm_cost_wot` — the cost the row was priced against, which is what the item card has always stored here |
| `smp_markup_*_perc` | `ipm_price_*_markup_perc` |
| `smp_min_price`, `smp_disc_perc`, `smp_disc_qty`, `smp_profit_type`, `smp_round_off` | same names with `ipm_` |
| `smp_mrp` (the printed MRP) | `ipm_max_price` — see 1.3: on a bucket row the two are one number |

### 1.2 The one-row-per-bucket rule

```sql
-- One price set per bucket per scope AT A TIME. Replaces uq_item_price_master_scope, which it
-- subsumes: with the default dates every range is [1900, 9999] and this is the plain
-- one-row-per-(scope, item, unit, bucket) rule. Company/branch are COALESCEd because EXCLUDE
-- never conflicts on NULL (the old index used NULLS NOT DISTINCT for the same reason).
-- A chain row (branch NULL) and a branch row for the same bucket DO coexist — that is the override.
-- DEFERRABLE so a split-and-reschedule saves as one transaction. Same construction as ex_stp_overlap.
ALTER TABLE inventory.item_price_master
  ADD CONSTRAINT ex_ipm_overlap EXCLUDE USING gist (
    (COALESCE(ipm_company_id, '00000000-0000-0000-0000-000000000000'::uuid)) WITH =,
    (COALESCE(ipm_branch_id,  '00000000-0000-0000-0000-000000000000'::uuid)) WITH =,
    ipm_item_id    WITH =,
    ipm_uc_unit_id WITH =,
    ipm_key_mrp    WITH =,
    ipm_key_sp     WITH =,
    daterange(ipm_effective_from, ipm_effective_to, '[]') WITH &&
  ) WHERE (ipm_is_deleted = false)
  DEFERRABLE INITIALLY IMMEDIATE;

DROP INDEX IF EXISTS inventory.uq_item_price_master_scope;
```

Add the EXCLUDE **before** dropping the index, in the same migration, so there is no window
without the rule. `ipm_godown_id` stays OUT of the key, exactly as today (notes 67 B1).

### 1.3 Checks

```sql
ALTER TABLE inventory.item_price_master
  ADD CONSTRAINT ck_ipm_bucket_mrp CHECK (ipm_bucket_mrp IS NULL OR ipm_bucket_mrp > 0),
  ADD CONSTRAINT ck_ipm_bucket_sp  CHECK (ipm_bucket_sp  IS NULL OR ipm_bucket_sp  > 0),
  -- A bucket row's MRP IS its max price. One MRP column on every screen, never two.
  ADD CONSTRAINT ck_ipm_bucket_mrp_is_max CHECK (
      ipm_bucket_mrp IS NULL OR ipm_max_price = ipm_bucket_mrp),
  -- §20's ck_smp_not_above_mrp, scoped to BUCKET rows as §20 scoped it (smp_mrp IS NULL → pass).
  -- Headline rows keep the client-side rule: 5 live test rows price above their MRP today, and a
  -- headline MRP of 0 (skip_mrp shops) must stay legal.
  ADD CONSTRAINT ck_ipm_not_above_mrp CHECK (
      ipm_bucket_mrp IS NULL
      OR (ipm_sales_price_a <= ipm_bucket_mrp AND ipm_sales_price_b <= ipm_bucket_mrp
          AND ipm_sales_price_c <= ipm_bucket_mrp AND ipm_sales_price_d <= ipm_bucket_mrp)),
  ADD CONSTRAINT ck_ipm_dates CHECK (ipm_effective_to >= ipm_effective_from);
```

### 1.4 Indexes

```sql
-- The billing lookup: item + unit + bucket key, then scope. Dates ride along.
CREATE INDEX IF NOT EXISTS ix_ipm_lookup
    ON inventory.item_price_master (ipm_item_id, ipm_uc_unit_id, ipm_key_mrp, ipm_key_sp,
                                    ipm_company_id, ipm_branch_id)
    INCLUDE (ipm_effective_from, ipm_effective_to)
    WHERE ipm_is_deleted = false;
```

`idx_item_price_master_item` stays for the callers that read all rows of an item.

### 1.5 Prisma model

Add the six columns to `prisma/inventory/itemPricemaster.prisma` (`ipmKeyMrp` / `ipmKeySp` as
`Decimal @default(dbgenerated())` read-only). The EXCLUDE and CHECKs are raw SQL, as the comment
block there already says for the index they replace. Update that comment.

---

## 2. The resolver — one function, every caller

```ts
// src/modules/Inventory/items-price-master/price-resolver.ts   (pure; the till reuses it offline)
export interface BucketKey { mrp: number | null; salePrice: number | null }
export type PriceSource = 'BUCKET' | 'MASTER';
export type PriceScope  = 'BRANCH' | 'CHAIN';

export function resolveEffectivePrice<R extends PriceRowScope>(
  rows: R[], key: BucketKey, onDate: string,
): { row: R; source: PriceSource; scope: PriceScope } | null
```

Rules, in the order they decide (this is `fn_smp_effective` 16 prestrip:4612-4706 with the UNION
collapsed):

1. Only rows in force: `ipm_is_deleted = false`, `onDate BETWEEN ipm_effective_from AND _to`.
2. Only rows of the caller's scope or wider: `(company = ? OR company IS NULL)`,
   `(branch = ? OR branch IS NULL)`.
3. **Exact bucket first**: rows whose `ipm_key_mrp = COALESCE(key.mrp, -1)` and
   `ipm_key_sp = COALESCE(key.salePrice, -1)`. Among them **branch beats chain, then company beats
   shared** — `ORDER BY (branch IS NULL), (company IS NULL)`, the `fn_stp_effective` pattern.
   Found → `source = 'BUCKET'`.
4. Else the same ordering over rows with key `(-1, -1)` → `source = 'MASTER'`.
5. Else `null` — "no price row", which the lookup already reports as 404 `Item price not found`.

`scope` is `'CHAIN'` when the winning row's `ipm_branch_id IS NULL`, else `'BRANCH'`. Menu 30's
`priceSource` / `priceScope` / `bucketId` columns are exactly these three outputs; nothing else
computes them.

**The bucket key is policy-blanked before it reaches the resolver.** A caller passes whatever the
line carries; `bucketKeyFor(policy, { mrp, salePrice })` keeps `mrp` only when the item's effective
policy has `track_mrp`, and `salePrice` only under `track_sale_price`. This is the rule
`resolveLots` already applies to lot identity (`stock-voucher-posting.helper.ts:367,
:569`) — **export that one function from the helper and call it from both places**, so a price
bucket and a lot can never disagree about which dimensions exist. An untracked item therefore always
resolves `(-1, -1)`, i.e. its headline, whatever MRP the bill typed. The policy comes from
`effectivePolicyLateral` (helper:288) at the document's date.

One Prisma fetch feeds it: all non-deleted rows of `(item, unit?)` at scope-or-wider. That is the
query `getItemPriceLookup` already runs (`item-price.lookup.ts`, `prisma.itemPriceMaster.findMany`),
so no new read.

---

## 3. `/master-lookups/item-price` — the sales side

The lookup already returns the four prices, the MRP (`max_price`), min, cost, cess, loading,
freight, loyalty and the default godown off ONE row picked by `selectUnitRate` +
`preferBranchPriceRows`. It stays that shape. Three changes:

### 3.1 Inputs

| new query param | meaning |
|---|---|
| `mrp?` (numeric) | the MRP on the line — what the operator sees on the packet, or retyped under `sales.allow_mrp_edit` |
| `sale_price?` (numeric) | the line's selling-price dimension, for `track_sale_price` items |
| `lot_id?` (uuid) | when the line already holds a lot (a picked lot, or a loaded line): its `slt_mrp` / `slt_sale_price` ARE the key, and `mrp` / `sale_price` are ignored |
| `doc_date?` (date) | the document's date for the effective window; default today |

### 3.2 Row pick

Replace the current "pick the unit row" with: policy-blank the key (§2) → `resolveEffectivePrice`
over the rows of the chosen unit → that row is `rate`. The unit rule (`selectUnitRate`: explicit
unit wins, else retail → highest slno, else base) runs FIRST to choose the unit, then the resolver
chooses the row within the unit. `preferBranchPriceRows` is subsumed by the resolver's ordering
and goes.

### 3.3 Outputs

Add to `ItemPriceLookupPayload`:

| key | value |
|---|---|
| `price_source` | `BUCKET` / `MASTER` |
| `price_scope` | `BRANCH` / `CHAIN` |
| `price_row_id` | the `ipm_id` that answered |
| `buckets[]` | **only for items whose policy tracks MRP or sale price**: every live bucket at this branch that has stock — `stock_balance` grouped by `(sbl_mrp, sbl_sale_price)` with `SUM(sbl_available_qty) > 0`, each with its resolved four prices. This is 16q Q24 without the F12 screen. Empty array for untracked items |

`max_price` keeps meaning "the MRP on the line": the bucket's MRP on a BUCKET answer, the headline's
`ipm_max_price` on a MASTER answer.

**Customer rates** (`sales.cust_item_rates.csr_unit_rate_id`) are per item and unit, not per MRP:
look them up by the **headline** row's id of the same unit (the `(-1, -1)` row at the narrowest
scope), never by the bucket row's id. Say so in the column comment; 0 rows exist today.

### 3.4 What the four Qt screens do with it (for the client side; recorded here so the contract is whole)

- A new line with no MRP typed calls the lookup as today. If `buckets[]` has one entry, the line takes
  that bucket's MRP and prices. If several, the line opens the bucket picker (the existing F12
  `lineStockRequested` hook) and re-calls with `mrp`. If none, it is the headline.
- Retyping the MRP under `sales.allow_mrp_edit` re-calls the lookup with `mrp` and re-prices. Today
  the retype changes only the savings figure; that is the defect this closes.
- Loaded lines (`sbiLotId` present) pass `lot_id`.
- Posting is unchanged: `sales-stock.service.ts` already carries `mrp: i.maxPrice` into the shadow
  voucher and writes the resolved lot back onto the bill line (`bill-lifecycle.service.ts:1179`).

---

## 4. Item entry — `POST /items/create`, the `prices[]` collection

The item card already shows one row per unit with an MRP column. After this migration it shows one
row per **unit × bucket**, which is the natural "this item at MRP 40, this item at MRP 45" view, and
menu 30 is its bulk form. Changes in `item-master-update.service.ts`:

1. **Natural key** for matching payload rows against existing rows becomes
   `ipm_company_id + ipm_branch_id + ipm_uc_unit_id + ipm_bucket_mrp + ipm_bucket_sp`
   (the key of `ex_ipm_overlap`, NULLs matching NULLs). Two payload rows with one key are refused,
   as now.
2. **The bucket columns are derived on the server, not sent by the client.** After the row's scope
   is resolved: under a policy with `track_mrp`, `ipm_bucket_mrp := NULLIF(ipm_max_price, 0)`; under
   `track_sale_price`, `ipm_bucket_sp := the price at sales.default_price_level` (the doctrine that
   `slt_sale_price` mirrors the default level); otherwise both NULL. Same `bucketKeyFor` as §2.
   The Qt price grid (`ItemPriceGridController::collectJson`) therefore changes nothing; a second
   MRP is simply a second grid row for the same unit, which the old unique index refused and the
   EXCLUDE accepts.
3. **Re-key on policy change.** When the item's effective tracking signature flips (own preset set
   or cleared, group changed — the case `item-group-change-warning` already warns about), every price
   row of the item is re-derived under the new policy in the same transaction. Two rows collapsing
   onto one key (MRP tracking switched OFF with two MRPs priced) is refused with both rows named —
   the same refusal the group-change check gives for dimensioned stock.
4. `DELETE /items-price-masters/delete` and `GET …/get` are untouched: they address rows by id.

`ItemsPriceMasterService.save(dto[], tx?)` keeps its signature; only the matching key and the
derivation step are new.

---

## 5. Menu 30 — `SellingPriceBulkModule` retargeted

The module's shape, DTOs, `resolveTargetScope`, the below-cost round trip and the exception filter
are all kept. What changes is confined to `stock-mrp-price.gateway.ts`, which now owns statements
against `inventory.item_price_master` (rename the class `PriceBucketGateway`; delete `isDeployed`
and `STOCK_MRP_PRICE_NOT_DEPLOYED`). Method by method:

| method | statement |
|---|---|
| `listPrices` (Q25) | `item_master ⋈ item_unit_conversion`, LEFT JOIN LATERAL the live buckets from `stock_balance` grouped by `(sbl_mrp, sbl_sale_price)` with stock ≠ 0, LEFT JOIN LATERAL the resolver over `item_price_master` for that key. One row per (item, unit, bucket-with-stock) **plus the headline row** for items with none. `priceSource` / `priceScope` / `bucketId` from the resolver |
| `listBuckets` (Q24) | the same lateral for one item, both dimensions grouped |
| `validateRows` (Q26) | unchanged in meaning: above-MRP (bucket rows), below-min, below-cost via `stock_item_cost`; the `ck_ipm_not_above_mrp` text in the filter map |
| `findBucketRowForUpdate` (S1) | `SELECT ipm_id FROM inventory.item_price_master WHERE ipm_company_id IS NOT DISTINCT FROM :company_scope AND ipm_branch_id IS NOT DISTINCT FROM :branch_scope AND ipm_item_id = :item AND ipm_uc_unit_id = :uom AND ipm_key_mrp = COALESCE(:mrp,-1) AND ipm_key_sp = COALESCE(:sp,-1) AND CURRENT_DATE BETWEEN ipm_effective_from AND ipm_effective_to AND ipm_is_deleted = false FOR UPDATE` |
| `updateBucketPrice` (S2) | `UPDATE … SET ipm_sales_price_a..d, ipm_price_*_wot, ipm_price_*_markup_perc, ipm_min_price, ipm_cost_price, ipm_cost_wot, ipm_profit_type, ipm_round_off, ipm_updated_on, ipm_updated_by WHERE ipm_id = :found` |
| `insertBucketPrice` (S3) | **copy the attribute columns from the headline row of the same unit at the narrowest scope** — `ipm_godown_id, ipm_addl_cess, ipm_disc_perc, ipm_disc_qty, ipm_loading_charge, ipm_freight_charge, ipm_loyalty_points, ipm_uom_remarks` — then set the bucket columns, `ipm_max_price := :mrp` (ck_ipm_bucket_mrp_is_max), and the prices. Without the copy a bucket-priced line loses its cess and loading charge, which is the exact fault the two-table design had |
| `listNoStock` (Q27) | rows among the touched ids whose bucket has no `stock_balance` row with stock at this branch |
| `findOpeningSeedBucket` (19q Q6) | the dearest bucket row at the most specific scope; **when there is none, the headline's `ipm_max_price`** — a better seed than the 0 the stub returns today |

**Deleted:** `isHeadlineRow`, the `ItemsPriceMasterService.save(rows, tx)` call in the save, and
`masterRowsSaved` in the response (keep the key, always 0, for one release if the DTO is frozen).
A row with neither dimension is a headline row **in the same table**, and S1-S3 handle it with
`(-1, -1)` — `ck_smp_identity` was the only reason the fan-out existed. The `scope` switch and
`assertScopeAllowed` (403 for CHAIN from a non-HQ user) stay exactly as built.

Two open items in the module README close here: §13.1 (the column list — it is this table's) and
§13.6 (Q23 is the §3 lookup, not a route of this module).

---

## 6. What is dropped, and where the reasoning goes

- `16_stock.sql` §20: the `CREATE TABLE stock.stock_mrp_price`, its indexes, the three
  `DROP FUNCTION fn_smp_effective` lines and the catalog comments come out. Leave a ten-line
  banner pointing here so the "why per MRP, why not per lot" argument is not lost (it is
  restated in §0 above). `16p_stock_pricing.sql.bak` is history and stays.
- `16q` Q23-Q27 / S1-S3 get a one-line note each: "moved to plan-nestjs-one-price-table.md §5,
  table is inventory.item_price_master". Do not rewrite them in the share; the repo's gateway is
  the live copy.
- `16_docgen/docreset.sh` no longer needs the `item_price_master` stub for §20.
- The `stock` schema keeps exactly one function. Nothing here adds one.

---

## 7. Tests

`test/one-price-table.e2e-spec.ts`, against a database built from the migration:

| # | case | expects |
|---|---|---|
| 1 | migration twice on `erp_dry` | idempotent; the 48 rows unchanged, all `(-1, -1)` |
| 2 | resolver: headline only | MASTER, CHAIN/BRANCH per row |
| 3 | resolver: bucket 40 at chain, bucket 40 at branch, headline | key 40 at branch → the branch row; key 40 at another branch → the chain row; key 45 → MASTER |
| 4 | resolver: bucket dated 2026-10-01..9999, today 2026-09-30 | MASTER today, BUCKET tomorrow |
| 5 | untracked item, lookup with `mrp=40` | MASTER — policy blanked the key |
| 6 | MRP-tracked item with two buckets in stock, lookup with no mrp | `buckets[]` has 2 with prices; `sales_price` is the headline's |
| 7 | lookup with `lot_id` | prices of that lot's bucket |
| 8 | `ex_ipm_overlap` | second row for one scope+unit+bucket → 23P01 → 409 |
| 9 | `ck_ipm_not_above_mrp` | bucket row priced above its MRP → 23514 → 422; a headline row above its MRP → accepted |
| 10 | composite save, MRP-tracked item, two rows unit=PCS MRP 40 and 45 | two rows, bucket columns derived, `ipm_max_price = ipm_bucket_mrp` |
| 11 | composite save, untracked item, two rows unit=PCS MRP 40 and 45 | refused (one key), both rows named |
| 12 | policy flip ON → OFF with two priced MRPs | refused, both rows named; flip OFF → ON re-keys the one row |
| 13 | S3 for a fresh bucket | `ipm_addl_cess`, `ipm_loading_charge`, `ipm_godown_id` copied from the headline |
| 14 | menu 30 save, `scope=BRANCH`, CHAIN-sourced row | branch override created, chain row untouched (the README table, all four rows) |
| 15 | opening seed | dearest bucket; headline MRP when none; 0 only when no row |
| 16 | the existing `selling-price-bulk.service.spec.ts` | passes with the fan-out cases deleted |

---

## 8. Phases

1. **Migration + resolver + lookup** (§1, §2, §3). Unblocks MRP-wise billing; every existing
   caller keeps working because no column was removed and every existing row is a headline.
2. **Item entry key + derivation + re-key** (§4). No Qt change.
3. **Menu 30 gateway fill** (§5), then the Qt screen from `change_selling_screen_plan.md` with its
   "untracked → ipm endpoint" paragraph ignored.
4. **Dated reprice** — columns live from phase 1, feature waits for a screen. Unchanged from §20.

---

## 9. Do not build

- A second price table, under any name. The resolver's single read is the point.
- Prices on `stock_lot` or `stock_balance` (§0).
- A client-sent `ipm_bucket_mrp`. The server derives it from the policy, or the item card and the
  lot can disagree about whether an MRP is a dimension.
- A `(NULL, NULL)`-refusing check. The headline row IS the `(NULL, NULL)` bucket.
- An "MRP editable on the bill re-prices" rule anywhere but the §3 lookup.

## 10. Open items

1. `HQ_USER_TYPES` for `assertScopeAllowed` is still a placeholder (module README §13.3).
2. The 5 live rows priced above their MRP: test data, not a work item; the constraint is bucket-only
   so they stand. Decide whether the item card should warn on a headline above MRP (3.0 blocked it,
   message 14, unless `skip_mrp`).
3. `ipm_godown_id` is an attribute today; whether a bucket row may carry a different godown from
   its headline is not decided. S3 copies it; the item card lets it be edited per row.
