---
name: erp-item-price-unit-id-is-iuc-id
description: "ipm_unit_id / ean_unit_id / ir_unit_id come back as iuc_id (conversion row), not unit_id — remap before rendering"
metadata: 
  node_type: memory
  type: project
  originSessionId: 5dbe0d3c-1cce-45d5-a65d-e7740473d838
  modified: 2026-07-18T06:28:42.843Z
---

In `/items/get`, the child rows' unit columns (`ipm_unit_id`, `ean_unit_id`, `ir_unit_id`) hold an **`iuc_id`** — a FK to `item_unit_conversion` — not a `unit_id`. The server surfaces the real unit id alongside as `ipm_unit_master_id` / `ean_unit_master_id` / `ir_unit_master_id`, plus `*_unit_name`.

Writes are lenient: `items-price-master.service` resolves `ipm_unit_id` by matching **either** `iucUnitId` or `iucId`, so a round-tripped value still saves.

**Why:** the client's linked-record editors match unit **ids**, so an unmapped `iuc_id` renders a blank Unit cell and makes the conversion-factor lookups (`itemUnitConversionsByUnitId.get(unitId)`) miss.

**How to apply:** run child rows through `mapLinkedRowUnitConversionIdsToUnitIds(rows, "<x>_unit_id", itemUnitConversionRows)` in `augmentItemDetailSource` before serializing. Reorders and EAN rows already did; price rows did not until 2026-07-18. Related: [[erp-item-company-token-override]].
