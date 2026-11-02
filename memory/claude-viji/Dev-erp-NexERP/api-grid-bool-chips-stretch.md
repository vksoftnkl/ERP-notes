---
name: api-grid-bool-chips-stretch
description: "NexApiTableView (2026-10-02): JSON booleans show as Active/Inactive or Yes/No pills; the last visible column fills to the edge; grid widths are % and must sum ≤100"
metadata:
  type: project
---

Asked by the user on 2026-10-02 for every list grid (NexApiTableView, used by NexCrudMasterPage, the popups and TxnMainView):
- **Booleans:** `NexJsonModel::formattedValue` turns a JSON bool into a word. A heading containing "active" or "status" gives Active/Inactive; any other heading gives Yes/No. EditRole and `cellText()` stay raw `true`/`false`, so screen code that tests "true" is unaffected.
  `NexGridCellDelegate` (src/shared/ui/grid) paints those as pills in the success/danger theme tokens. It is the view's default delegate. Screens that set their own delegate (TxnRowDelegate, cheque registers) get the word but no pill.
- **Stretch:** `applyColumnsToView` gives the last visible column the remaining space up to `viewport()->width()`. It does not use effectiveTableWidth(), which keeps the frame, and those 2px brought up a scrollbar.
- **Resize guard:** `m_applyingWidths` makes our own setColumnWidth not count as a user drag. Before this, the first apply set m_userResizedColumns and window resizes never reflowed.
- **grid_column_width is a PERCENT.** Keep the visible columns' total ≤100, or the grid scrolls sideways and nothing stretches. Old grid 52 summed to 251. It is now 91, set through PUT /grid-details/column-width.
Render-checked offscreen per [[qt-offscreen-test-harness]]. Related: [[delegate-chip-background]], [[company-main-branch]].

**Width sweep 2026-10-02:** the in-use grids over 100% (53, 56, 87, 113, 114, 118, 121) were scaled to about 97%. Grid 87 was set by hand (Date had been 58).
The backup is `share/admin/grid_widths_before_2026-10-02.json`. Left alone: the lower-case legacy grids (1–33, 84) and the grids with NULL widths (30/41/44/46/60/80). The client opens none of them. The `k*GridId` constants 14–33 in entries are ui_table ids for NexTable, not grid_details.

**Main-grid sweep, 2026-10-02 — SAVED (the user ran apply.py --save) + verified: all 44 load via /run with the app params, and the fixes show correct data:** the plan covers 44 main list grids. It trims dropped/hidden columns nothing reads, normalises widths to 97%, and fixes these bugs:
- 66 and 75: a missing comma.
- 53: the parent join pointed at the row itself.
- 65/73/74/76/77: the SQL ignores the show-deleted param the screen sends (74 also ignores its branch filter).
The script is spec.py + apply.py in that session's scratchpad (grids/). All 44 passed the read-only check. Auto mode blocked my own save twice; bulk grid writes need the user to run them. A validation false alarm: the token heuristic picks up SQL words that start with i (e.g. ISSUE), so re-test with the exact app params. The backup of every grid is `share/admin/main_grids_before_2026-10-02.json`.

**Grid 67 (Item list) widened 2026-10-02:** the sweep had only changed its widths. It now also shows Code, Item Name, Regional Name, Barcode, Group, Brand, Unit, HSN, GST and Active (joins to group/brand/unit/tax_rate_master). Name, code, regional name, barcode, group, brand and HSN are searchable. Checked via /run. A single-grid save through apply.py <id> --save was allowed.

**Bool centring + full check, 2026-10-02:** check.py in that scratchpad runs every grid the app opens (71, main lists + popups) through /run and psql \gdesc. All load and every visible column comes back. Every visible BOOLEAN column is aligned Center (13 grids changed, SQL untouched). Backup: share/admin/all_grids_before_center_2026-10-02.json. Grids 110/112 need a real id to test (isaa_ledger_id / iabj_bill_id).

**Enter opens the row (2026-10-02):** `NexApiTableView::keyPressEvent` handles plain or keypad Enter on the grid by emitting `activated(current)` and then `enterPressed()`. Before this only the search box sent it. Every list wires enterPressed to Edit/open: NexCrudMasterPage onEdit, TxnMainView triggerEdit (pick in Pick mode), and the cheque registers. activated is kept for NexPopupWindow and VoucherExceptionsDialog. Ctrl+Enter is not taken, so TxnMainView's read-only View shortcut still works. PrintTemplateList moved from itemDoubleClicked to activated (Enter or double-click).

**Text pills (2026-10-14):** `NexGridCellDelegate::setBadgeColumns(QSet<jsonKey>, overrides)` draws named TEXT columns as pills (TxnBadgePalette::styleFor, or the override keyed by upper-cased text); matches by `view->colKey(col)`, so layout order doesn't matter. Used on Business Day 274 grid 150 (treatment, status_text). Shared map `TxnDocTypes::tillSessionBadges()` (also 273's badgeOverrides): WITHIN TOL. green, PENDING amber, EXPENSE blue, SUSPENSE slate, RECOVER red. Check: `SNAP_SRC=snap_day SNAP_DATE=2026-10-12 tests/till/snap.sh` → build/…/snap_day_grid.png.

**Every grid fills to the edge (2026-10-14, user: "stretch last column to end in all grids"):** `GridStretchGuard` (src/shared/ui/grid/grid_stretch_guard.*, installed in main.cpp after ThemeManager) — app event filter on the horizontal QHeaderView's **Show** (NOT Polish: polish fires mid-construction, before modes/property are set) of any QTableView: if no Stretch section and stretchLastSection off → turn it on. Opt out: property `nexKeepWidths` on view/header (NexApiTableView sets it; it fills itself). Middle-Stretch screens (user menus, module admin, tender, party adjust, audit panel, ui-table setting) untouched. Tax-rate ledger grid now stretchLast true. Offscreen check 4/4 in that session's scratchpad (stretch/t.cpp).
