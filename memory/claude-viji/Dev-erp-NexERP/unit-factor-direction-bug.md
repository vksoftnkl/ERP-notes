---
name: unit-factor-direction-bug
description: 2026-10-23 — item unit to-base factor stored INVERTED by the server vs the item card (Samsung H1 Bottle 0.1 not 10); 16 items; notes-147 A NOT done; notes-148 = decision: ITEM CARD's rule
metadata:
  type: project
---

Found 2026-10-23 when the user said quick-rate showed Samsung H1 "Bottle cost ~13, should be ~138".

**Fact:** Samsung H1 units Bottle (slno 1, unit factor 1, default) / PCS (slno 2, unit factor 10, BASE). Client `ItemEntry::recalcAllUnitConversionFactors`: rows ABOVE base are bigger → Bottle to-base = 10 (1 Bottle = 10 PCS). Server `item-unit-conversion.service.ts` (~700–760, chain recomputed on save since notes-138): cumulative multiplies DOWN the list, to-base = rowCum/baseCum → Bottle **0.1** — that is what is stored and what stock/cost/billing use. OPN0006's 5 Bottle went in as 0.5 PCS. 16 of 34 multi-unit items on .106 have a factor < 1.

**Why it matters:** wrong base qty in stock and wrong per-unit cost everywhere a non-base unit is used. Not a quick-rate bug — the popup shows price-bulk's figures.

**How to apply:** don't "fix" display client-side. notes-147 A (`inventory/notes-147-unit-factor-direction-and-rate-cost.md`) asks Prathap to pick ONE direction, align the other side, re-derive factors + list affected stock rows. Wait for the answer before touching item card factor code; then re-run tests/itemcard. Related: [[item-base-unit-lock]], [[quick-rate-change-design]].

**10-23 later:** Prathap ran 147 B only; A unchanged on .106 (still 0.1, 16 items). Wrote `inventory/notes-148-unit-direction-decision.md`: ITEM CARD's rule wins (row above base = bigger; matches the user's expectation Bottle ≈ 138), server derivation must match the card row for row, data fix + list of stock rows keyed in flipped units (OPN0006) BEFORE any re-post.

**10-27:** server source now has the card rule (README "notes 147 A"), but OLD rows not re-derived: Samsung H1 Bottle 0.1, Sakthi Manjal BOX 0.1, SUGAR Kilogram 0.02 (rows above base <1); OPN0006 posted at factor 0.1. Asked in **notes-161 §1** (inventory/notes-161-stock-pending.md, the full stock pending list).

**10-27 CLOSED:** Prathap ran notes-161 §1 — factors re-derived live (Bottle 10, BOX 10, Kilogram 50; 0 rows above base <1); sales lines clean. OPN0006 CANCELLED by me via API (it was also in another company's godown "dharmapuri" / branch saw mill); user said leave it cancelled, no re-post. notes-161 §2 (notes-160 A/B/C) still not live; §3 notes-152 verified live.
