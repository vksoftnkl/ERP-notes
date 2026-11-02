# Notes 75 — Change Selling Price: cost per MRP bucket, not the item average (2026-10-01)

**The user's ask:** "the cost showing is the average of all stocks, that's good — but for MRP-based
items the average should be based on that MRP only; then the user knows the landing cost, and only
then will they fix the price."

## What happens today

All three cost reads in `selling-price-bulk/price-bucket.gateway.ts` take
`stock.stock_item_cost.sic_avg_cost_rate × iuc_to_base_factor`, which is ONE moving average per
item per branch:

| Reader | Line | Used for |
|---|---|---|
| `gridStatement()` | ~916 | `costRate` on every grid row (`GET /stock/price-bulk`) |
| `listBuckets()` | ~375, both UNION branches | F12 bucket list |
| `loadRowCosts()` | ~524/525 | the save: BELOW_COST check, markup/margin recompute, and the `ipm_cost_price` / `ipm_cost_wot` it writes |

So every MRP row of an item shows the same blended figure. `sbl_stock_value` cannot help either,
because `applyItemCost()` stamps the item average onto every holding (the "distribution of
stock_item_cost" comment), so value ÷ qty per MRP gives back the same average.

**Live (Acme, Coimbatore):** MRP STOCK ITEM was opened as 10 @ MRP 40 at cost 25, and 20 @ MRP
550 at cost 500. Both rows show **341.67** today. Priced from that number, the MRP 40 stock looks
like a loss at any price under 341, and the MRP 550 stock looks like it has 60% margin when it has
10%.

`stock_lot.slt_cost_rate` is not the answer either: it keeps the FIRST inward cost. MRP ITEM's lot
says 200, while its two openings average 252.63.

## Ask

For a row whose **policy tracks MRP and whose bucket has an MRP** (`track_mrp AND mrp IS NOT
NULL`), cost = the inward cost of that MRP's lots in this branch, weighted by what is still on hand.
Every other row keeps today's expression unchanged. The non-MRP rows and the headline row stay on
the item average, which the user said is right.

Definition, per (company, branch, item, MRP):

1. For each lot with `slt_key_mrp = <bucket MRP>`, compute its inward average: Σ`sml_cost_value` ÷
   Σ(`sml_base_qty + sml_free_base_qty`) over `sml_direction = 1`, `sml_is_reversal = false`, not
   deleted, and not reversed (no live row with `sml_reverses_id = sml_id`), so a cancelled opening
   or purchase drops out. Do the same with `sml_cost_value_wot` for `costWot`.
2. Weight each lot by its current on-hand qty (Σ`sbl_on_hand_qty` over the branch's godowns,
   negatives as 0).
3. If nothing at that MRP is on hand, use the plain inward average of all those lots, so a sold-out
   MRP still shows what it cost.
4. If there is no inward at that MRP at all, fall back to today's expression.
5. Scale by `iuc_to_base_factor` as today.

Per-lot weighting means an item that tracks batch AND MRP gets the cost of the batches actually
on the shelf. With MRP-only tracking there is one lot per MRP, so this equals that MRP's inward
average.

Use `sml_cost_value`, the figure the moving average itself is built from, so a single-MRP item
reads exactly as today: MRP ITEM gives 252.63 both ways. If "landing cost" should include
freight etc., `sml_landed_value` sits next to it. Say which one you want; I took cost.

### SQL — verified read-only on 192.168.0.106, 2026-10-01, ~2 ms

```sql
LEFT JOIN LATERAL (
  WITH lots AS (
    SELECT sml.sml_lot_id AS lot_id,
           SUM(sml.sml_cost_value)                       AS in_value,
           SUM(sml.sml_cost_value_wot)                   AS in_value_wot,
           SUM(sml.sml_base_qty + sml.sml_free_base_qty) AS in_qty
      FROM stock.stock_ledger sml
      JOIN stock.stock_lot slt ON slt.slt_id = sml.sml_lot_id
     WHERE sml.sml_company_id  = :company
       AND sml.sml_branch_id   = :branch
       AND sml.sml_item_id     = :item
       AND slt.slt_key_mrp     = :mrp          -- the bucket's MRP
       AND sml.sml_direction   = 1
       AND sml.sml_is_reversal = false
       AND sml.sml_is_deleted  = false
       AND NOT EXISTS (SELECT 1 FROM stock.stock_ledger rev
                        WHERE rev.sml_reverses_id = sml.sml_id AND rev.sml_is_deleted = false)
     GROUP BY sml.sml_lot_id
  ), onhand AS (
    SELECT b.sbl_lot_id AS lot_id, GREATEST(SUM(b.sbl_on_hand_qty), 0) AS qty
      FROM stock.stock_balance b
     WHERE b.sbl_company_id = :company AND b.sbl_branch_id = :branch
       AND b.sbl_item_id = :item AND b.sbl_is_deleted = false
     GROUP BY b.sbl_lot_id
  )
  SELECT CASE WHEN SUM(oh.qty) > 0
              THEN SUM(oh.qty * l.in_value / NULLIF(l.in_qty, 0)) / SUM(oh.qty)
              ELSE SUM(l.in_value) / NULLIF(SUM(l.in_qty), 0) END     AS cost_rate,
         CASE WHEN SUM(oh.qty) > 0
              THEN SUM(oh.qty * l.in_value_wot / NULLIF(l.in_qty, 0)) / SUM(oh.qty)
              ELSE SUM(l.in_value_wot) / NULLIF(SUM(l.in_qty), 0) END AS cost_rate_wot
    FROM lots l LEFT JOIN onhand oh ON oh.lot_id = l.lot_id
) mc ON pol.track_mrp AND <bucket mrp> IS NOT NULL
```

and then the cost columns become:

```sql
COALESCE(NULLIF(mc.cost_rate, 0)       * iuc.iuc_to_base_factor,
         NULLIF(sic.sic_avg_cost_rate, 0) * iuc.iuc_to_base_factor, p.ipm_cost_price, 0) AS "costRate"
-- same shape for costWot with mc.cost_rate_wot / sic_avg_cost_rate_wot / ipm_cost_wot
```

Live result:

| Item | MRP | today (item avg) | per-MRP cost |
|---|---|---|---|
| MRP ITEM | 300 | 252.63 | 252.63 (one MRP — unchanged) |
| MRP STOCK ITEM | 40 | 341.67 | **25.00** |
| MRP STOCK ITEM | 550 | 341.67 | **500.00** |

The grid lists up to 1000 rows. Compute the lateral once per (item, MRP), as a CTE over the
`stock` CTE's MRP buckets joined back, rather than once per unit row. `ix_sml_item_date`
(company, branch, item, …) covers the ledger probe.

**All three readers must change together.** If the grid shows 25 but the save checks below-cost
against 341.67, a price of 30 shows green and is then refused (restrict) or asked about (warning),
and `ipm_cost_price` is written with the wrong figure.

### Small extra: tell the client which cost it is

Add `costBasis: 'MRP' | 'ITEM' | 'PRICE_ROW'` to the grid row and the F12 row. That is 'MRP' when
the per-MRP figure was used, 'ITEM' for the branch average, and 'PRICE_ROW' for the `ipm_cost_price`
fallback. The screen will then label the cell, for example "Cost @ MRP 40", so the user knows the
figure is for that MRP only.

---

## Verified 2026-10-01 — applied and live

- **Grid and F12:** MRP STOCK ITEM shows MRP 40 → 25 and MRP 550 → 500 (`costBasis` MRP). The headline row keeps ITEM 341.67. MRP ITEM @ 300 gives 252.63 (MRP), which is unchanged as expected.
- **Save path:** a price of 20 at MRP 40 → `needsConfirm`, "20 is below the cost 25". Nothing was written (price rows 1 → 1).
- **Client:** it reads `costBasis`. The Cost cell carries an "MRP" tag, and the row card says "Cost 25.00 (22.32 before tax) is the landing cost of the stock at MRP 40.00 only".
- **Not sent:** `costWot` on the grid row. The client derives the new "Cost B.Tax" column as cost ÷ (1 + tax%), which is exact unless the item has cess. If you add `costWot` to the grid/F12 row, the client will switch to it.
