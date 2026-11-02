# Notes 148 — Unit factor direction: the DECISION for notes-147 A (2026-10-23)

notes-147 **B** (costPrice on `POST /stock/price-bulk`) is live and verified:
the Ctrl+R popup edits a price row's own cost, and the Price Change Log
records it. Thanks.

notes-147 **A** is not done yet. On .106 Samsung H1 still reads Bottle
`iuc_to_base_factor = 0.1`, its opening is still 5 Bottle = 0.5 PCS, and 16
items have a factor below 1. The note asked which direction to keep, so here
is the answer.

## Decision: the ITEM CARD's rule

A row ABOVE the base row is the BIGGER unit. Its to-base factor is the product
of the unit factors of the rows between it and the base.

| row | unit | unit factor | base | to-base (correct) |
|---|---|---|---|---|
| 1 | Bottle | 1  | | **10** (1 Bottle = 10 PCS) |
| 2 | PCS    | 10 | ✓ | 1 |

This is what users key and what they expect: the user reads Bottle cost
≈ 138 against PCS ≈ 13.8, i.e. a Bottle is ten PCS. It is also exactly what the
card computes (`ItemEntry::recalcAllUnitConversionFactors`, client):
- rows above the base: running ×= the unit factor of the row below;
- rows below the base: running ÷= the row's own unit factor.

## Ask

1. **Server derivation** (`item-unit-conversion.service.ts`, the chain recomputed on save, ~700–760): produce the card's numbers. A test: Bottle(1) / PCS(10, base) → Bottle 10, PCS 1; and PCS(1) / DZ(12, base) / BOX(50) → PCS 12, DZ 1, BOX 1/50 — whatever the card's own comment table shows, the server must match it row for row.
2. **Data fix for the 16 items:**
   - Re-derive `iuc_to_base_factor` under the rule.
   - Then list every stock row keyed in a unit whose factor changed: `stock_voucher_item.svi_to_base_factor / svi_base_qty`, `stock_ledger.sml_to_base_factor / sml_base_qty`, and balances. Those quantities were stored with the old factor.
   - For opening stock that's OPN0006 (Samsung H1, 5 Bottle). Tell us the list before re-posting anything, so we can decide per document (cancel and re-key, or a corrective adjustment).
3. Tell us when it's live. We'll re-run `tests/itemcard` and the opening stock suite against it.
