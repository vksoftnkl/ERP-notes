---
name: godown-transfer-screen
description: Godown Transfer (41) + Branch Outward (192, same class Mode::BranchOut) + Branch Inward (193) BUILT + live 10-26, suites ALL PASS; notes-153/154; theme v38; menus hidden/ungranted
metadata:
  node_type: memory
  type: project
  originSessionId: 9f0add08-de5f-4644-a836-0665690adb44
  modified: 2026-10-26T04:19:44.798Z
---

Started 2026-10-26 ("shall we write godown transfer").

**User decisions 10-26:**
- Layout = the EXISTING Sept mockup `share/godown_transfer_entry_ui_mockup.png` ("use our existing designed mockup") — not a new one. Add only 3.0's Loading / Unloading boxes (user: "in 3.0 i have fields like loading & unloading details refer for any missing features").
- Loading/unloading charges **post to accounts** (3.0 only recorded them). My design: Dr LOADING_CHARGES / Cr LOADING_PAYABLE (new roles), paid later by Payment; independent of cogs_mode.
- One To godown per transfer (server rule kept).
- Unloading box on the same screen (same branch); on Branch Inward later.

**Facts:**
- Server routes exist and work live: save → despatch (same branch = POSTED, 2 ledger rows) → cancel. Save & Post = save, then `/stock/transfer/despatch` (two calls). Header TO godown comes back as `godownId`/`godownName`, FROM as `fromGodownId`.
- Device must be a registered device (test: `01a14a14-…` debug-void-dev); refno TRF/2026-2027/<device>/<n>.
- /txn/adjacent docType STOCK_TRANSFER exists (menu 41 rights). Menu 41 "Stock Transfer" hidden; 192/193 = Branch Outward/Inward.
- 3.0 source: `~/Projects/NextLvl/3.0/pos/Stock/stocktransferentry.*`, DB `3.0` `public.stock_transfer` (+ `fstock_transfer`). Per-line cooly = units.unit_loading when item.add_cooly; NexERP has `item_unit_master.unit_loading/unit_unloading` + `item_master.item_allow_loading`. 3.0 Godown Dispatch (237) = sale-bill delivery, NOT a transfer.
- notes-153 (share inventory/) = server ask: svh_ crew/load/unload columns, svi_load_rate/unload_rate, rates in item-lookup/pick-stock, the posting, a TRF voucher type, list totals on cancel, menu-41 rights.

Registered 10-26: list grid **183**, line ui_table **54** (share grids/godown_transfer_registration.py).

**BUILT 10-26 (uncommitted):** `src/modules/inventory/stock/transfer/` (entity, delegate, entry .h/.cpp/.ui), `TxnDocTypes::godownTransfer()` (grid 183), `fm.reg(41)`, `ApiEndpoints::StockTransfer`, `NexDropdownMulti::setSelections/selectedTexts` (no signal). `GodownTransferEntry::kHandlingLive=false` keeps the crew/rate fields OFF the wire (server is forbidNonWhitelisted) — flip when notes-153 A–C land. Reason combo disabled until /stock/reasons takes TRANSFER_OUT (notes-153 E4). Loaded lines get batch from pick-stock (E5).
- Test: `EXTRA_LIBS=-lQt6Test tests/godowntransfer/run.sh` (GT_SNAP=<dir>) — posts 2 pcs Coimbatore→namakkal (Texttile Item lot 5437-0001) and cancels; ALL PASS. Ctrl+PgUp gets 403 until VKPOS has menu 41.
- Theme **v36 PUBLISHED** 10-26 (md5 3cfd946a): QDialog#GodownTransferEntry block.
- Trap: run.sh's ninja sometimes links a stale .ui — run ninja yourself first after a .ui edit.
- Waiting on user: make menu 41 visible + rename "Godown Transfer" (PATCH /menu-masters/visibility is GLOBAL), grant VKPOS menu 41 (user_menus save is a FULL REPLACE), pass notes-153.

Files: `src/modules/inventory/stock/transfer/`. Related: [[stock-adjustment-screen-built]], [[stock-section-ownership]], [[mockup-fidelity]].

**Branch Outward + Inward BUILT 10-26 (user: "he is running now shall we go branch outward and inward after ran we will back to godown transfer" — Prathap doing notes-153; RETURN to Godown Transfer after: flip kHandlingLive, rates, reasons).**
- Outward = `GodownTransferEntry(parent, Mode::BranchOut)`: header relaid in code (applyMode) to the branch mockup; To branch dd 61 (ibr_comp_id), To godown dd 26 filtered by the TO branch; LR/vehicle/expected go on despatch body (stored on transit rows, read back from transit); transit panel + Settle short (TRANSIT_LOSS via /stock/reasons/list); never cancelled. List grid 184, menu 192.
- Inward = `BranchInwardEntry` (branch_inward_*): worklist grid 186 (one row per despatch, no year filter) → prefill → lines at remainder; + = DAMAGED split, − drops split/zeroes; outcome card; Unloading box (gated). List grid 185, ui_table 55, menu 193. Receipt GET missing on server → loadVoucher says "cannot open yet" (notes-154 #1).
- Godown dd 26 now filtered ibranch_id (was listing all branches' godowns).
- Tests: `SRC=test_branch_transfer EXTRA_LIBS=-lQt6Test tests/godowntransfer/run.sh` (Coimbatore→Counter2/karur 2 pcs, receive, send back by API) ALL PASS; godown suite still ALL PASS.
- Theme v37 (BranchInward block, transit pill) + v38 (outcome title tone) PUBLISHED, md5 e5bee3a0.
- Another device 44c01ccc (Prathap e2e?) posts transfers on the box concurrently — tests use relative stock checks.

**10-26 pm:** VKPOS GRANTED menus 41/192/193 (all flags; full-replace diff = only +3 rows). Menus still HIDDEN (visibility not asked to flip). notes-153: DB + source done (synced 10:35) but the running API on .106 is the OLD build (save → VALIDATION_WHITELIST loadEnabled; no pick-stock rates; reasons picker refuses TRANSFER_OUT) → **notes-155** asks build+restart (now covers 154 too: source 11:11, not running). Probe script: scratchpad gt/p153.py. Next when live: flip kHandlingLive in both entries, read loadRate/unloadRate from pick-stock, use /stock/reasons?voucherType=TRANSFER_OUT, check posted TRF voucher + reversal.

**10-26 late: notes-153 + 154 LIVE after Prathap's restart, verified.** kHandlingLive=true in both entries. Godown: loading 2×1.50 → TRF voucher 3.00 (DR LOADING_CHARGES / CR LOADING_PAYABLE), reversed on cancel; GET returns crew/charges/batch. Branch Inward reopens receipts (draft → laid over prefill, editable; posted → read-only, despatch columns hidden). Server REFUSES labour rates on branch transfers (STOCK_LOADING_CHARGE_INTER_BRANCH) → branch screens record crew only (rate cols/charges hidden, rates sent 0) → **notes-156** asks branch posting (loading at despatch/sender, unloading at receipt/receiver). Both suites ALL PASS; lot 5437-0001 Coimbatore 96 / karur 10 (restored after a test-cleanup bug, now guarded: send back only what arrived).

**Committed 10-26:** 62d1d87 on main (fast-forwarded at the user's request; user commits straight to main). Menus 41/192/193 made VISIBLE (PATCH /menu-masters/visibility, user said yes). Menu 41 still named "Stock Transfer" (no rename route used).

**notes-156 LIVE 10-26:** branch despatch loading → TRF voucher at the SENDING branch; receipt unloading → voucher at the RECEIVING branch. Rate columns/charges back on in branch mode (despatch sends unloadRate 0). Both suites ALL PASS. Committed on main (after 62d1d87). Nothing open on transfers.

**10-26: headers to 4 columns** (user: 'make 4 cols in header'), 11-col grid (pairs at 0/1,3/4,6/7,9/10; gaps = columnminimumwidth 16). Branch layout set in applyMode. Both suites ALL PASS. Committed on main.

**10-26 eve, user review (commit 75c682b):** Free qty now limited to the holding's FREE stock (sbl_free_in − free_out; pick-stock `freeAvailableQty` asked in notes-160 A, absent = 0 → Free locked); `#` col hidden, vertical header shows row numbers; Scan barcode button + long hint labels removed; footer = compact strip (Loading 2×3 | Unloading 2×2 | totals 2×2), grid stretches (`loadLayout(id, false)` — NexTable pins height otherwise); New → focusEntryStart + 250 ms focus retry; crew pickers → **dd 72 EMPLOYEES BY DESIGNATION** (iemp_branch_id + iemp_role, falls back to all); designations Loadman/Driver/Supervisor + ZT crew employees created; StockPickDialog ↑/↓/PgUp/PgDn from search. Branch Outward Cancel = **recall** while nothing received, `kRecallLive=false` until notes-160 B. Refno shows machine uid because resolveDeviceCode uses dev_device_uid → notes-160 C (dev_code, 5-digit). Edit & amend like sales: user said SKIP for now. One flaky core dump seen in the branch suite (not repeated in 4 runs).

**10-27 user UI asks (uncommitted):** lblShapeNote removed; Pick from stock moved into the button footer (lines bar + "THE LINES" title gone); audit (Created · Posted · Cancelled · Reason) is ONE line right-aligned under the status pill; voucher no shown via `shortRefno()` (uuid segment → first 8 hex, full refno in tooltip) until notes-160 C. Branch Inward .ui not changed. Build trap: Qt 6.10.3 vanished from ~/Qt (only 6.10.2) → build/Desktop_Qt_6_10_3-Debug cannot regenerate; compile-checked in a scratch build against 6.10.2.

**10-27: notes-160 A/B/C LIVE + verified.** A: pick-stock freeAvailableQty, save refuses 422 STOCK_FREE_QTY_EXCEEDS_FREE_STOCK. B: /stock/transfer/recall → kRecallLive=true; branch suite R1 recalls from the screen. C: dev_code D1..D14 (NEXERP MACHINE D1, dbg device D14), refno TRF/2026-2027/D14/00056. Both suites ALL PASS (uncommitted, with the 10-27 UI asks). Still bad on box: 2 Coimbatore SALEABLE holdings with free_out > free_in (lots 3f0e912c, 00afd732 CBE26-00016) — not fixed, not asked.
