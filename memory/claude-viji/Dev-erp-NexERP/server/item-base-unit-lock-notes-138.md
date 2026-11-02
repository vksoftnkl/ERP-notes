---
name: item-base-unit-lock-notes-138
description: "Notes 138 built 2026-10-22 — a saved item's base unit is locked (409 ITEM_BASE_UNIT_LOCKED on /items and /item-unit-conversions), header derived from the base row; A3 found sales posting falls back to the CURRENT iuc factor when a line's factor is 0"
metadata:
  node_type: memory
  type: project
  originSessionId: 56dfa405-c57c-4e8d-9354-9f02fed703fc
  modified: 2026-10-22T04:48:54.643Z
---

User's rule (2026-10-22): once an item has a live base unit_conversion row its base unit never
changes, stock or no stock (offline tills make "has stock" unknowable). Factors, weight, remarks,
default/big flags, adding/removing non-base rows stay open. No unit rows = not locked.

Built: `src/modules/Inventory/utils/item-base-unit.helper.ts` (`baseUnitChange` / `refuseBaseUnitChange`),
called from `ItemMasterUpdateService.syncUnitConversions` (payload-indexed field, plus "base row left
out" → field `unit_conversions`), `ItemUnitConversionService.save` (before normalisation; one
findMany over the batch) and `.delete` (base row), and `ItemsMasterService.updateItem` (header =
base row's unit on every update; a different non-null value 409s). Same code string in
ItemErrorCode and ItemUnitConversionErrorCode on purpose. Migration `20261022100000` backfilled 322
NULL headers. Proof: `test/items-master-notes-138.e2e-spec.ts` (7, rolled back). No
Error_Messages.json row — no ITEM_* code has one.

**A3 answer (not fixed, reported):** `sales/posting/sales-stock.service.ts` `writeShadow` uses the
line's `*_to_base_factor` only when > 0, else the item's CURRENT `iuc_to_base_factor`; every sales
factor column defaults 0 and the Qt bill lines on the box carried 0 (all base-unit lines then).
Sale-order reservations (`stock-reservation.service.ts`) always use the current factor; the opening
CSV import derives factor at save. Stock engine/reversal/rebuild use stored svi/sml values only.

**Why:** a late-synced offline sale must count in the unit and factor it was made with.
**How to apply:** never let a route bypass the helper; a fix for A3 needs Qt to send the factor
first, then the server can refuse 0 on non-base lines. Related: [[item-master-notes-137]],
[[cloud-sync-is-local-only]].
