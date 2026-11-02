---
name: till-masters-screen
description: "Till masters = FOUR NexCrud screens since 2026-10-09 (275 counters, 280 safes, 281 reasons, 282 denominations); live test 58/58 under each own menu; notes-101 DONE"
metadata:
  node_type: memory
  type: project
  originSessionId: 03c71e67-0fa4-4cad-8b97-744c2796ec2a
  modified: 2026-10-09T13:07:20.465Z
---

**Split 2026-10-09 (user):** the one tabbed Till Masters screen (menu 275, 1,367-line class + 1,762-line .ui) was "difficult to understand and maintain". User chose four screens with four menus, entry as a NexCrud popup. The old till_masters_entry.* is deleted; a backup is in that session's scratchpad (old_till_masters/).

**Files** (src/modules/sales/till/masters/):
- till_master_util: TillMaster::text/flag/num/money/idOrNull/textOrNull, and NoteField (a QLabel + NexCrudFieldWidget that posts nothing, for facts and warnings in a popup).
- till_master_list: base NexCrudMasterPage. Show inactive (iinactive), no Delete, "Active on / off" action (btnToggleActive, Ctrl+D). Reload 1.2 s after a save (grid cache).
- till_{counter,safe,reason,denomination}_{list,entry}. fm.reg: 275 / 280 / 281 / 282 in form_sales_menu.h.

**Popup pattern:** endpoints.get is left EMPTY. The server /get needs id+companyId and lacks display names, so the popup fills from the grid row: setEditId + fillFromJson(fromRow(row)). fromRow() is also the toggle body; strip the display keys (deviceName / safeName / ledgerName), because the DTOs forbid unknown keys.
- Shipped reason / denomination: lockReadOnly.
- Reasons have Copy shipped (Mode::Copy: code and category locked).
- A used denomination locks currency / kind / value.
- A live counter locks device and drawer.

**Shipped rows (user 10-09: "why can't I edit any reason"):** every reason on the box is shipped (company NULL). Edit on a shipped, not-yet-replaced row opens a COMPANY COPY: reasons Mode::Copy with category and code locked; denominations copy=true with currency/kind/value locked. Active off on a shipped row posts an inactive company copy, which hides it for this company. A replaced shipped row (seen only with Show inactive) opens read-only. The Copy shipped button was removed. Grid 135 got `overridden` (same currency + kind + value, the server's denominations/list rule) and hides replaced rows like 134 does.
**Grid 134 has OTHER readers:** Open Till, Cash Movement and Settlement pickers send icategory '' + iinactive 'false'; the Session Cockpit sends iinactive 'true' (names past events). The split broke all four (0 rows) until fixed 10-09. Expense Voucher 277 now reads its EXPENSE reasons from grid 134 instead of /expenses/quick-reasons, which lists a shipped reason beside its company copy; rows are mapped to the quick-reasons keys.

**Traps:**
- A NexCrud staticCombo starts on NO item and posts null: set defaults in the ctor for new records.
- NexCrudMasterPage action objectNames are on the QAction, not the QToolButton. Tests trigger() them.
- List pages are modal QDialogs: a harness modal-closer must skip NexCrudMasterPage and NexCrudForm.
- Locked NexCrud line edits stay white: the theme has no QLineEdit:disabled style.

**Grids 133-136** updated in place with `share/grids/till_masters_registration.py --apply --update`. Tokens: + iinactive; 134 + icategory; widths sum to about 97; bool columns typed "Boolean". The backup of the old grids is in the scratchpad.

**Server (notes-101, DONE 10-09, migration 20261009190000):** menus 280-282 exist and 275 is renamed Till Counters. TILL_MENU.COUNTERS/SAFES/REASONS/DENOMINATIONS: each master is judged by its own menu; denominations/list = 272 VIEW else 282 VIEW. 275's grants were copied to 280-282.

**Menu group (notes-102, open, 10-09):** the user wants the four under Sales → Till → **Till Masters** (new group, proposed id 283, parent 271, pos 4.00; group needs a VIEW grant). Menus are server data only; the client needs no change (buildMenu is recursive). Approval Setup 276 stays under Till.

**Tests:**
- tests/till/run.sh → test_till_masters.cpp, live 58/58 (incl. TEA ledger change → expense voucher list): creates ZQ rows, deletes them, and restores ZTS as the default safe.
- SNAP_SRC=snap_masters tests/till/snap.sh → snap_till_<screen>[_entry].png.

User 10-09: the denomination screen shows NO count-grid preview ("no need to show how the count grid will work").
