# Notes 147 — Unit factor direction is inverted on save; cost on the quick rate save (2026-10-23)

Found while the user tested the Ctrl+R quick rate popup on **Samsung H1**. The
popup showed Bottle cost w/o tax ≈ 13.85 where the user expects ≈ 138.

## A. The item's units are stored the wrong way round (HIGH — wrong stock and cost)

**The item as keyed on the item card (menu 29):**

| row (slno) | unit | unit factor | base |
|---|---|---|---|
| 1 | Bottle | 1  | |
| 2 | PCS    | 10 | ✓ |

The user means **1 Bottle = 10 PCS**.

**The client's rule** (`ItemEntry::recalcAllUnitConversionFactors`): a row ABOVE
the base is the bigger unit; its to-base factor multiplies the factors of the
rows below it. So Bottle = 1 × 10 = **10**.

**What the server stores** (`item-unit-conversion.service.ts` ~700–760, the
chain recomputed on save): the cumulative multiplies DOWN the list from row 1,
and to-base = rowCumulative ÷ baseCumulative. So Bottle = 1 ÷ 10 = **0.1**:
1 Bottle = 0.1 PCS.

Live on .106 now:
```
Samsung H1  Bottle  iuc_to_base_factor 0.100000  iuc_unit_factor 1   (default unit)
            PCS     iuc_to_base_factor 1.000000  iuc_unit_factor 10  (base)
```

**Consequences (the server's 0.1 is what everything reads):**
- The 23-10 opening of **5 Bottle** went into the ledger as **0.5 PCS**
  (`stock_ledger` OPN0006: qty 5, base 0.5, factor 0.1, cost 1030 per Bottle).
- Price-bulk / the popup show Bottle cost = PCS cost × 0.1 (13.85 instead of
  ≈ 1,385 or whatever the true figure is). Billing a Bottle moves 0.1 PCS of stock.
- **16 items** on .106 have a to-base factor < 1 on some unit, out of 34 with
  more than one unit. Each one needs checking.

**Ask:**
1. Settle ONE direction, and use it on both sides. The item card's rule (rows above the base are bigger) is what users key and what the screen shows. If the server keeps "multiply down the list", the client must change instead. Tell us which, and we'll align the item card and its tests (`tests/itemcard`).
2. A one-time data fix for the existing items: re-derive `iuc_to_base_factor` under the agreed rule. List every stock / price row keyed in a unit whose factor flips (ledger base qty, `svi_base_qty`), because those quantities are wrong too. That's opening stock on Samsung H1 today; please list any others.
3. A test: Bottle(1) / PCS(10, base) must give Bottle = 10.

## B. Cost on `POST /stock/price-bulk` (the quick rate popup and menu 30)

The user wants **Cost** editable in the Ctrl+R popup, as on the 3.0 screen and
the item card. `SaveSellingPriceRowDto` has no cost field (`lineNo, itemId,
uomId, bucketId, priceScope, mrp, salePrice, levels, minPrice, roundOff`).

**Ask:** an optional `costPrice` on the row. It writes the price row's
`ipm_cost_price` and its `ipm_cost_wot` (= costPrice ÷ (1 + tax%)), in the same
transaction and the same writer (so the Price Change Log records it, source
CHANGE_SELLING_PRICE). Rules:
- It's the price row's own cost, not the stock's moving average (that keeps
  coming from inward, and is what `costRate` shows when the item has stock;
  `costBasis` says which).
- The response's row `costRate` should answer what the popup should now show.
- Absent = unchanged, as today.

Until it lands, the popup shows the cost read-only with a note saying why.
