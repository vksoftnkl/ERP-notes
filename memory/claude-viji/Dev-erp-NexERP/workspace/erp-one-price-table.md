---
name: erp-one-price-table
description: "ERP one-price-table (item_price_master = MRP-bucket table) built 2026-09-30; what's done, what's pending, and the gotchas hit"
metadata:
  node_type: memory
  type: project
  originSessionId: 6109bc63-7e3c-43d8-b369-81d2e4e0b065
  modified: 2026-09-30T12:29:07.230Z
---

`plan-nestjs-one-price-table.md` (copy in repo `plan/`, with an implementation-notes appendix) was built 2026-09-30, committed in 0626a3c7: migration `20260930160000_one_price_table` (applied to dev DB), pure resolver `Inventory/items-price-master/price-resolver.ts`, `PriceBucketService` (derive + rekey), lookup `mrp/sale_price/lot_id/doc_date` + `buckets[]`, menu 30 `PriceBucketGateway` (replaced `StockMrpPriceGateway`). Verified by `test/one-price-table.e2e-spec.ts` (16 cases, real opening stock, rolled back).

**Why:** buckets live in `item_price_master` (bucket cols NULL = headline); `stock.stock_mrp_price` is never created.

**How to apply:**
- Pending: plan §6 edits to the SHARE files (16_stock.sql §20, 16q, 19q, docreset.sh) — not done, awaiting the user's go-ahead (shared drive, no VCS). Group-preset edits (`syncFromItemGroup`) do not re-key items' price rows.
- F12 (`listBuckets`) lists every visible price ROW (headline first, per the share mockup) PLUS, since notes 74 (2026-10-01), every stock bucket at the branch with no row of its own, priced through `effectivePriceLateral` exactly as the grid (Q25) prices it. `/stock/price-bulk` takes an optional `itemId`. Since notes 75 (2026-10-01) an MRP-tracked item's MRP bucket is costed by THAT MRP's lots (inward sml_cost_value, weighted by on hand) via `mrpCostCte` + `costColumns` in price-bucket.gateway — grid, F12 and loadRowCosts (save) share it; rows carry `costBasis` MRP | ITEM | PRICE_ROW. Uses sml_cost_value, not sml_landed_value (the user said "landing cost" — open question). Notes 76 (2026-10-01): /stock/price-bulk also takes search / itemCategoryId / trackPresetId (effective: own else group's) / taxId / activeOnly (default true, OptionalQueryBoolean); grid + F12 rows carry barcode and costWot.
- Gotcha: `effectivePolicyLateral` aliases its inner table `p`; an outer alias `p` passed into it is shadowed ("column p.x does not exist").
- Readers that pick "first price row per unit" must filter to headline rows (`ipmBucketMrp: null, ipmBucketSp: null`).
- Notes 71 (live check) fixed 2026-09-30: menu 30 audit now one row per price row (before/after); lookup gives a provisional dearest bucket when an item has only bucket rows; price-master save fills omitted wot/markup.
- E2E suites must use the REAL AuditLogService (`new AuditLogService(db, ctx)` with `getIpAddress` on ctx): it refuses an `update` without originalRecord, and a mocked audit hid exactly that (every valid menu 30 save was a 400 on live).
Related: [[erp-a2-posting-services-state]], [[erp-server-http-testing-without-credentials]].
