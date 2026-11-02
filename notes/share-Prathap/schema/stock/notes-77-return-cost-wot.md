# Notes 77 — stock posting: returns write cost_rate_wot = the tax-inclusive rate (2026-10-01)

Found while verifying notes 76: ZT-ITEM-A (18% tax) shows `costRate` 79.29 but `costWot` **7.90**.
It should be about 67.19. Notes 76 reports the figure correctly; the figure itself is wrong in
`stock.stock_item_cost.sic_avg_cost_rate_wot`, because of what the engine wrote into the ledger.

## The defect (structural; it would happen the same way in production)

Inward ledger rows by source, in all history on the box:

| sml_src_doc_type | inward rows | `cost_rate_wot = 0` while cost > 0 | `cost_rate_wot = cost_rate` (tax not taken off) |
|---|---|---|---|
| SALE_RETURN | 119 | 0 | **119 (all of them)** |
| DC_RETURN | 16 | 0 | **16 (all of them)** |
| PHYSICAL | 10 | 1 | 6 |
| OPENING | 150 | 18 (older) | 111 (older) |

**Openings posted TODAY are correct:** 25 → 22.32 at 12%, 300 → 254.24 at 18%, 500 → 446.43 at 12%.
So the opening path has been fixed since. **The sale-return and DC-return paths have not.** The
latest SALE_RETURN (2026-09-30 17:54) still writes wot = 20 on a cost of 20.

ZT-ITEM-A in Coimbatore shows the effect. Its opening was posted 500 @ 80 with wot 0 (old path), and
then each sale return added 1 @ 80 with wot 80. The wot average is therefore 3,940.95 / 499 = 7.90.
Every later outward row inherits it (sale bill wot 0.16 … 1.43, adjustment wot 7.62).

## Ask

1. **SALE_RETURN / DC_RETURN inward:** `sml_cost_rate_wot` (and `_value_wot`) must be the
   tax-exclusive figure. These returns come back at the item's current average, so use the
   average's own wot (`sic_avg_cost_rate_wot`), or cost ÷ (1 + tax%) of the line's rate. Do not
   copy the inclusive rate.
2. **PHYSICAL gain:** 6 of 10 rows have wot = cost. Same fix.
3. **Repair (one-off SQL):**
   - Recompute `sml_cost_rate_wot` / `sml_cost_value_wot` on the bad inward rows from the item's
     tax rate as at the doc date.
   - Recompute `stock_item_cost.*_wot` and `stock_balance.sbl_*_wot` for the affected
     (company, branch, item) by replaying the ledger.
   - The OPENING rows with wot 0 or wot = cost from before the fix need the same treatment.
   - Send it to me first and I will dry-run it on erp_dry.

## Why it matters to the user

The Change Selling Price screen shows **Cost B.Tax** so the user can price from the landing cost
before tax, and margin % is computed on the tax-exclusive pair. With 7.90 for an item that costs
67.19 before tax, every margin on that item is wildly wrong. Until this is fixed, the screen
derives Cost B.Tax as cost ÷ (1 + tax%). It uses the server's `costWot` only for an item with cess,
where that derivation is wrong anyway.

---

## Re-check 2026-10-01 evening

- **Asks 1 + 2 (code) are IN:**
  - `sales-stock.service.ts costRateWot()` writes the line cost net of its tax, for sale and DC returns.
  - `stock-voucher-posting.helper.ts` derives a PHYSICAL gain's wot from the item's tax rate, and no longer mixes the wot of one source with the cost of another.
  - Not exercised live yet: there has been no new return since.
- **Ask 3 (repair) is NOT done.** No script is in the share or in prisma.
  - The ledger still has 119 SALE_RETURN, 16 DC_RETURN, 6 PHYSICAL and 111 OPENING inward rows with wot = cost, plus 18 OPENING rows with wot 0.
  - 127 of the 146 `stock_item_cost` rows have an implausible wot average (0, ≥ the cost, or cost/wot > 1.30). ZT-ITEM-A is still 79.29 / 7.90.
  - Every outward row posted since inherits it, and so do the stock value reports on the without-tax side.
