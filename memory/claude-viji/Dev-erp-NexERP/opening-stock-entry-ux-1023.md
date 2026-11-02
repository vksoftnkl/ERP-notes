---
name: opening-stock-entry-ux-1023
description: "2026-10-23 user's 10-point Opening Stock (menu 44) entry rework — what was built, the item_name_en root cause, theme v31 + notes-141 live"
metadata:
  node_type: memory
  type: project
  originSessionId: d91ee3ca-fe8a-4093-a454-fea514a4cc2b
  modified: 2026-10-23T05:07:32.490Z
---

User's 10 asks on OpeningStockEntry (menu 44), built + live-tested 2026-10-23 (`SRC=test_entry_ux tests/openingstock/run-live.sh`, 23/23, read-only, item "Samsung H1" — barcodes 750257 default, 750252, 750253).

- **Pick didn't advance (root cause):** grid 71 sends `item_name_en`, the screen read `item_name` → Item cell blank → Item is a REQUIRED stop so focusNextEditableCell refused. Fixed (falls back to item_name).
- **Barcode = item search too, ONLY on ↓** (user 10-23 correction): ↓ in its editor (delegate `itemSearchRequested`) or on a blank cell (screen eventFilter) opens grid 71 with the text. Enter = barcode check only; a 404 shows "not found", clears the cell, NEVER opens the list. `openItemList()` opens from the Item column so the pick lands in applyPickedItem.
- **Default barcode only fills a blank cell** (lookup used to overwrite a scanned code).
- **Click on a blank row** (any cell other than Barcode/Item) → that row's Barcode. Rows WITH an item: clicks untouched (my reading of the ask).
- **Enter stops** (setFocusStops + new `NexTable::setFocusCellFilter`): blank row Barcode→Item; item row Qty → tracked identity cells only → Cost/Unit → next row.
- **F4** = /item-switch-uom + fetchItemDetail(keepLineValues): qty kept, cost per BASE unit kept, MRP/SP/barcode kept. Was a "can't change" message.
- **Layout (rev 2, user 10-23 "4 cols, lhs wider"):** title block left (hboxTop stretch 1:3 + 340px strut), header card right in 4 cols: Voucher No · Date · Rate Source · `frameGodown` (rowspan 2, nexTone missing/set); row 2 Your Ref No · Remarks. Tab = reading order Date → Rate Source → Godown → Ref → Remarks → grid, set on focus proxies. `OSE_SNAP=<dir>` makes test_entry_ux save 3 PNGs (themed).
- **Theme v31 PUBLISHED 2026-10-23** (md5 199230d0, read back = share seed-template-v31.qss). **notes-141 LIVE + verified 2026-10-23**: GET sends createdOn/createdBy/createdByName/modifiedOn; title shows "Created … by …" + "Posted … by …" (test U11). Test now 25/25.
- Old suite `run-live.sh` now 10 FAIL for box-data reasons: device 019e7286 (raman) deleted 10-12; ZT-ITEM-A no longer alreadyOpened in Coimbatore.
- **Crash fix 10-23:** barcode committed with NO godown crashed — resolveBarcode (modal "Godown first" + cell rewrite) ran inside the delegate's setModelData. Now queued from onGridCellEdited; requireHeaderGodown re-activates the window and focuses the godown. Test U14. Rule: never open a modal or rewrite the cell from inside a delegate commit — queue it.
- **Review 10-23 (user "check opening stock front end backend codes"):** 10 Qt findings ALL fixed: LineTicket (QPersistentModelIndex + m_generation + QPointer) on every per-line reply (lookup / barcode / F4) — stale row, New/load/close mid-reply; entryLive() gates picks; grid NoEditTriggers when read-only; lineNumbers() = one line/split rule (split only continues under the same item, splits renumbered); Qty no longer wipes typed cost w/o tax; loadVoucher takes scope only on success; m_isSnapshot blocks Prev/Next; Barcode commit not dirty; re-pick clears old item's cells; F4 clears the old unit's EAN. Tests R1–R4 + popup-list snapshot run 10-23: test_entry_ux 57/57 with v32 live. Picker popup = 75% width × 70% height of the host window, min 900×560 (TxnMainView::openPicker; 85% was too big — user 10-23). Backend: **notes-143** (inventory/) — 10 server findings (draft save vs post race, unlocked already-opened, baseQty trusted, cancel can go negative under ALLOW, line godown/item unscoped, tenant from body…).
- **10-23 evening (user "yes" to pending list):** notes-142 LIVE → Prev/Next now `GET /txn/adjacent?docType=OPENING_STOCK…` (ApiEndpoints::TxnAdjacent; walkList/grid-99 walk deleted). notes-143 all 10 in server code ("notes 143 #N" markers); #4 decided: a stock voucher CANCEL is always refused if it drives a holding negative. Item lookup now sends the line's godownId (alreadyOpened per godown+year, #8); warning text "…already has an opening in this godown this year". Old suite repointed to device dbg (01a14a14…, login uid debug-void-dev): `tests/openingstock/run-live.sh` 36/36 (L2 'already opened' now ledger-driven; L5 pick-in-Trichy warns). test_entry_ux all pass (U11 waits for OPN0006). Uncommitted.
