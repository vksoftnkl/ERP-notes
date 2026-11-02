---
name: item-default-barcode
description: Item card Barcode field = the item's DEFAULT EAN row (base unit); grid = extra barcodes only; item_default_barcode is a display copy; billing scans only the EAN table
metadata:
  type: project
---

Decided 2026-10-22 (user: "all ean code should be in ean table, item_default_barcode is just a temp field to show"; unit = BASE unit; old items get in "only as items are saved" — NO server backfill).

- **Why it mattered:** the sale bill scan (`/master-lookups/item-by-barcode?barcode=&company_id=`) reads ONLY `inventory.item_ean_codes`. Live 10-22: 10,002 items had a barcode only in `item_master.item_default_barcode`, none in the EAN table → unscannable. Only 10 items had EAN rows (4 with default ≠ the copy). `/item-price-details/get-by-barcode` reads item_default_barcode (other route).
- **Client (ItemEntry):** load → the `ean_is_default` row goes into txtItemDefaultBarcode (m_defaultEanId / m_defaultEanUnitId), not the grid; no default row → field keeps item_default_barcode (and a grid row with that same code is taken as the default). Save (collectJson) → field = default row sl 1 (same ean_id = update in place; new = base unit via defaultEanUnitId(), stored unit kept while still on the item), grid rows = extras; field empty → grid row 0 default; `item_default_barcode` = copy of the default code. Validation: field code repeated in the grid refused.
- EAN column is `ean_uc_unit_id`; the API key `ean_unit_id` takes a unit-master id.
- Tests: tests/itemcard SRC=test_item_default_barcode (read-only payloads, 14); live round trip scratchpad n138/checkean.py (ZT-EAN item, create/scan/edit/clear 7/7, deleted).
