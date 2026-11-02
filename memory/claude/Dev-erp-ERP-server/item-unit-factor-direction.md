---
name: item-unit-factor-direction
description: "Item unit chain = the item card's rule (biggest unit first; row factor = how many of it make one of the row above; to-base = base cum / row cum). 99d0a468 had inverted it; restored 2026-10-23 (notes 147 A)"
metadata:
  node_type: memory
  type: project
  originSessionId: c349e680-a950-41b9-a156-1bc150064582
  modified: 2026-10-23T13:42:59.007Z
---

**The rule:** `normalizeItemUnitConversionFactors` (item-unit-conversion.service.ts) follows the Qt item
card (`ItemEntry::recalcAllUnitConversionFactors`).
- The list runs from the biggest unit down.
- A row's `iuc_unit_factor` is how many of IT make one of the row above.
- to-base = baseCumulative / rowCumulative. Bottle (1) / PCS (10, base) gives Bottle = 10.
- A row below the base gets a factor under 1.
- A step derived from to-base factors = previous / this.

**History:**
- 41e1e637 (2026-07-03) created the module with this rule.
- 99d0a468 (2026-07-27, titled "add item_inclusive_of_tax…") silently inverted both formulas.
- Restored on 2026-10-23 for notes 147 A ("Unit factor direction is inverted on save").

**Why:** what users key on the card and what the card shows. The inversion stored Samsung H1 Bottle = 0.1,
so opening OPN0006's 5 Bottle went into the ledger as 0.5 PCS.

**Data on .106 — FIXED 2026-10-27 (notes 161 §1.1)** by `plan/notes-161-unit-factor-fix.sql` (guarded, re-runnable;
second run = 0 rows). 28 rows: Samsung H1 Bottle / Sakthi Manjal Powder BOX / SUGAR Kilogram to-base 10/10/50;
11 BOX = 12 PCS test items reordered (BOX slno 1 uf 1, PCS slno 2 uf 12); 3 E2E-COLMAP base rows 6 → 1.
Afterwards all 406 live rows agree with the rule. No sales line ever used the bad rows.
OPN0006 (also holds BILL ITEM 2 @100) is to be cancelled + re-keyed by the USER from the Opening Stock screen
(their choice, 2026-10-27). Other sites ran the inverted code 07-27 → 10-23 too and were never checked.

**How to apply:** e2e payloads that send to-base factors round-trip under either order. Never send unit
factors in the reversed sense. "Factor < 1" is not itself wrong (rows below the base).
