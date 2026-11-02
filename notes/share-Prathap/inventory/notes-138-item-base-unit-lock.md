# Notes 138 — Item base unit: lock it once the item is saved (2026-10-22)

Read from the nexerp-api-ro snapshot of 2026-10-17 and the live catalog on
192.168.0.106. Paths are under `src/modules/Inventory/` unless they start
with `src/`.

## The rule (decided with the user today)

Stock is counted in the item's **base unit**: `stock_ledger.sml_base_qty`,
`sml_free_base_qty`. So:

- **Once an item is saved, its base unit never changes.** That holds whether
  or not the item has stock.
- **Unit factors MAY change.** Every txn line stores its own factor
  (`sbi_to_base_factor`, `sdi_`, `sdri_`, `sri_`, `soi_`, `sqi_to_base_factor`),
  so a new factor only affects bills made after it. Old bills keep theirs.
- **New units may be added** to a saved item, above the base row.
- A wrong base unit is fixed by making the item inactive and creating a new
  item. A head-office "change base unit" tool that converts history may come
  later. It is not asked for here.

### Why not "block only when the item has stock"

That check can't be trusted once tills run offline:

1. A till is offline and sells 5 PCS of item X. Its stock row is not on the
   server yet.
2. At head office, the server sees no stock for X and allows a base change
   from PCS to BOX.
3. The till syncs. Its `sml_base_qty = 5` now reads as 5 BOX.

Nobody gets an error and the stock is wrong. "Saved once = locked" depends only
on the item row, so it gives the same answer on every device at every moment.
It's the same reasoning as the offline-sync invariants note
(share/sales/offline-sync-invariants.md).

The Qt item card already does this on its side, as of today (details at the end).
**The server still accepts a change**, from any client.

---

## A. Asks

### A1. Refuse a base unit change on update

Today a changed base unit is written straight through:

- `items-master/items-master.service.ts:980-982`: `item_base_unit_id` on
  update is stored as sent.
- `items-master/item-master-update.service.ts:197-238` (`syncUnitConversions`):
  - A different row may arrive marked `iuc_is_base_unit`.
  - The old base row may simply be left out. It's then soft-deleted as stale
    (lines 225-229). The comment there even names this case: "releasing
    removed rows (e.g. an old base unit)".
- `item-unit-conversion/item-unit-conversion.service.ts:276-298` (update by
  `iuc_id`, also reachable through `/item-unit-conversions/create` with an id)
  takes a new `iuc_base_unit_id` / `iuc_is_base_unit` / `iuc_unit_id` on the
  base row.

**Ask:**

- Once an item has a base row, refuse any save whose result would change the
  base unit, with 409 `ITEM_BASE_UNIT_LOCKED` naming the field. That covers:
  - a different row marked base, or the base row's `iuc_unit_id` changed
  - the base row left out of `unit_conversions` (it would be soft-deleted)
  - `item_base_unit_id` sent with a different unit
  - the same changes through the `/item-unit-conversions` routes
- Base the check on what is stored, not on stock, for the reason above.
- An item with **no** unit rows yet is not locked: its first save sets the base.

**Still allowed:**

- `iuc_unit_factor` and `iuc_to_base_factor` on any row
- weight, remarks, default and big-unit flags
- adding new non-base rows
- removing non-base rows (notes 137 A4 still applies to their dependants)

### A2. `item_base_unit_id` should come from the base row, not the client

Live today:

| | count |
|---|---|
| items (not deleted) | 10,359 |
| items with a base unit row | 364 |
| of those, `item_base_unit_id` **NULL** while the base row names a unit | **322** |
| items with more than one base row | 0 |
| items with unit rows but no base row | 0 |

All six items that have `stock_ledger` rows are among the 322 (item `001` has
a Bottle base row and a NULL header; the other five are e2e items). The
header field is a copy that the create and update paths take from the payload
(`items-master.service.ts:763`, `:980-982`). The Qt card sends it, other
clients don't. (`inheritClassDefaults`, `:856-911`, fills it from the
group/category default unit, but only when NO unit rows are sent. It never
copies the base row.) So any reader that uses `item_base_unit_id` gets NULL for most
items (e.g. `items-master.service.ts:471-478` falls back to it).

**Ask:**

- On every create and update, set `item_base_unit_id` from the base
  conversion row, and ignore whatever the client sends (or 400 if it differs).
- Backfill the 322 rows from their base row once.

These rows are box data, so they're not urgent in themselves. The ask is the
derivation, so the copy can't drift again.

### A3. Please confirm: a synced txn line uses its OWN factor

The lock above is only safe if posting never re-reads the item's current
factor for an old line. Please confirm that when an offline till's bill syncs
late, the stock row's `sml_base_qty` is `qty × <line>_to_base_factor` as
stored on the line, not the factor the item has now. If it re-derives from
`item_unit_conversion`, a factor change between the sale and the sync
re-counts that sale.

---

## B. What the Qt card does now (for reference)

Item card (menu 34), `src/modules/inventory/master/items/item_entry.cpp`:

- **New item:** the base tick follows the LAST unit row by itself (units run
  biggest first, base last). The tick can't be clicked.
- **Saved item:** the base row is locked:
  - the tick can't be clicked
  - the base row's unit can't be re-picked
  - the base row can't be removed
  - Save refuses a base unit different from the one loaded
- A new unit on a saved item is inserted just above the base row, and the
  operator types its factor.
- Factors stay editable.
- Tests: `tests/itemcard/run.sh` with `SRC=test_item_base_unit`, 28 checks,
  read-only.

So the Qt card never sends a base change. A1 is for every other client and
for direct API calls.
