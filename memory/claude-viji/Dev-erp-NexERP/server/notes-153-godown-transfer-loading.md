---
name: notes-153-godown-transfer-loading
description: "Notes 153 (2026-10-26, 2 revisions) Godown Transfer loading/unloading — built A–D + E1/E3/E4/E5: svh_load_*/svh_unload_* + svi_load_rate columns, server-summed (qty+free)×rate charges, TRF voucher DR LOADING_CHARGES / CR LOADING_PAYABLE at despatch (any cogs mode), menu-41 rights; branch-transfer charges refused for now"
metadata:
  node_type: memory
  type: project
  originSessionId: 46d6391f-c740-4636-9dfe-826333b87402
  modified: 2026-10-26T05:11:33.712Z
---

Built 2026-10-26 (uncommitted at hand-off) from Viji's notes 153, migration
`20261026100000_transfer_loading_charges_notes_153`:
- **A/B** 13 header columns on stock_voucher (crew uuid[] no FK, driver/supervisor/unload-supervisor →
  employee_master FK, vehicle → vehicle_master FK, two charges numeric(18,2) with
  ck_svh_(un)loading_charge = 0 unless ticked) + svi_load_rate/svi_unload_rate numeric(18,4).
  Shared header DTO carries the fields (adjustment OmitType's them; opening/physical are standalone,
  so 400); `rules.allowsLoading` (TRANSFER_OUT + TRANSFER_IN) gates them in assertPayloadRules →
  422 STOCK_LOADING_*. `vehicleNo` on the shared header now stores svh_vehicle_no (it used to be
  silently dropped on save). Charges summed in writeHeaderTotals → loadingCharges: round2(Σ (svi_qty +
  svi_free_qty) × rate) — the user's rev-2 rule: labour is paid on what was carried.
- **E4** (migration `20261026110000_transfer_reasons_notes_153`, + Stock_Reason_Master.sql) shared BOTH
  reasons REPLENISH / REARRANGE / RETURN_STORE for {TRANSFER_OUT,TRANSFER_IN}; picker accepts
  voucherType TRANSFER_OUT/IN, strict + direction BOTH only (drops TRANSIT_LOSS by rule, not by code);
  transfer saves refuse a non-BOTH reason (STOCK_REASON_WRONG_DIRECTION).
- **E5** getById LEFT JOINs stock_lot when rules.requiresLot and reads COALESCE(lot, line) for
  batch/mfg/expiry/mrp/sale price/serial (+ the batch-detail lateral); read-only, other types unchanged.
- **C** item-lookup (keyed unit `um`) + adjustment pick-stock (holding's unit) return loadRate/unloadRate.
- **D** StockPostingService.post → accounts.postTransferCharges when postShape TRANSFER_OUT and status
  POSTED; voucher type **TRF** "Stock Transfer" (id 47 on .106, BOTH/JOURNAL, menu 41, not in register);
  roles LOADING_CHARGES (Direct Expenses) / LOADING_PAYABLE (Current Liabilities), catalogued; global map
  rows created by fn_seed_ledger_map in the migration. Unmapped → 422 STOCK_LOADING_ROLE_UNMAPPED.
  Cancel → reverseForVoucher (allowsLoading added to the cancel condition).
- **Inter-branch built 2026-10-26 (notes 156)**: StockPostingService.post calls postTransferCharges for any
  allowsLoading doc ending POSTED or IN_TRANSIT — branch despatch posts its loading at the sending branch
  (STOCK, TRANSFER_OUT, out id), the receipt its unloading when it posts (STOCK, TRANSFER_IN, in id) at the
  receiving branch. LOADING_WRONG_HALF stays; LOADING_CHARGE_INTER_BRANCH retired. Settle-short moved to
  (STOCK, **TRANSFER_SHORT**, out id) (SHORT_SETTLE_SRC_DOC_TYPE) — ux_avh_src allows one live voucher per
  link and the despatch's labour voucher holds TRANSFER_OUT. e2e `test/transfer-branch-charges-notes-156`
  round-trips an item with NO stock at Counter2 (value-neutral) and deletes its own stopped-run drafts via API.
- **E1** recomputeHeaderTotals skips reversal rows for postShape TRANSFER_OUT only (every other type
  still re-totals a cancel to 0 — the notes' "other lists keep figures" was true only for cancelled drafts).
- **E3** /stock/transfer gated on menu 41 (TRF_RIGHT_*); /receive not gated. On .106 only `ravi` holds a
  menu-41 row — VKPOS and tester1 hold none.

**Why:** 3.0 recorded loading "cooly" but never posted it; the user decided it posts.
**How to apply:** e2e `test/transfer-loading-notes-153.e2e-spec.ts` (8 cases) is the live proof. Do not
run the old transfer HTTP suites — see [[stale-transfer-http-suites-are-destructive]].
Related: [[stock-to-accounts-posting]], [[new-ledger-role-needs-catalogue]].

**Notes 154 (same day, reads only):** `GET /stock/transfer/receive` (StockTransferService.getReceipt — TRANSFER_IN rules, header + linkSrcRefno/fromBranchId/fromBranchName, lines, the DESPATCH's transit rows); inbound rows + outVoucherId/outAccYear/fromBranchName/fromGodownName; prefill outVoucher + from/to names; every header toBranchName. Read-only e2e `test/transfer-receipt-load-notes-154.e2e-spec.ts` opens the newest posted receipt. /receive still ungated (193), branch form still on menu 41 (not 192).
