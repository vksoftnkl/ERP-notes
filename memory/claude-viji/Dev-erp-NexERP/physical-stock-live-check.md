---
name: physical-stock-live-check
description: Physical Stock (menu 45) live-verified 2026-09-30 — 3 client bugs fixed (F9 Load Sheet never wired, variance value lost on save/reopen, wrong "N of M" count), suite tests/physicalstock/run-live.sh
metadata:
  type: project
---

Checked against the live server on 2026-09-30. The contract matches:
- every payload key is in `SavePhysicalStockVoucher*Dto`;
- the count sheet answers with `data.items`;
- the /get lines carry every field the screen reads;
- grid 101 binds `isvh_*` (company / branch / year / status / date range).

**Client bugs found and fixed:**
- **`loadCountSheet()` was NEVER connected.** No button and no F9 in any commit since 09-09, so the whole-godown sheet was unreachable. Added `btnLoadSheet` ("Load Sheet - F9", shortcut in the .ui).
- The reload confirm fired on a fresh screen (there is always one blank row). It now asks only when a holding is on screen.
- **Saved lines carry no avgCostRate**, so after a save repaint or a reopen the variance VALUE went blank and a re-save sent `totalValue` 0.
  - `applyLines` now keeps the averages already on screen, keyed by lot|bucket.
  - `refillAverageCosts()` fills them on a fresh load from the cached count sheet.
  - `applyHeader` drops the sheet cache, because `setSelection` does not fire `selected`.
- The save message said "142 of 1 holdings": the total was read after the repaint. It now reads "142 of 143".

**Grid 101 column names:** `net_variance_qty`, `variance_lines`… not `svh_total_*`. Read the stored header totals through /stock/physical/get.

**Suite:** `tests/physicalstock/run-live.sh`, 24/24. It counts only ZT-ITEM-A's Coimbatore SALEABLE holding, posts +1, then cancels. The on-hand figure moves between runs (other tests), so the suite reads it live.

**Still deferred, by design:** the sheet-filter chooser (group / brand / section / include-empty); `countSheet()` is called with no filters.

Related: [[opening-stock-txn-list]], [[stock-adjustment-screen-built]].

**2026-10-02 relayout (same as opening stock, user ask):** title + scope + status pill, header `frameFields` card in 4 columns (Sheet/Date/Godown/Reason · Rate/Ref/Remarks wide · Freeze/From/To/Blind), grid, lblAudit + `frameTotals` strip, lblHint, `frameButtons` (Load Sheet F9 · New · Save Draft F5 · Save & Post Ctrl+Enter · Cancel Sheet · Edit F2 · Show List F8 · Close). Ctrl+Enter = save & post. focusEntryStart() / refocusGridAfterPopup() as on opening (item pick → focus column Counted Qty). Template **v15** LIVE (md5 273fcd91…). tests/physicalstock/run.sh passes; run-live.sh not re-run.

**2026-10-06 — Load Sheet filter + paging (user: "filtering like Change Selling, if not worth avoid"). Decided: worth it, small.**
- **BUG fixed:** count-sheet was called with no limit, and the server default is 200 (ceiling 1000), so the sheet AND the picker cache silently stopped at 200 holdings. `fetchSheetPages()` now pages 1000 at a time until a short page.
- `countSheet()` always sends includeZero (the server default is true, so it could never say no) plus limit / offset.
- **F9 opens `PhysicalSheetFilterDialog`:** item group (dd 17), bucket, "include holdings with no stock" (default OFF; 38 of 152 were empty even on the box). Only what the route filters; brand / rack would need server work.
- **Load ADDS** missing holdings and keeps counted lines; the destructive "replace every line?" confirm is gone. Matching is lot × bucket (`rowForHolding`, was lot only).
- `buildLines` numbers lineNo 1..n by position, because merged queries' numbers collide.
- Test: `tests/physicalstock/run-sheet.sh` (read-only) 9/9. `run-live.sh`'s modal closer ACCEPTS the dialog with defaults; not re-run.

**2026-10-24 rework (user: "change ui as like opening stock entry", textile, keys, test front+back end) — uncommitted:**
- .ui = Opening Stock's top band (title/scope/pill/lblAudit left; card right: Sheet No · Count Date · Rate Source · frameGodown rowspan 2; Reason · Ref · Remarks; Freeze row + Blind), lblGridHint + totals, buttons via TxnKeys (Ctrl+N/S/Enter, F2, Alt+X, Ctrl+H/O, Esc, F1; Load Sheet keeps F9; Ctrl+PgUp/PgDn → /txn/adjacent PHYSICAL_STOCK). Notices go through setNotice() under the audit line.
- Enter stops: blank row Barcode→Item, holding row Counted Qty only → next row; ↓ in Barcode = item list; click blank row → Barcode; barcode commit QUEUED (opening's mid-commit crash); persistent index on the scan reply.
- Textile: user said NO separate columns — size/colour/model ride in the Item text (Cols::itemText "Shirt · XL · Maroon · Slim fit"); scan falls back to /stock/lots/by-batch (includeEmpty) when the EAN 404s and auto batch is on.
- BookQty now Text format (0 was painted blank). Save warns when the server re-based a counted line's book.
- Theme **v34 PUBLISHED** 10-24 (md5 7eabc35b; share theme/seed-template-v34.qss): PhysicalStockEntry frameGodown/lblGridHint/lblAudit grey. (v33 = PriceChangeLog, another session.)
- Tests: run.sh all pass, run-sheet 9/9, run-live ALL PASSED (repointed to Trichy QUARANTINE lot dd08de2b…, dbg device, filter "include empty" ticked; grid 101 has no svh_usr_refno → matched by refno), NEW `SRC=test_physical_ux EXTRA_LIBS=-lQt6Test tests/physicalstock/run-live.sh` 30/30 (PSE_SNAP=<dir> for PNGs).
- Backend review → **notes-150** (inventory/): duplicate lot×bucket lines double-post, two freeze drafts deadlock, backdated docDatetime dodges freeze, re-save re-bases book, totals/createdBy from body, no menu-45 rights, tenant from body, older count cancellable; textile count-sheet lacks size/colour/model, by-batch SALEABLE-only.
- Box: ZT-ITEM-A's only holding is now Trichy QUARANTINE (book 0); Coimbatore has none. PHY0001–0009 all CANCELLED test sheets.
**2026-10-24 later: notes-150 LIVE + verified** (#1–5, 8, 9, 10, AM/PM; 6/7 unverifiable). Client: lines now send sviId (server's bookChanged/previousBookQty only flags matched lines) and the save message prefers the server flag; label scan = by-batch bucket=ALL + godownId; cancel confirm suggests a second count. NEW test `SRC=test_physical_textile tests/physicalstock/run-live.sh` (posts a textile opening, scans its label, moved-book warning) ALL PASS. test_physical_ux now counts picked holdings (ZT-A has an empty Trichy SALEABLE row since the adjustment review).
