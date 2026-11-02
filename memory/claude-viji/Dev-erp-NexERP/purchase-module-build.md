---
name: purchase-module-build
description: "Purchase module build state 2026-10-27 — what is live on .106 / Qt, ids, tests, what is next"
metadata:
  node_type: memory
  type: project
  originSessionId: 5f84b1e7-06ed-44c0-b0af-5205a5dbd71c
  modified: 2026-10-27T13:18:41.503Z
---

Purchase is being built straight onto Prathap's server folder (~/Dev/erp/ERP server on .106, branch DEV-v1, commit only my files by explicit path) and Qt main.

**Live (2026-10-27):**
- Invoice menu 24 (grid 189, ui_table 56).
- PO menu 23 (grid 190, ui_table 57): `/purchase/orders/{save,validate,get,pending-lines,cancel,short-close}`.
- GRN menu 196 (grid 191, ui_table 58): `/purchase/grns/{save,post,get,cancel}`.
- Return / claims menu 25 (grid 192, ui_tables 59 lines + 60 open claims): `/purchase/returns/{save,post,get,cancel,sources,source-lines}`, `/purchase/claims/{open,settle,settle-undo}`. Claims open as a dialog from the Return screen; menu 179 "Claim Management" (under Accounts) exists but is NOT used — not ours to take.
- Shortages menu 195 (grid 193, ui_table 61): `/purchase/shortages/{open,receipt/save,receipt/post,receipt/get,receipt/cancel,close,close-undo,closes}`. One screen: open shortages as rows + Supplier CN / Write off / Undo close (undo only for a PART close from the screen — a full close drops the row).
- Backend commits: 49ba9b69, 07098cdd, 5455da58, 8d41c8be, c11109f9, 167b21dd, 3378ea69. Qt commits: 3ce8487, fc46bde, 8ca6ff3, b8f615e.

**Test API (10-29: RETIRED, use :3011, see [[after-move-facts]]):** pm2 `erp-purchase` on :3012 served `.claude-dist`, built with `npx nest build -p .claude-staging/tsconfig.claude-build.json`.

**Live API (10-27, user said "do restart"):** :3011 now runs under pm2 `api` (dist/src/main.js; `npx nest build` then `pm2 restart api`). Before, Prathap ran it unsupervised from an editor terminal. `pm2 save` was refused by the permission check — a reboot will not bring it back until the user saves / allows it.

**Tests:**
- Share `purchase/test_purchase_{invoice,po_grn,return,shortage}_api.py`.
- Qt `tests/purchase/run.sh`; `SRC=test_purchase_{po_grn,return,shortage}` run the other screen tests.

**Return rules:** money per billed unit of the source line (debit note matches the bill); stock leaves the source lot in the row's bucket (one lot can be in SALEABLE and DAMAGED); bill value − lot cost → PURCHASE_PRICE_VARIANCE; a bill / GRN with a live return cannot be cancelled. Claim CREDIT = CN only (cash comes through a Receipt).

**GRN coolie rule:** loadmen are picked as employees (dd 72, LOADMAN). The server uses the company's off-bill UNLOADING charge and pays each `emp_loan_ledger_id`; `payCoolie: false` turns it off.

**Invoice from GRN — BUILT 2026-10-28: backend a772c172 (on :3012; :3011 needs a restart by Prathap/user), Qt 0fbe19c ("From GRN" combo on menu 24; GRN rows edit only Billed/Rate/Disc; ui_table 56 = 27 cols, visible widths 97 — 100 overflowed the row-number gutter):**
- line `grnItemId` (+ `grnItemAccYear`) bills a GRN line WHOLE; `GET /purchase/invoices/grn-lines?companyId&branchId&supplierId[&piId]` (has `blocked` reason).
- P13 = `stock.stock_cost_revaluation` (migration 20261028120000, user OK'd): value-only append-only rows summed by rebuildLotCosts/rebuildItemCosts; writer `revalueOnHand`/`reverseCostRevaluation` in stock-voucher-posting.helper.ts. WAVG share = item on-hand, LOT_ACTUAL = lot on-hand; scr_basis_base_qty = on-hand spread over.
- Cancel reverses × min(1, on-hand now ÷ basis); the rest → Jrl voucher Dr/Cr INVENTORY vs PPV (PUR series is preset = pi_slno, auto-number collides → 23505).
- Challan short adopted via psh_bill_* (reason → SHORT_BILL); bill cancel restores from the GRN line.
- R18 BUILT 10-28 (server 6fab55d3, Qt fff8da8): kept = recd+dmg − posted GRN returns; billed beyond kept up to returned = REJECTED (pii_rejected_at_grn_qty); post writes a no-stock PURCHASE_RETURN debit note (Dr Supplier / Cr GRNI / Cr input GST) — recognised by NO pia alloc rows, cancelled only with its bill; GRN return cancel refused while its line is billed. Test part C (API now 80 checks), Qt G5.
- Charges grid BUILT 10-28 (server 95e20707, Qt 88cd3f2): own NexTable ui_table 62 (NOT the shared ChargeGridController/grid 21 — purchase money is server-only), picker grid 82 imodule_name=P, auto-apply seeded via ChargeGridController::autoApplyCharges, off-bill payees = loadmen (dd 72) → staff ledger via shared payeeLedgers; tests call clearCharges() because the user's test charge "purchase" (P, Less 12, auto-apply) seeds every new bill. Qt test test_purchase_invoice_charges (20).
- F12 batches/serials BUILT 10-28 (Qt only, server already took allocs[]): PurchaseAllocDialog, ui_table 63 (visible 94 — popup gutter), line JSON in hidden col 27 "Allocs"; serial mode from signature 'R' (test item ZZ TEST R (SERIAL)); godown = bill's (no per-row godown yet). Test test_purchase_invoice_allocs (24).
- Textile BUILT 10-28 (server 66a31483, Qt e8789a6): purchase inward posting calls assignAutoBatchNumbers + syncBatchDetails (purchase-doc.helpers numberTextileBatches/writeTextileDetails); line cols 28-32 Size/SizeId/Colour/ColourId/Model (positions after Batch), F12 cols 11-15; R17 = sizes are alloc rows under one printed line. Dropdown pages start at 1 (page 0 fails silently). Test item ZZ TEST B (BATCH). Qt test_purchase_invoice_textile (19), API test_purchase_textile_api.py. Not built: ratio pack, quick-add size/colour, value-based GST slab (P15).
- 10-28 11:00: :3011 (Prathap's nest --watch) stopped listening after my uploads; user OK'd `touch src/main.ts` to nudge it — came back. Never pass a commit message with an apostrophe inside a single-quoted ssh command (truncated 67e2690b; amended).
- Agents (menu 28, grid 194) + Supplier–Item Map (menu 295, grid 195) BUILT 10-28: server 42844aea (agent = ledger under Sundry Creditors 019eee86-f34b-7d73-8a79-f5c6f036439a; psi_id uuid v5; bill post adds AUTO rows), Qt 97a5b48 (NexCrudMasterPage/NexCrudForm). Migration 20261028130000 (user OK'd): POST+CANCEL verbs on 195/196 — the missing verbs are why a rights save dropped them; VKPOS granted 295. Tests: share test_purchase_masters_api.py, Qt test_purchase_masters. NexCrudForm Number fields fill ONLY from JSON numbers; custom fields get setFieldValue(undefined) for absent keys.
- Settings + price writer BUILT 10-28 (server 00d2b259, Qt 9279283): migration 20261028140000 = 13 purchase.* keys (user OK'd; NO cash day limit — s.40A(3) is a statutory limit); loadPurchaseSettings + GET /purchase/invoices/settings; inv_amount_check / duplicate_bill_mode WARN → save returns warnings[]; pi_policy snapshot read at post; price writer = SellingPriceBulkService.applyPurchasePrices (source PURCHASE, menu 24, branch scope; D16 KEPT_NEWER vs price log changed_on; refused prices NONE). Qt: Sell A–D cols 33–36 shown only when allowed; PRICE KEPT in Check. Tests: share test_purchase_settings_api.py, Qt test_purchase_invoice_prices (both restore the company + prices). Other settings (shelf life, rate tolerance hold, PO approval, labels, close PO, return default, price_set_at GRN, COST_ONLY/OPEN_CHANGE_SELLING) are CATALOG ONLY — not wired.
- GRN textile + F12 BUILT 10-28 (Qt 85cab40, no server change): ui_table 58 = 26 cols (share purchase_po_grn_registration.py --update-grn; Signature 19, Allocs 20, Size 21, SizeId 22, Colour 23, ColourId 24, Model 25); GrnEntry copies Purchase Entry's textile/F12 code (not a shared helper); fetchSignature for PO-filled + reopened lines. Test Qt test_purchase_grn_textile 22/22.
- B9 lookups BUILT 10-28 (server dc0974e3, Qt 8f2a1f4): GET /purchase/lookup/supplier-context (GSTIN, MSME payBy via msmeDueDays — category from sup_msme_category else led_msme_type, activity sup_msme_activity; pendingPos; outstanding from acc_bill_balance by party = sup_id; billsOnHold; licence warnings) + /item-context (last 3 posted bill lines all suppliers, poRate from PO pendingLines, stock_balance SALEABLE/DAMAGED, item_price_master row for the unit, pur_supplier_item code). No menu right check (read-only). Qt PurchaseSupplierChips + PurchaseItemStrip (purchase/common/purchase_context.*) on menu 24 only; theme v41 (QDialog#PurchaseInvoiceEntry pills + lblItemContext; share/theme/seed-template-v41.qss). Tests: share test_purchase_lookup_api.py 19, spec 8, Qt test_purchase_invoice_context 18. GAP: supplier form/API (Prathap's suppliers module) does not expose sup_msme_activity/agreement/agreed_days, licence expiries, show_notes_on_entry — MST-01 form fields not built; claude DB role cannot write purchase.suppliers.
- B10 reports: mockup share/purchase/purchase_reports_ui_mockup.png (one screen, 12 tabs, RPT-06 = jump to Party Outstanding) drawn 10-28, AWAITING user approval + needs a new menu (migration).
- Payment hold R19 BUILT 10-28 (server 3a637ba4, Qt 348957f): /purchase/invoices/hold (hold = edit, release = OVERRIDE on menu 24); auto hold at post when a GRN-from-PO line > rate_tolerance_pct; Payment posting calls assertPaymentHoldCleared (purchase/common/purchase-hold.ts) → 409 PUR_PAYMENT_HOLD or HOLD_OVERRIDE logged; open-items PayableBill.paymentHold; TxnStatusEvent HOLD/RELEASED/HOLD_OVERRIDE. B11 (return set-off) was already in Payment (HELD_DEBIT_BILL_TYPES) — tested. Tests: share test_purchase_hold_api.py (toggles VKPOS Override on 24 and restores, user OK'd), Qt test_purchase_invoice_hold. tests/payment/run.sh has 9 DATA failures (E2E suppliers have no open bills left) — not code.
- CASH PURCHASE DEFERRED by user 10-28 ("else we could stop cash purchase"): agreed design when resumed = bill + auto Payment (PaymentPostingService.saveInTransaction/postInTransaction; export from payment.module.ts).
- Still refused: draft GRN return on the line, challan short received/closed before the bill, billed < kept, off-bill charges on such a bill.
- Tests: share `purchase/test_purchase_invoice_grn_api.py` (56) + the 4 older API suites green 10-28; Qt `SRC=test_purchase_invoice_grn` (27) + `test_purchase_invoice` green.
- Open on menu 24 (pre-existing): Godown blank on a reopened bill (no name in /get), "Disc %"/"Cost w/o tax" headings clipped.
- 10-28: a 10:04 User-Rights save dropped VKPOS Post/Cancel/Amend/Override on 195+196; user OK'd restoring Post + Cancel only (via /user-administration/create full set). claude DB role cannot write user_menus.
- 10-28: after a reboot :3011 is Prathap's own `nest start`, pm2 `api` is :3000; I restarted `erp-purchase` (:3012) with `PORT=3012 pm2 start .claude-dist/src/main.js --name erp-purchase`.

**10-28 PM parallel build (user: "do the whole base plan in parallel, stop asking") — ALL phase-1 items except reports (on hold) + cash (deferred):**
- WS1 Purchase Entry (server 638f45cf, Qt 9028830): from PO, cost back-solve + a/b, sch%/extra disc, markup + 10%, ITC col, TCS/EWB, F4/Ctrl+F11/Ctrl+F8 (PaymentEntry::startForParty), last-purchase defaults, unknown barcode→EAN, godown fix, ui_table 56 cols 37–44.
- WS2 keys (server 2fa24c7b adjacent registry, Qt 7c809f9 purchase_doc_nav): F2/Prev-Next(Ctrl+PgUp/Dn)/Ctrl+H/Print on 5 screens; print templates share/print/purchase_doc_templates.py.
- WS3 PO (server d33a0bce, Qt 6c6b5c2): amend+revisions (audit log), clone, merge, approval (OVERRIDE on 23, trail), close-on-first-bill, Reorder screen (ui_table 64, purchase/reorder).
- WS4 (server 861d0345 + fe8888cf, Qt 6e025ef): ratio pack, labels /purchase/labels/from-doc + labels_after_post, GRN shelf life, excess action (ui_table 58 = 29 cols), per-row godown F12, price_set_at GRN (Sell A+MRP), OPEN_CHANGE_SELLING, return settlement default.
- WS5 (server 70b1901e, 53ea3ad9, 006d7dc3, Qt 4c09536): sister bill (GSTIN/PAN match, P19 lock in sales bill-lifecycle), RCM notified-scrap refusal (self-invoice = phase 2 per R13), hold F9/F10 + autosave, attachments /txn-attachments Ctrl+A, value slab fn (inert), MST-01/03/04 fields.
- WS6 (server aad2d6dc): direct-bill shelf life; refusals lifted: draft GRN return, short received/closed before bill, billed<kept (unbilled = free goods, warning PUR_GRN_KEPT_NOT_BILLED); off-bill charges on GRN bill stay refused (R4).
- Integration commits: Qt eb55337, server 8cbae241.
- 10-28 eve APPLIED (user: "apply the migrations directly"; db execute + migrate resolve worked this time) — were: share/purchase/sql/20261028150000_supplier_item_scheme (ENT-07 code not written), 160000 print purposes (+ purchase_doc_templates.py --post), 170100 reorder menu (committed in prisma/migrations, unapplied; then grant 296), 180001 grn prices (Sell B–D), 190000 rcm notified hsn, 190200 tax value slab (+ Prisma model fields).

**10-29 FIDELITY REBUILD (uncommitted; status: share/purchase/status-purchase-fidelity-2026-10-29.md, notes-163):**
- Audits per screen in share/purchase/audit/audit-*.md (before tallies YES/PARTLY/DIFFERENT/NO: invoice 9/20/8/10, PO 18/8/7/20, GRN 17/11/8/21, return 18/15/9/19, claims 10/9/5/15, shortage 9/10/9/15). Qt rebuilt to the mockups: invoice 24, PO 23 (Reorder = F7), GRN 196, return 25, NEW Supplier Dues page menu 297 (src/modules/purchase/dues/; replaces PurchaseClaimsDialog, ui_table 60 retired). Shortage 195 IN PROGRESS. Shared parts purchase/common/purchase_parts.* and invoice/purchase_invoice_parts.*.
- Server (staged, not committed): /purchase/supplier-dues/{open,settle,settle-undo}; shortages/close kind CLAIM; /purchase/terms/*; GRN validate + overrideWarnings + transport band; invoice agent/billFrom/transport/override; returns/validate. Migration 20261029130000 APPLIED 17:32 (menu 297, PRET_ reasons, pur_terms + dd 73, pur_claim_settlement settles a shortage CLAIM close too, Override on 196). Grid 195 scheme cols applied (f2b509ed). VKPOS granted 279, 297, Override on 196.
- Themes v42-v47 in share/theme (UNPUBLISHED); live look still v41. Error_Messages.json is .4, DB catalogue still .1 (seed not run).
- Decisions: dues Settle has no cash/bank (Receipt link); old menu 179 unused. Snapshot harnesses tests/purchase/*snap*; new Qt test test_purchase_supplier_dues. Exact pass counts for the rebuild were NOT recorded.

**Next (old list, superseded 10-29):**
- Charges grid / F12 batch popup on the invoice.
- Supplier-item, agent, settings, price writer.

Related: [[stock-posting-engine]] [[sql-vs-nestjs-rule]] [[grid-runner-binding-rules]] [[feedback-commit-to-main]]

- Prisma trap: schema.prisma is GENERATED (npm run prisma:schema:build from prisma/**/ split files). Edit split files only; a772c172 had put StockCostRevaluation in schema.prisma only → rebuild dropped it; fixed ef26ba7b (prisma/stocks/stockCostRevaluation.prisma). Partial indexes stay out of the model (comment), else migrate diff shows drift.
- Purchase API tests live in /home/vk/Shared/Prathap/purchase/test_*_api.py (not tests/purchase). Print templates script: /home/vk/Shared/Prathap/print/purchase_doc_templates.py (validator rejects ':MI' inside a literal → chr(58)).
