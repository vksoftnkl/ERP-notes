---
name: grid-delegate-edit-pattern
description: "Grids notify edits via the delegate's cellEdited, not the table's itemChanged — converted for sales/quotation/charges, inventory still pending"
metadata: 
  node_type: memory
  type: project
  originSessionId: 02bb4c26-2d49-4279-9c1f-475e2e47df12
  modified: 2026-08-04T05:22:35.290Z
---

Two grid wirings existed side by side. The correct one: the delegate overrides
`setModelData()` and emits `cellEdited(row, column)`, the host listens on that,
so programmatic `setCellValue()` notifies nobody. Converted as of 2026-08-04:
`SalesItemDelegate` (was already right) and `ChargeGridDelegate` +
`ChargeGridController` (rewired off `QTableWidget::itemChanged`, 8 blockers
deleted).

**Why:** `itemChanged` can't tell a user commit from the hundreds of cells
`distribute()` writes on every recompute, so every write path had to wrap itself
in a QSignalBlocker. See [[widget-signal-wiring-rule]].

**How to apply:** user decided on 2026-08-04 that sales + quotation are done and
the rest is a **later, separate pass** — don't fold it into unrelated work.
Still on `itemChanged`/`cellChanged`, ~52 blockers between them:
`item_entry.cpp` (12, three grids), `item_price_grid_controller.cpp` (9),
`item_qty_price_entry.cpp` (5), `nex_bank_grid.cpp` (3),
`sale_bill_tender.cpp`. Same fix each time.
