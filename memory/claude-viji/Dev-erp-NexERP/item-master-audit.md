---
name: item-master-audit
description: 2026-09-25 item master audit. Open+save rewrote prices on 13 of 25 items (tax fetch recalculated on load), and the server nulls omitted base unit / packing ids. What was fixed, the harness modes, and what is still open (notes 50 + client decisions)
metadata:
  type: project
---

**Fixed in the client (item_entry.cpp, item_price_grid_controller.cpp):**
- `setTaxId(id, recalcRows)`: load and template pass false, so they only
  cache the rate. recalcAllRowsForTax on load had rewritten cost_wot and all
  sales prices (magic masala price A 1459 → 1303).
- `addBlankUnitConversionRow(false)` on load, so the stored to-base factors
  are no longer recomputed on open.
- The default barcode loads as row 0, because the save marks row 0 as default.
- Always sends `item_base_unit_id` (the grid's base row) and
  `item_packing_item_ids` (echoed from the load). The server NULLs both when
  they are omitted.
- The service tick is wired to `clicked`, not `toggled`, so the confirm no
  longer pops on load.
- An empty group reply no longer crashes, and a stale group reply is ignored.
- `m_saving` guards against a double save (duplicate items).
- The duplicate-barcode check, the barcode/reorder unit ∈ conversion units
  check, and reorder min ≤ max.
- Price and conversion rows send *_updated_by (and *_created_by on new rows),
  because the server stamps no actor there.

**Harness modes (tests/salebill/run-full.sh):**
- `itemdrift` is READ-ONLY: it loads 25 real items and diffs collectJson
  against GET. It now shows only the base-unit / default-barcode repairs.
- `itemroundtrip` seeds the never-sent fields, saves unchanged, and diffs
  every column (0 changes now).

**Still open:**
- Server: `share/inventory/notes-50-item-master-backend.md`, 8 items.
- Client, needs the user's decision: the unit-conversion convention (the math
  treats row 0 as the LARGEST unit, the validation text says smallest);
  fillEmptySiblingCosts overwriting sibling rows' cost on any cost edit (its
  guards are commented out); removing a non-base conversion row doesn't
  re-run the chain.
- The grids still use itemChanged plus QSignalBlocker, and the house rule
  (delegate cellEdited) is deferred ([[grid-delegate-edit-pattern]]).

Related: [[crud-blank-number-dropped]], [[verify-against-artefact]]

**User decisions (2026-09-25):**
- UNIT ORDER: biggest first, base LAST. validateForm now enforces
  findUnitConversionBaseRow() == last row, and the recompute math is
  unchanged. Evidence the user may want to revisit: 26 of 32 multi-unit items
  were keyed base-FIRST (fractional to-base, e.g. CHILLI POWDER 0.5 / 0.1),
  and none of those 26 can be saved until they are re-keyed.
- COST CASCADE: overwrite every sibling price row of the same company/branch
  (the commented-out guards are removed for good), then recalcRow(ProfitType)
  on each, so the sales prices follow the new cost.

**Re-check 2026-09-30:**
- **Notes 50 is still mostly open live.** 1 (header keys only; godown fixed), 2 (partly), 3, 4, 5 and 8 all reproduced on a throwaway `ZT-N50-PROBE` item.
- New server items are in `share/inventory/notes-67-item-master-recheck.md`:
  - B1: price and reorder rows are matched on unit + godown, with no company/branch;
  - B2: the tax name is joined from the old table, so it is always null;
  - B3: 25 legacy BY_PERCENT / BY_AMOUNT rows;
  - B4: a restore revives every child row.

**Client fixed the same day:**
- legacy profit types are normalised on load (they gave a 400, and the prices collapsed to cost);
- the Tax label is filled from `/tax-rates/get` via the `taxNameResolved` signal;
- a stale tax reply is dropped;
- the cost pull no longer cascades back (`m_pullingCost`);
- removing the base unit row promotes the LAST real row;
- the unit choices skip the blank row;
- the stored `ir_/ean_is_active` flags are kept;
- the photo is shown from the `item_photo` base64;
- the template refreshes the preset requirement;
- the close guard compares `collectJson()` to an `m_baseline`;
- a 404 on load closes the form;
- the list labels the toggle Restore and blocks Edit on a deleted row;
- validate refuses a NEW price row whose unit + godown collide.

**Harness:** `itemdrift` now also reports the Tax label (25/25) and puts legacy-type items first. `itemroundtrip` changes 0 of 151.

**Not directly tested:** the unit-row removal and the cost-pull guard (no offscreen unit-grid driver exists).

**Notes 67 ran and was verified live on 2026-09-30.** All of it is fixed except #8 (item policy "N" overrides the group; a design question, still open).

New route: `POST /items/restore?item_id=` (409 when the item is not deleted); `/items/delete` no longer toggles. The client list uses Restore in "Show Only Deleted", and the B1 client guard was removed. The `itemdrift` profit-type noise is gone.

**#8 decided by the user on 2026-09-30: an item FOLLOWS ITS GROUP unless it names its own preset.**
- Only a preset (NONE included) makes an ITEM policy.
- The server still derives an ITEM row from the item's flags; even negative stock OFF becomes BLOCK, and that row is written.
- The ask is `share/inventory/notes-68-item-follows-group.md`. The client already matches the rule (preset optional when the group has one).
- The box has 0 groups with a preset today, and 1 flag-derived ITEM row with no preset.

**Notes 68 verified on 2026-09-30:**
- The item side is correct (no preset = no ITEM row), and the data is cleaned.
- **But group rules never apply (notes 69):** `syncFromItemGroup` files the GROUP row under the LOGIN token's company (`access-token.guard.ts`). VKPOS's token is CHANDRA SAW MILL, not the working company Acme.
- Recommended fix: company NULL.
- Test recipe: `scratchpad n68.py` pattern. Create a ZT group with preset BE and an item with no preset; `/stock/opening/item-lookup` `trackSignature` should be BE.

**Notes 69 FIXED and verified on 2026-09-30:**
- GROUP rows now carry company NULL.
- An item with no preset resolves to its group's signature (BE); preset NONE resolves to N; clearing the preset goes back to BE.
- The item-follows-group rule is DONE end to end. Every item-master server note (50, 67, 68, 69) is closed.

**Audit 2026-10-21 — user said "do all three"; DONE same day (see end):**
- ui_table 17 (price grid) has 22 junk rows (ids 1918-1939) appended 10-12 by VIJAY via /ui-table-masters/create WITHOUT replaceColumns → duplicate clm_no; NexTable takes the later row → Uom/Godown/Cost/PriceA… columns HIDDEN, labels wrong; new price rows can't pick a unit. Fix = soft-delete 1918-1939 (needs user OK) + applyLayout should refuse duplicate clm_no.
- Clearing a dropdown never clears on the server: client sends "" (selectedId), server toNullableUuid("")→undefined = keep. Fix client: send null. item_company_id is @OptionalUuid → even null ignored → item can never go back to shared (server note needed).
- spinExpiryDays / spinIntimateBeforeDays have no maximum (QSpinBox 0..99): a 365 item is clamped to 99 on open and saved as 99 (5,674 box items >99).
- Others verified: middle unit removal leaves stale to_base; factor <1 accepted; group-change track race; RoundOff seeds dead (0.00 saved); below-cost/min check fires on blank levels.
- Server: child row id from another item is re-parented (price/ean/iuc/reorder services update by id with no item check); null on NOT NULL booleans → 500; no company scoping/actor spoof; reorder unique index lacks branch; EAN case-sensitive unique vs case-insensitive lookup; unit removal leaves its prices/EANs live.

**10-21 DONE:** ui_table 17 rows 1918-1939 soft-deleted via /ui-table-masters/column-delete (copy Shared/Prathap/inventory/ui_table17_rows_removed_2026-10-21.csv); NexTable::applyLayout keeps the FIRST row per clm_no. Client: idOrNull() on header ids + ipm_company/branch/godown + ir_branch/godown/unit (iuc_base_unit_id stays a string, NOT NULL col); expiry spins max 9999 + labels "Shelf life (days)"/"Alert before (days)" + checks; factor <1 refused on rows>0; middle unit removal re-runs the chain + price grid; group-race guard (m_trackedGroupId != group → refuse + refetch); new price rows round-off 0.01; blank levels skipped in below-cost/min checks; min>max, disc>100 refused, MRP<cost confirm; reorder number validator + reorder level between min/max; maxLength on texts; notes >250 refused (no silent cut); HSN digits ≤8. Tests: run-full.sh itemroundtrip 0 changed, NEW mode `itemnullclear` 3/3 (an `itemclear` mode already exists — min price). Server: Shared/Prathap/inventory/notes-137-item-master-backend.md.
NOT done (client): price key without MRP bucket, price/reorder row company scope, incl-tax ignored in price math, silent tax lookup failure, unit insert above row 0 / base flag on append, group-change tracking warning, preset dropdown filtered by session company, itemChanged→delegate pass, delegate hand-painted rows.
