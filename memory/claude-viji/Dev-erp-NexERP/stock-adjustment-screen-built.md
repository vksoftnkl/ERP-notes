---
name: stock-adjustment-screen-built
description: "Menu 264 Stock Adjustment coded + live-tested 2026-09-30 — file set, the contract facts the live API forced, box ids, notes 65, and the VKPOS grant still owed"
metadata:
  node_type: memory
  type: project
  originSessionId: 447f5bbc-a547-4078-a179-c98ad1d28ec3
  modified: 2026-09-30T05:21:25.028Z
---

**Built 2026-09-30** in `src/modules/inventory/stock/adjustment/`:
- `stock_adjustment_entity.h`: the Cols enum plus a `Kind` enum of six;
- the delegate, a predicate-driven editor;
- `stock_pick_dialog.*`: Pick from stock, plus the damaged list grouped by supplier;
- `stock_adjustment_entry.*/.ui`.

The list is `TxnDocTypes::stockAdjustment()` over grid 122, registered as `fm.reg(264, …)` in `form_inventory_menu.h`. Live suite: `tests/stockadjust/run.sh`, 50/50. It posts, then CANCELS, an adjustment and a move. `ADJ_SNAP=<dir>` saves PNGs.

**Contract facts the live API forced** (see [[verify-against-artefact]]):
- The header godown ALWAYS goes in `fromGodownId`. The load's is `fromGodownName`; `godownId` is the TO side.
- The same adjustment rules run at SAVE: a bad line is a 422 whose `errors[].field` is `lines.<payload index>`. The screen maps it back through `m_sentRows`.
- Save with `status: POSTED` = save+post in one transaction. The response has no `kind`, only GET does.
- Lines come back as magnitudes plus `direction`: ±1 on ADJUSTMENT and re-lot, null on the write-offs (always out), −1 with `toBucket` on a move. The screen holds Qty SIGNED.
- Header totals are SIGNED (the net) since notes 65 was fixed.
- `pick-stock` `bucket` is optional since notes 65; empty means every bucket.
- The Adjustment kind hides RELOT_* reasons; the Re-lot kind offers only them.
- The item lookup reuses `/stock/opening/item-lookup` for unit + signature.

**Server defects:** `share/schema/stock/notes-65-stock-adjustment-screen.md`:
- a same-lot re-lot posts; the screen guards against it;
- totals Min(0);
- grid 122 floods with sales shadow ISSUE rows;
- pick-stock bucket is not optional;
- costRate unit text.

**Changed on the box:**
- ui_table 41 re-registered to 31 columns. The first attempt posted twice without ids and DUPLICATED 21–30; a `replaceColumns` post with ids fixed it. So don't post new columns twice without ids.
- Grid 122 widths scaled to 100.

**VKPOS granted menu 264 on 2026-09-30, at the user's request.** It holds all eight flags, including post and cancel. The full-echo save ([[user-menu-full-replace]]) was diffed before and after: the only change was the +264 row.

**Notes 65 all FIXED and verified live 2026-09-30.** The client workarounds are removed: totals are signed, the bucket is optional, and the signature is read from the /get line. The client re-lot guard is kept as an early warning.

**Re-laid to the mockup 2026-09-30** ([[mockup-fidelity]]):
- ui_table 41 now has 33 columns: 31 Note (the row note) and 32 → (the move arrow). Barcode is hidden.
- `applyKindColumns()` owns visibility per kind.
- The damaged list is inline (`frameDamaged`) on Move stock.
- F2 is a shortcut, not a button.
- Edit and Cancel show only for a saved document.

**2026-10-24 rework (user: "like opening and physical stock", textile, keys, test front+back end) — uncommitted:**
- .ui = Opening Stock's top band (title + lblKindHint + scope + pill + lblAudit left; card right: Doc No · Date · Rate Source (in) · frameGodown rowspan 2; Default Reason · Ref · Remarks), Type row, grid, cards/damaged panel kept, frameButtons with TxnKeys captions + new btnHistory. Backup of old .ui: scratchpad only.
- Keys: TxnKeys like Opening; **F2 = Edit now; Pick from stock moved to F12** (NexTable lineStockRequested); Ctrl+PgUp/PgDn → /txn/adjacent STOCK_ADJUSTMENT; F1 sheet lists F12/↓/+/−.
- Barcode column now shown on keyed kinds (hidden on Move); ui_table 41 Barcode moved to position 1 (share schema/stock/adjustment_barcode_first.py, upsert by id). ↓ item list, click blank row → Barcode, Enter stops via whyNotEditable, barcode commit queued + persistent index.
- Fixed: item pick read item_name (grid 71 sends item_name_en) → Item cell was blank.
- Textile: size/colour/model in Item text (Cols::itemText, also Pick from stock); a scanned code that 404s as EAN (auto batch on) opens Pick from stock with setSearch(code), all buckets.
- Theme **v35 PUBLISHED** 10-24 (md5 d11e7f4d): StockAdjustmentEntry frameGodown/lblAudit/frameButtons.
- Tests: tests/stockadjust/run.sh ALL PASSED after repoint (dbg device; keyed lines use ZT-ITEM-A because Coimbatore SALEABLE = only E2E items with no default unit); NEW `SRC=test_adjust_ux EXTRA_LIBS=-lQt6Test tests/stockadjust/run.sh` all pass (ADJ_UX_SNAP=<dir>).
- Server review → **notes-151** (inventory/): no menu-264 rights + company from body, cancel leaves avg at cancelled cost (ZT-A QUARANTINE avg now 60, was 50 — needs DB fix), draft totals/createdBy from body, docDatetime vs freeze, rules outside post txn, re-lot can change bucket, lot company unchecked; textile pick-stock/get lack size/colour/model, no exact batchNo, by-batch SALEABLE-only.

**notes-151 LIVE + verified 2026-10-24** (all 10 done; #1 company compare declined by design like 150 #7; #5/#7 code-read). Client: barcode 404 + auto batch → `resolveBatchLabel` = pick-stock `batchNo` exact (1 row → taken, >1 → Pick dialog, 0 → "Not found"); `pickStock(..., limit, batchNo)`. test_adjust_ux A10 rewritten (posts 5 of CBE26-00006, scans it, cancels).
**Cost anchor on the box:** ADJ/2026-2027/debug-void-dev/24 = ZT-ITEM-A 1 pc DAMAGED Coimbatore godown @50, POSTED, keep it. Without it ZT-A has no cost (the #2 fix removed the cancelled-doc average) and stock-in at AVG_COST → 422 STOCK_LINE_COST_UNRESOLVED, failing tests/stockadjust/run.sh and physical run-live.
