---
name: qty-price-screen-redesign
description: Menu 244 Qty-wise Price rebuilt 2026-10-22 in the promotion page's look — slab grid + "Applies to"/"When" panel + preview; theme rules drafted, NOT published
metadata:
  type: project
---

2026-10-22, user (6 points: change the UI, focus moves on after a popup pick, Uom like the price grid, no button autofocus, an Item-list shortcut gated by permission, "looks like a promotion page"). The user picked option A, "Slabs + detail panel", from an ASCII preview — always confirm a layout before building (see [[sale-bill-new-layout]]).

- **.ui** (generated; original in scratchpad qp/item_qty_price_entry.ui.orig): title + subtitle, then four sections whose headings carry a rule like the promotion page's (Item, Quantity slabs, Applies to — slab N, When — slab N), a violet preview card (`framePreview`), and Delete slab | Clear F7 | Save F5 | Close. The root stretch puts the grid at 1.
- **Grid** shows Uom · From · To · Mode · Value only (IqpCols::isGridColumn; re-asserted on layoutLoaded because ui_table 19 still lists 22 columns, so a harmless "22 vs 23 columns" log line appears). The new `IqpCols::Value` (22) is the one figure; `applyModeGating` copies it into DiscPerc/DiscRs/Price for the mode, so the save path is unchanged. Modes read "% off / ₹ off / Fixed price" (modeChoices/modeCodeOf; legacy P/Q/F still parse). An open To shows "& above". Enter stops: setFocusStops(Uom, From, To, Mode, Value). +/- line keys. Uom = ItemUnitPicker.
- **Panel** ddCompany 22 / ddBranch 32 / ddCustomer 39 / ddPriceLevel 34, dtFrom/dtTo, chkActive/chkInclTax write the current slab's cells (setRowCell → checkRowUniqueness). The old grid popups (grids 72/70/79/78) are removed. A dropdown pick moves focus on by itself (NexDropdownSingle::focusNextChild).
- **"Today (BOX): WS ₹105 · RET …"** comes from /stock/price-bulk?itemId (headline rows per iuc_id); level shorts from /price-level-masters/get. The preview reads "Buy 777–787 BOX at RET: 5% off → ₹104.50 each (today ₹110.00)".
- **No button takes focus** (NoFocus). First focus is the barcode box (or the grid when opened for an item). applyPermissions: Save needs Create or Edit on 244; Delete needs Delete.
- **Item Master "Qty price"** action (Ctrl+Q, requiresSelection), shown only when permissionsFor(244).canView is not false. It sets ctx.menuId=244, applyMenuGovernance, then openItem(id, name).
- **Theme:** the rules are drafted in scratchpad qp/itemqtyprice_rules.qss (blue headings, light rules, cap muted, violet preview card) and previewed in the snapshot via QSS_DRAFT. **PUBLISHED as v30 on 2026-10-22** (user: "publish v30"); the live md5 033b3278 was read back equal to the share seed-template-v30.qss.
- **Tests:** itemmasters SRC=test_qty_price_screen (33/33; test data now has non-overlapping slabs because the server refuses overlaps since notes-139 A4 — that was the old 9 failures; SNAP_OUT + QSS_DRAFT for the look) and itemcard SRC=test_item_list_qtyprice (5/5).
