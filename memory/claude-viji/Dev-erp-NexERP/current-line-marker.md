---
name: current-line-marker
description: NexTable::paintEvent draws the 3.0-style current line — whole-row tint + stronger current cell with a 2px border, from the theme's primary; every NexTable grid gets it
metadata:
  type: project
---
2026-10-01, user ask ("as in 3.0, simple and elegant"): the selected LINE must be obvious.
NexTable::paintEvent (src/shared/ui/table/nex_table.cpp) paints AFTER the delegates:
row = primary at alpha 30 (16 unfocused), current cell = primary alpha 40 + 2px rounded primary border
(1px, faded when the grid lacks focus); SelectRows grids get the row tint only. Colours DERIVE from the
`primary` token, so no new theme keys and no server change. currentCellChanged → viewport()->update().
Only NexTable grids — plain QTableWidget/QTableView screens (e.g. TxnMainView lists) don't have it.

**Why:** the cell-only selection gave no sense of which line you were on.
**How to apply:** don't add per-screen current-row painting in delegates; it is already drawn here. Related: [[theme-template-decision]].
