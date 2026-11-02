# Status: purchase screens rebuilt to the mockups (fidelity), 2026-10-29

Plan side: the approved mockups `purchase_*_ui_mockup.png` in this folder, plans `plan-purchase-frontend.md` and `plan-purchase-backend.md` (not edited). Audits: `audit/audit-{invoice,po,grn,return,claims,shortage}.md`. Nothing is committed. Server notes for Prathap: `notes-163-server-changes-2026-10-29.md`.

## 1. Per screen

"Before" counts are the audit's own tallies (rows judged against the mockup). "After" is only written down where the builder updated the audit (Supplier Dues). For the other screens the after-snapshots exist (`audit/*_after_*.png`) but NO re-tally was written; I did not guess one.

| Screen (menu) | Audit rows | Before: YES / PARTLY / DIFFERENT / NO | Built today | After state |
|---|---|---|---|---|
| Purchase Invoice (24) | 47 | 9 / 20 / 8 / 10 | rebuilt (theme block v42, shared parts `purchase_parts.*`, `purchase_invoice_parts.*`) | after snapshots `invoice_after_*` (6 files); no re-tally |
| Purchase Order (23) | 53 | 18 / 8 / 7 / 20 | rebuilt; Reorder suggestions moved to F7 (button "Reorder suggestions - F7"); theme v43 | `po_after_01..11` (11 files); no re-tally |
| GRN (196) | 57 | 17 / 11 / 8 / 21 | rebuilt; inline F12 panel, transport popup, serial-refuse strip, textile mode; theme v44 | `grn_after_01..11`; no re-tally |
| Purchase Return (25) | 62 (61 judged) | 18 / 15 / 9 / 19 | rebuilt; theme v45; "Supplier dues" button opens page 297 filtered | `return_after_01..10`, `return_modal_*`; no re-tally |
| Supplier Dues (NEW, 297; replaces the Claims dialog) | 40 (39 judged) | 10 / 9 / 5 / 15 | built: `SupplierDuesPage` + `ShortageCloseDialog` in `src/modules/purchase/dues/`; theme v46 | audit table: 38 of 40 rows YES (row 24 "shape differs"), row 17 PARTLY (no bold on one column), row 40 NOT TESTED; `dues_after_01..06` |
| Shortage Receipt (195) | 44 (43 judged) | 9 / 10 / 9 / 15 | IN PROGRESS (theme block v47 drafted, `shortage_receipt_*` files modified) | not finished; not tested |

Theme files (all UNPUBLISHED, share only): `theme/seed-template-v42.qss` to `v47.qss`, with per-screen blocks `block-purchase-invoice-v42`, `block-purchase-order-v43`, `block-grn-v44`, `block-purchase-return-v45`, `block-shortage-receipt-v47`. v46 (dues) has no block file of its own. Publishing v47 includes v42 to v46. The audits were taken on v41.

Old `PurchaseClaimsDialog` (ui_table 60) is deleted from Qt (`purchase_claims_dialog.*` shown as deleted in git).

## 2. Server follow-ups the screens depend on (see notes-163)

New: `/purchase/supplier-dues/{open,settle,settle-undo}`, shortage `close` kind CLAIM, `/purchase/terms/*`, GRN validate + override + transport band + richer `/get`, invoice agent / bill-from / transport / override, `/purchase/returns/validate`. Migration `20261029130000_purchase_fidelity_db_needs` applied 17:32.

## 3. Deviations from the mockups (as recorded by the builders)

1. Supplier Dues, Settle: cash / bank is NOT a settle option; it is the Receipt link (user decision 10-29). Credit note or "Adjust in next bill" only.
2. Supplier Dues, "Settled" column is headed "Credited" and counts credit only (server `credited`); write-offs are a separate column.
3. Supplier Dues: one claim row at a time for a partial amount; several selected rows each settle their whole open amount.
4. Supplier Dues row 17: one column cannot be bold (no fonts in code, QSS cannot reach one grid column).
5. Return: ITC reversed on the strip counts destroyed goods only (server `prItcReversed`); the mockup's figure covers every outcome. Not changed server-side.
6. GRN: Print labels keeps no Ctrl+P (Ctrl+P is document Print); "Labels..." opens after post.
7. Fonts and colours come from the theme only; the old hard-coded green / red on the invoice Difference figure was a house-rule breach and moves into the theme.

## 4. Test results

Exact counts could NOT be confirmed by me. What is on disk and what I could see:
1. Snapshot harnesses (not assertions): `tests/purchase/audit_snap_po_grn.cpp`, `snap_po_after.cpp`, `snap_grn_after.cpp`, `snap_return_after.cpp`, `snap_return_dues.cpp`, `test_audit_invoice_snap.cpp`.
2. New Qt suite `tests/purchase/test_purchase_supplier_dues.cpp`; run with `SRC=test_purchase_supplier_dues tests/purchase/run.sh`.
3. New / updated API suites in this folder: `test_purchase_dues_shortage_claim_api.py` (17:18), `test_purchase_return_api.py` (17:15), `test_purchase_shortage_api.py` (17:15).
4. Server specs added: `purchase-supplier-dues.spec.ts`, `purchase-grn-checks.spec.ts`, `purchase-invoice-checks.spec.ts`, `purchase-order-status.spec.ts`, `purchase-shortage-close-ref.spec.ts`.
5. No pass/fail counts were handed to me. The main session should paste the last run of the Qt purchase suites and the three API suites here before commit. The 10-28 baseline was 19 API + 19 Qt suites green (before the rebuild).

## 5. Open questions

1. Publish themes v42 to v47 (and v47 once Shortage is finished)? Nothing has been published; the live look is still v41 (as the audits were taken).
2. Run the error-message seed (catalogue in DB is 2026-10-29.1; file is .4).
3. Menu 179 "Claim Management" (Accounts) is now unused; leave it hidden, or retire it?
4. Return Reason dropdown: the seven `PRET_*` reasons are shared seed rows; confirm the wording list is right.
5. GRN "Received by" vs "Supervisor": audit asked whether they are two people; the server now stores both (`receivedBy[]`, `supervisorId`). Confirm the screen labels.
6. Cash purchase stays DEFERRED (10-28). Purchase reports stay SKIPPED.
7. A supplier CN number typed on the Return header still does not create a settlement on the dues page (row 40, not tested).
8. Prathap's open P1/P2 topology answer from notes-162.
9. Against-GRN return mode was never rendered (needs a posted, un-billed ZT GRN); logic read from code only.
10. Textile mockup comparison was not done on invoice / GRN (`stock.auto_batch_no` is off on the test company).
