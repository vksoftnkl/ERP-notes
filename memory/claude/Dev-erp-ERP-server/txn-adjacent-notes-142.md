---
name: txn-adjacent-notes-142
description: "GET /txn/adjacent (notes 142, built 2026-10-23; notes 152 walk key 2026-10-24) — one Prev/Next for 14 docTypes from a fixed registry; a per-device serial is NOT in the walk key"
metadata:
  type: project
---

`src/modules/txn-adjacent/` (registry, service, controller, README). Built 2026-10-23, uncommitted
at hand-off. One raw query per docType from `TXN_DOC_REGISTRY`: (date[, serial], created_on, id) walk,
current doc checked under four keys AND docType membership (404 TXN_DOC_NOT_FOUND), VIEW on the
docType's fixed menu (403 TXN_RIGHT_VIEW via assertMenuRight prefix TXN); VOUCHER joins user_menus
per row on vchr_menu_id like /vouchers/adjacent. Optional filters are appended only when set (no
`$x IS NULL OR`, see [[prisma-generic-plan-slow-reports]]).

Facts the notes' proposal got wrong (checked live): stock_voucher has 11 types; STOCK_ADJUSTMENT =
ADJUSTMENT/ISSUE/DAMAGE/EXPIRY_WRITEOFF and must exclude sale-bill shadows (ISSUE rows with
svh_link_src_module='SALES', grid 122's COALESCE(...,'STOCK')='STOCK'); transfers are TRANSFER_OUT
(STOCK_TRANSFER) and TRANSFER_IN (STOCK_TRANSFER_RECEIPT), both menu 41. Draft-serial sentinel
follows each list: sale bill / Rct / Pmt HIGH (bigint max), register vouchers LOW (0). RECEIPT and
PAYMENT were added beyond the notes (Rct menu 99, Pmt menu 100, avh_against_voucher_id IS NULL).

**Why:** the Qt client (TxnKeys, OpeningStockEntry::walkList) replaces per-screen walks with this.
**How to apply:** a new transaction screen = one registry entry (table/cols, own-row filter,
statuses from the CHECK, menu, draft sentinel from its list grid's ORDER BY). Error_Messages.json
went to 2026-10-23.1 with 4 TXN_* rows reusing approved wording. ExpV (menu 277) not registered.

**Notes 152 (2026-10-24):** the walk key carries a serial only where it is counted branch-wide
(`TxnDocSpec.serial`, null otherwise). svh_slno is per DEVICE (ux_svh_slno) and a POS bill's
sb_bill_slno comes from the device's own series (`posSeriesPerDevice`), so the five stock docTypes
and SALE_BILL walk by (date, created_on, id) — created_on follows the branch refno because both are
given at the first save. Quotation/order/DC/DCR/SR number on the branch 'MAIN' series (no
deviceCode passed to allocateVoucherNumber) and keep their serial; Rct/Pmt/VOUCHER keep the
company-wide avh_voucher_slno with the draft sentinel. Don't put a per-device serial back, and
don't add the device to the key: a per-device counter has no cross-device order.
