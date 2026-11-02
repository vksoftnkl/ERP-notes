---
name: erp-grid-enter-focus-chain
description: Entry-grid Enter chain comes from ui_tbl_clm_column_focus (not visibility); which grids flag what
metadata: 
  node_type: memory
  type: project
  originSessionId: 181d18e8-ba69-4471-bed7-8832156c2096
  modified: 2026-09-15T00:00:00.000Z
---

The quotation / sale-order entry grids walk Enter along `ui_tbl_clm_column_focus`
(`fixed.ui_table_columns`), NOT along visibility. Wired 2026-09-02 in
`features/sales/quotation/components/grid-focus.ts`; the flag reaches the DOM as
`data-quotation-focus-stop` via `GridCell`'s `focusStop` prop.

**Why:** the operator keys a handful of columns out of ninety. Visibility is the
wrong key — on grid 23 every one of the 90 columns is visible, so a
visibility-driven walk needs ~20 Enters to get from Description to Rate. The
`focus` flag is the Qt screen's tab-stop set and had been resolved into
`ResolvedColumn.focus` for a year while nothing read it.

**Row hop (fixed 2026-09-04).** `moveCellFocus` now walks a row at a time: when
the current row's chain is spent it steps into the next row and picks that row's
own landing spot — first flagged cell, else the cell marked
`data-quotation-lookup` (the picker), else its first editable cell. Before that
it was one flat list of flagged cells, so a layout that did NOT flag the lookup
column left Enter dead on the last line with data: every flagged column of the
trailing blank row is `disabled` until the row names an item, so there was no
next stop anywhere ahead. Local grid 23 happens to flag Description (the lookup),
which is why it only showed up on the live config.

**How to apply:**
- Flagged columns on the LOCAL db for 23 (Quotation-item): Description, Size,
  ExpiryDate, StockQty, GodownName, Uom, ToBaseFactor, OrderQty, Case Qty, Quote
  Qty, Length Qty, Net Qty, Sch, IsFree, Rate (a live db may differ — it is
  admin-editable per screen). 21 (CHARGES): Charge Name, Rate, Amount. 18 (Qt quotation): + NetQty.
  **24 (SO - ITEM): none**, which is why the walk falls back to every editable
  cell when a layout flags nothing — a chain of nothing would leave Enter dead.
- Flagging a read-only column (no `write` in `ITEM_COLUMN_MEANINGS`, e.g. Hsn)
  does nothing: it renders `disabled` and disabled cells are not stops.
- It is live-editable from each grid's right-click → Admin settings; the
  `saveQuotationColumnLayout` mutation patches the cached layout in place, so the
  chain changes with no reload (verified end to end).
- **Both pickers hand the chain back on the way out** (charges wired 2026-09-04,
  items before it): a pick leaves focus in a dismissed dialog, so each entry view
  keeps a `pickedItemRow` / `pickedChargeRow` ref and an effect that calls
  `focusNextStopFrom(grid, rowKey, lookupFieldKey(columns))` on the render the
  master landed in — the cells after the lookup are `disabled` until the row has
  one. On a FIXED charge that lands on Amount, since Rate comes back disabled.
- **A grid shortcut that reads the DRAFT must blur the focused cell first.**
  `GridCell` commits its buffer on blur/Enter only, so a figure the operator has
  typed but not left the cell on is not in Redux yet. Alt+R (copy row, added
  2026-09-15) calls `cell.blur()` before dispatching: the commit dispatch lands
  first and the copy carries it. Ctrl+± has the same latent hole. Alt+R then
  lands on the SAME column of the new row via `focusNextRowAfterRender(grid,
  sourceRowKey, fieldKey)` — the copy's key is the reducer's to mint, so the
  helper is told the row it was copied from and takes the one after it; entering
  on the row's first flagged stop would put focus on Description, where Enter
  re-opens the item picker on a row that already has its item.
- Hidden columns need no handling in the walker: the grids render only
  `column.visible`, so they are not in the DOM.

Related: [[erp-quotation-entry-port]], [[erp-sale-order-entry-port]],
[[erp-ui-table-layout-settings]], [[erp-client-ui-browser-verification]].

## Exception: Opening Balance (menu 55), 2026-09-24

That screen's two grids stop Enter at **every editable cell** and use
`ui_tbl_clm_column_focus` only to pick where the walk LANDS when it enters a row.
The flag subset is not applied there, and re-applying it would be a regression.

**Why:** every non-keyed column on those grids renders as a `<span>`, never a
form control, so there is nothing for the flag to run past — it can only
subtract. The live layouts flag ui_table 30 = Ledger + Opening (leaving Dr/Cr
unreachable, so a liability could not be set to Cr without the mouse) and
ui_table 31 = Invoice no ALONE (so Enter walked one column straight down the grid
and the date, days, side, amount and narration could not be keyed at all).

**How to apply:** before subsetting a walk by this flag on any screen, check how
many columns the layout actually flags — one is not a chain. Tests:
`features/accounts/opening-balance/components/grid-focus.test.ts`. See
[[erp-opening-balance-port]].
