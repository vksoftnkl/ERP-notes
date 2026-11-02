---
name: nextable-grid-traps
description: "Three NexTable traps that make a new child grid look dead, all hit while building the GST rate Ledgers tab"
metadata:
  type: feedback
---

Every one of these produced the same user report — *"I can't choose anything,
nothing happens"* — and none of them is visible from reading the code.
Confirmed by running the grid under the offscreen harness
([[qt-offscreen-test-harness]]), which printed the cause each time.

**1. `loadLayout()` hides EVERY column immediately** and only un-hides what the
server names back. A grid with no local fallback is blank and inert whenever its
`ui_table` row is unreachable — API down at launch, stale token, row never
inserted. Seed titles/hidden/widths locally in the ctor AND again on
`NexTable::layoutLoaded`, so a partial server layout cannot blank the rest.
`promotion_scheme_entry.cpp::seedLocalLayout()` is the reference.

**2. `editItem()` right after `insertRow()` is refused** — the view has not laid
the row out yet inside that handler. Qt prints `edit: editing failed` and the new
row looks dead. Defer with `QTimer::singleShot(0, ...)`. Verified: immediate call
fails, deferred call opens the editor (`focusWidget: QComboBox`).

**3. A delegate returning `nullptr` from `createEditor` IS a silent no-op.**
Cells that depend on another cell (a ledger picker filtered by the row's role)
must say why they are not editable — grey text in the cell, a line in the status
label, and move the operator to the cell they need — not just refuse.

**Why:** all three fail silently and identically, so the user cannot tell them
apart and neither can a code read.

**How to apply:** when adding any new NexTable child grid, do the local seed,
defer the first edit, and never return a bare nullptr from `createEditor`. Then
run it offscreen and print columnCount / hidden / `createEditor` per column
before handing it over.

Also: `ui_table` ids are claimed in client constants BEFORE being registered
(`promotion_scheme_entry.h` held 26-29 as unregistered TODOs). Grep for
`loadLayout(` across `src/` before registering a new id — I collided with 29 and
had to move that grid to 30.

**A FOURTH trap, found 2026-09-16 on the receipt (menu 99) — the row-number
gutter comes BACK when the layout arrives.** `NexTable::loadLayout()` snapshots
`m_verticalHeaderShown = !verticalHeader()->isHidden()` at the moment it is
called, and `applyLayout()` restores that snapshot when the server answers. So
`verticalHeader()->setVisible(false)` AFTER `loadLayout()` works only until the
layout lands — then a gutter nobody asked for reappears, and because it eats
width the columns overflow into a horizontal scrollbar. **Hide it before
`loadLayout()`** (or in the `.ui`, which is the Designer setting the snapshot
is actually reading). Invisible until the ui_table row is registered, which is
why the receipt looked right for a day.

**And a fifth, same day:** `DisplayFormat::Currency` formats
UNCONDITIONALLY — an empty QVariant paints as "0.00". Only `Number` blanks a
zero. Forty rows of 0.00 hide the three figures that matter, so a screen that
wants blank cells must write an empty `QTableWidgetItem` itself; do NOT change
the shared behaviour, because a 0.00 rate on a sale-bill line is a fact, not
noise.

**Width percentages have a floor.** `minimumSectionSize(48)` wins over a stored
percentage, so a column registered below ~3.3% (of a ~1470px grid) is widened
anyway and the surplus becomes a scrollbar. Registered receipt "#" at 3.03%
did exactly that; 3.80% fixed it.

**A SIXTH trap (2026-09-16, the receipt's F8 list).** `NexApiTableView` sizes
its columns as a PERCENTAGE of its own width and only re-applies them in
`resizeEvent`. Call `setup()` on a `QStackedWidget` page that has never been
shown and the width is meaningless: every column comes out a sliver with the
rest of the view blank beside it, and no resize follows to correct it. **Defer
`setup()` to the first time the page is shown** — which also saves a grid fetch
for an operator who never opens the list. `applyColumnsToView()` is private, so
there is no way to re-apply it by hand afterwards.

And the widths themselves must sum to **100 over the VISIBLE columns**. Grid
108 shipped at 104 and clipped its Status badge off the right edge; ui_table 32
had to be re-registered at exactly 100 when a column was added. Fix widths via
`PUT /grid-details/column-width` (`{columns:[{grid_column_id, grid_column_width}]}`)
or by re-posting the ui_table with `replaceColumns: true`.


## A fourth way a grid looks dead: an async list behind a QComboBox editor

Found 2026-09-18 on the receipt's Bank column, reported by the user as "cant
choose bank name".

The delegate built a `QComboBox` and filled it from a **static QStringList
fetched asynchronously**, the fetch kicked off lazily by the first caller. So
the FIRST editor was constructed from a list that was still empty and had
nothing in it — the operator opened the cell and there was no option to pick.
Nothing errored; the combo was simply blank.

**Rule: never fill a cell editor from a list that is fetched on demand.** Use a
`NexDropdownSingle` bound to the dropdown id instead — it fetches when the
POPUP opens, so the list arrives because the field was opened, rather than the
field being built and hoping the list is already there.

Two mechanics that go with it:
- the editor subclass lives in the .cpp, so moc never sees it → recognise it
  with **`dynamic_cast`, not `qobject_cast`** (which static-asserts on a type
  with no Q_OBJECT).
- read the typed-but-not-chosen text by subclassing and exposing
  `inputField()` (protected on NexDropdown). A column backed by a varchar
  rather than an FK must still save a name that is not on the list.

## Registering a NEW column takes the POST twice

`/ui-table-masters/create` **ignores `uiTblClmColumnVisibility` for a column it
is creating** and only honours it on a later update. Four columns added to
ui_table 32 came back `visibility=false` despite carrying a width, and the grid
hid them. Posting the identical body a second time fixed all four. Register,
then register again, then read it back.

Also: the column keys are `uiTblClmColumnVisibility` / `uiTblClmColumnFocus` /
`uiTblClmColumnNecessity` (NOT `...Visibility`/`...Focus`/`...Necessary`), and
`uiTblClmNo` is a STRING. Wrong names are a 400 per column.


## A grid with SQL and NO columns: `replace_columns` on an UPDATE

2026-09-18, registering grid 112. `/grid-details/create` with `replace_columns:
true` and an existing `grid_id` **soft-deletes every column of the grid AFTER
inserting the new ones**, so it takes its own rows with it. Four posts, four
201s, no errors, and a grid that renders nothing:

    SELECT count(*) FROM fixed.grid_columns WHERE grid_id=112;                -- 48
    SELECT count(*) … AND NOT grid_column_is_deleted;                         --  0

**It only bites on an UPDATE** — a first create has nothing to delete, which is
why grids 110 and 111 (one post each) are fine. **Workaround: omit
`replace_columns` on the re-post.** Reported as notes (33).

Not the same code path as `/ui-table-masters/create`, whose `replaceColumns`
works — ui_table 32 was re-registered twice the same day and kept its 24.

**UPDATE 2026-10-02 — the replace_columns bug is FIXED server-side** (grid-details.service.ts: it now retires only columns live BEFORE the save that the payload did not name by grid_column_id). And the old workaround is now WRONG: an update posted WITHOUT replace_columns and without ids APPENDS a second full column set (grid 114 briefly had 37 = 17 old + 20 new duplicates). **Correct habit now: update a grid with `grid_id` + `replace_columns: true`, once, then read fixed.grid_columns back.**

**(Historical) So the two registration endpoints need OPPOSITE habits, which was the trap:**
- `/ui-table-masters/create` — post TWICE (a new column ignores
  `uiTblClmColumnVisibility` on the first).
- `/grid-details/create` — post ONCE, or omit `replace_columns` on the second.

Always read the columns back from `fixed.grid_columns` with
`NOT grid_column_is_deleted` after registering. The response echoes an empty
`columns` array either way, so it proves nothing.

## A combo editor that "does not take" (2026-10-11, Expense Voucher)
`setModelData` writing the TEXT first and the id (UserRole) second loses the id: the EditRole write fires `QAbstractItemView::dataChanged`, which calls `setEditorData` on the STILL-OPEN editor; a `setEditorData` that matches by the (still old) UserRole id puts the combo back on the old row, and the next line writes that old id. Read `currentData()` + `currentText()` into locals FIRST, then write UserRole, then EditRole. (VoucherLineDelegate escapes it only because its setEditorData matches by text.)

**2026-10-22 — leaving a grid.** `focusNextChild()` on a NexTable never left it: QAbstractItemView::focusNextPrevChild sends the view a synthetic Tab KEYPRESS, keyPressEvent took it as a typed char and OPENED the column's lookup popup (Enter on the last lookup cell of the last row). Even QWidget::focusNextPrevChild started from the wrong widget (item card: jumped to Branch). Now `NexTable::leaveGrid(next)` walks next/previousInFocusChain FROM the grid (skips own children, proxied containers, hidden/disabled). Shared change — every grid's Enter-past-last / Backspace-before-first. Also `advanceAfterPopupPick(row,col)`: opt-in, call from a gridPopupSelected handler; stays put if the cell ended empty (refused/reverted). Enter stops = `fixed.ui_table_columns.ui_tbl_clm_column_focus`; item card tables 16/17 FIXED 10-22 (user OK) via `PUT /ui-table-masters/layout-settings` (by uiTblClmId; flags only — the narrow route, no replaceColumns): 17 hides ipm_id/Comp#/Br#/Unit#/Gdn#/Profit, stops Company>Branch>Uom>Godown>Cost>Profit_A>PriceA>…>PriceD>MRP>Dis%>DisQ; 16 hides ir_id/ir_branch_id/ir_godown_id/ir_unit_id, stops uom>branch>godown>min>max>reorder level>qty>type. Backup: Shared/Prathap/inventory/ui_table16_17_before_focus_fix_2026-10-22.csv. Layouts are fetched per open (no cache). Reorder headings renamed 10-22 via /ui-table-masters/create + replaceColumns (all 12 cols with ids; 12 rows after, none appended): Unit, Branch, Godown, Min Qty, Max Qty, Reorder Level, Reorder Qty, Type. Backup ui_table16_before_rename_2026-10-22.csv.

## Old value showing through the cell editor (2026-10-22, every grid)
Cause: theme template Part A `QTableWidget QLineEdit / QLineEdit:focus / QSpinBox / QSpinBox:focus { background: transparent }` (lines 68-94) — also hits delegate editors, so typing 80 over 100 showed both. **Fix is a STYLESHEET change, not code** (user: "just reverse the code… its a simple stylesheet change"; a code helper was written then fully reverted — it also broke the no-style-in-code rule [[theme-template-decision]]). Seed v29 = v28 with those 4 lines → `background: {{table.row}};` (share: Shared/Prathap/theme/seed-template-v29.qss). Preview: `SS_QSS=<seed> COL=15 SRC=snap_price_edit tests/itemcard/run.sh` prints editor bg (v28 #00000000, v29 #ffffffff). Offscreen grab does NOT show the overlap even when broken — read the editor's palette Base alpha instead.

## Tab LEAVES every grid (user rule, 2026-10-22)
"On every table Tab should move to the next focus widget, not the next cell." Enter still walks cells. NexTable overrides `focusNextPrevChild(next)` (what QWidget::event calls for Tab/Shift+Tab — before keyPressEvent) → `leaveGrid` → `NexGridLook::focusOutOf(view, next)` (nex_table_header.cpp: walks next/previousInFocusChain FROM the view, skips own children, proxied containers, hidden/disabled and QTabBar — "tabs are not fields"). Tab INSIDE a cell editor: delegate closes with EditNextItem/EditPreviousItem → `NexTable::closeEditor` commits with NoHint then leaves. NexApiTableView overrides focusNextPrevChild the same way. Test: tests/itemcard SRC=test_grid_tab (8, real keys). Enter on an EMPTY lookup cell still opens its lookup (not a move) — pick a filled non-lookup cell when testing Enter.
