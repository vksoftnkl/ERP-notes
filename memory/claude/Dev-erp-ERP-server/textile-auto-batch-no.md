---
name: textile-auto-batch-no
description: "Textile plan 2026-10-23 — server S1/S2 built: auto batch no on opening save (stock.auto_batch_no, BATCH counter), TEXTILE_BATCH preset, GET /stock/lots/by-batch; notes 149 (10-24) fixed opening DTO whitelist, re-save ownership, draft-cancel retire; tag plan (notes 136) PARKED"
metadata:
  node_type: memory
  type: project
  originSessionId: c349e680-a950-41b9-a156-1bc150064582
  modified: 2026-10-24T00:00:00.000Z
---

Textile plan (pasted 2026-10-23, "auto batch no on purchase, scan the batch at the sale") replaces the
heavy tag page; the tag plan (notes 136) is PARKED, not cancelled. Server S1 + S2 built 2026-10-23:

- **S1:** `assignAutoBatchNumbers` (`stock-voucher-batch-numbering.helper.ts`) runs inside
  StockVoucherService.save() when the rules say `autoBatchNo` (OPENING only). The batch format is
  `<br_bill_prefix else br_code, upper><YY>-<NNNNN>`. The counter is allocateVoucherNumber under
  acc_voucher_types code **'BATCH'**, looked up by code: the id is a sequence value (45 on dev).
  Device MAIN, reset NEVER.
- **Setting:** `stock.auto_batch_no`, COMPANY, default false. It is read via fn_app_settings_effective
  inside the tx.
- **Refusal:** no bill prefix and no code → 422 STOCK_BATCH_PREFIX_MISSING.
- **Preset:** `TEXTILE_BATCH` (BMS, LOT_ACTUAL, FIFO, ALLOW, sort 90) lives in the Stock_Track_Presets.sql
  seed only.
- **S2:** `GET /stock/lots/by-batch` (src/modules/stocks/lot-by-batch) matches slt_key_batch on the new
  partial index `ix_slt_key_batch`. It counts SALEABLE stock only. Nothing found → 404
  STOCK_BATCH_NOT_FOUND.
- **Migration:** 20261023150000_textile_auto_batch_no.

**Plan claims corrected:**
- The voucher type CHECK needs affects_accounts OR affects_inventory, so BATCH has inventory = true.
- The reset enum is NEVER, not NONE.
- ix_slt_batch_no indexes the raw spelling, which is why the new index exists.

**Addendum 2 (size / colour / model, same day), built 2026-10-23.** Migration
20261023160000_textile_size_colour_batch_detail.
- **Masters:** `inventory.item_size_master` (isz_) and `item_colour_master` (icl_), company NULL = shared.
  Routes `/item-sizes/*` and `/item-colours/*` use the house verbs create/get/delete/restore, not the
  plan's /save. A taken name → 409 with `errors[0].existingId`. Registry ids 921/922.
- **Batch detail:** `stock.stock_batch_detail` (sbd_, registry 1022) is keyed by (company, folded batch).
  sbd_id = uuid v5 (BATCH_DETAIL_ID_NAMESPACE, computed in TS).
- **Writes:** the opening save writes it via `syncBatchDetails`, only for auto batches. On a typed batch
  it refuses with 422 STOCK_BATCH_DETAIL_NEEDS_AUTO_BATCH. A deleted draft soft-deletes its rows.
- **Routes:** `/stock/batch-detail/{models,update}`. update is refused on a draft (409) and audits under
  screen "Batch Detail".
- **Reads:** the scan, bill / DC / return /get (sbiSizeName / sbiColourName / sbiModel) and the
  `sales.bill.items` print all join on company + item + folded batch.
- **Screens (notes 147, migration 20261023170000):**
  - Menus 291 Item Size Master / 292 Item Colour Master, under 34. They are also in Menu_Master.sql.
  - Grids 181 / 182 (`MAIN LIST - ITEM SIZE / COLOUR`).
  - Dropdowns 70 ITEM SIZES / 71 ITEM COLOURS.
  - Free on 2026-10-23: menu 293+, grid 183+, dropdown 72+.
- **Notes 147 Q1:** BATCH is hidden from the voucher register and the day summary picker
  (in_register false, menu NULL, affects_accounts false, no headers). It shows only as counter rows on
  /sequence/list.

**Notes 149 (2026-10-24, Qt retest of the opening screen) — all three fixed:**
- #1 `sizeId/colourId/model` had been added to the shared transfer line only; `SaveOpeningStockVoucherItemDto`
  refused them with 400 VALIDATION_WHITELIST. Now declared there too.
- #2 `syncBatchDetails` ownership is by DOCUMENT + batch (any sbd row, live or retired, whose
  sbd_src_doc_id is this doc), not by svi_id: the Qt grid re-saves without sviId and hands the
  server's number back, so the old `ownedBatchByLine.get(svi_id)` check 422'd NEEDS_AUTO_BATCH.
  A key left out = none (full-replace semantics).
- #3 `cancel()` of a DRAFT now calls softDeleteBatchDetailsOf (a POSTED cancel keeps the rows: the lot
  exists). The opening /get join is `sbd_src_line_id = svi_id AND sbd_batch_no = upper(btrim(svi_batch_no))`
  with NO is_deleted filter, so a cancelled draft still reads what was keyed.
- The Qt harness the notes cite (`SRC=test_textile_cols tests/openingstock/run-live.sh`) is not in
  /home/vk/VKNexERP/NexERP on this box; verified with test/textile-*.e2e-spec.ts instead.

**Open:**
- br_bill_prefix is NOT unique (br_code is), so two branches with one prefix could repeat numbers.
- Split lines (splitNo > 1) still need a typed batch.
- Qt Q1/Q2, the purchase module (D3 reuse) and labels (D8) are not built.

**How to apply:** a purchase module should call assignAutoBatchNumbers and then syncBatchDetails (with
its own srcDocType) rather than re-implement them.
Never use common/utils/batch-number.util.ts (MAX + 1). See [[textile-tags-notes-136-status]],
[[stock-engine-ddl-lives-outside-repo]].
