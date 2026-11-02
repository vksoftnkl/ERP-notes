# Commit message drafts, 2026-10-29 (plain text; nothing was committed)

Rules for the committer: commit by explicit path; never `git add -A`; do not change the ERP server index without looking at it first (it already holds about 126 staged files from several agents). Each message below ends with the two attribution lines from the session reminder (Co-Authored-By + Claude-Session); add them when pasting.

========================================================
REPO 1: NexERP (Qt), branch main, /home/vk/Dev/erp/NexERP
========================================================

--- Commit Q1: purchase screens rebuilt to the mockups ---

Subject:
Purchase: screens rebuilt to the mockups; new Supplier Dues page (menu 297)

Body:
- Invoice 24, Order 23 (Reorder suggestions = F7), GRN 196 and Return 25 rebuilt to the approved mockups: section headings, status chips, totals strips, WARN / REFUSE strip with the override tick, inline F12 panel, transport popup
- New Supplier Dues page (menu 297): claims + shortage claims on one page, live posting preview, several rows settled in one call, undo; replaces the Claims dialog on the Return screen (purchase_claims_dialog removed)
- Return screen's Supplier dues button opens the page filtered to its supplier
- Shared parts in purchase/common/purchase_parts.* and invoice/purchase_invoice_parts.*
- Shortage Receipt 195 is IN PROGRESS: leave shortage/ files out of this commit if the screen is not finished
- Snapshot harnesses and a supplier-dues test added under tests/purchase
- Endpoints and menu wiring for the new server routes

Paths:
CMakeLists.txt
src/core/api/api_endpoints.h
src/core/navigation/form_purchase_menu.h
src/modules/purchase/common/purchase_attach_dialog.cpp
src/modules/purchase/common/purchase_attach_dialog.h
src/modules/purchase/common/purchase_doc_nav.cpp
src/modules/purchase/common/purchase_doc_nav.h
src/modules/purchase/common/purchase_labels.cpp
src/modules/purchase/common/purchase_ui_util.h
src/modules/purchase/common/purchase_parts.cpp
src/modules/purchase/common/purchase_parts.h
src/modules/purchase/dues/
src/modules/purchase/grn/grn_cols.h
src/modules/purchase/grn/grn_entry.cpp
src/modules/purchase/grn/grn_entry.h
src/modules/purchase/invoice/purchase_alloc_dialog.cpp
src/modules/purchase/invoice/purchase_alloc_dialog.h
src/modules/purchase/invoice/purchase_invoice_delegate.cpp
src/modules/purchase/invoice/purchase_invoice_delegate.h
src/modules/purchase/invoice/purchase_invoice_entry.cpp
src/modules/purchase/invoice/purchase_invoice_entry.h
src/modules/purchase/invoice/purchase_invoice_parts.cpp
src/modules/purchase/invoice/purchase_invoice_parts.h
src/modules/purchase/invoice/purchase_sister_dialog.cpp
src/modules/purchase/order/purchase_order_cols.h
src/modules/purchase/order/purchase_order_dialogs.cpp
src/modules/purchase/order/purchase_order_dialogs.h
src/modules/purchase/order/purchase_order_entry.cpp
src/modules/purchase/order/purchase_order_entry.h
src/modules/purchase/return/purchase_return_cols.h
src/modules/purchase/return/purchase_return_entry.cpp
src/modules/purchase/return/purchase_return_entry.h
src/modules/purchase/return/purchase_claims_dialog.cpp   (deleted: use git rm or add -u by path)
src/modules/purchase/return/purchase_claims_dialog.h     (deleted)
tests/purchase/audit_snap_po_grn.cpp
tests/purchase/snap_grn_after.cpp
tests/purchase/snap_po_after.cpp
tests/purchase/snap_return_after.cpp
tests/purchase/snap_return_dues.cpp
tests/purchase/test_audit_invoice_snap.cpp
tests/purchase/test_purchase_supplier_dues.cpp

Held back until Shortage 195 is finished and tested:
src/modules/purchase/shortage/shortage_receipt_cols.h
src/modules/purchase/shortage/shortage_receipt_entry.cpp
src/modules/purchase/shortage/shortage_receipt_entry.h

Not confirmed: whether CMakeLists.txt holds only the new purchase sources (it shows +6 / -1); check `git diff CMakeLists.txt` before adding.

--- Commit Q2: test runners and deploy script moved to Qt 6.12 / :3011 (10-29, separate from the purchase work) ---

Subject:
Tests: runners default to Qt 6.12.0 and API :3011; deploy script path

Body:
- every tests/*/run*.sh defaults QT and BUILD to 6.12.0 / Desktop_Qt_6_12_0_Debug
- API base :3012 -> :3011, test psql -> localhost
- scripts/deploy-linux.sh updated for the move

Paths:
scripts/deploy-linux.sh
every modified tests/**/run*.sh, tests/**/*.py and the modified tests/**/*.cpp listed by `git status --short tests/` EXCEPT the purchase ones in Q1 (tests/purchase/run.sh and the modified tests/purchase/test_purchase_*.cpp belong to this commit, because they are the same 6.12 / :3011 switch)
(tests/itemmasters/__pycache__/api.cpython-312.pyc is a tracked byte-code file shown as modified: leave it out)

========================================================
REPO 2: ERP server, branch DEV-v1, "/home/vk/Dev/erp/ERP server"
========================================================

ALREADY COMMITTED by the user today: fafefb4b / 012860b7 / 5eded5e5 (party P1 + customer fixes + migrations 110000, 120000) and f2b509ed (grid 195). Do not repeat them.

--- Commit S1: DTO own-undefined sweep ---

Subject:
Fix DTO own-undefined trap in masters: use isPresentField, never hasOwnProperty

Body:
- a declared DTO field is an own property (value undefined) after the house pipe, so an omitted field wiped the stored value or was treated as sent
- print-template save no longer nulls ptv_params / ptv_note on a status-only version row; published revisions compare against a frozen copy (print-template-frozen-compare)
- same fix in print-template-assignment, charge master, items master, suppliers, sale agent, tax rate master, widget master, area, batch prefix, tender master, ledger shipping address, ledger bank account, device list master, items-cust-rates master, staff advance ledger
- specs now run the body through the house validation pipe

Paths (all src/, plus matching spec files; do not add dist/):
src/modules/settings/print-template/print-template.service.ts
src/modules/settings/print-template/print-template.save.spec.ts
src/modules/settings/print-template/utils/print-template-frozen-compare.ts
src/modules/settings/print-template/utils/print-template-frozen-compare.spec.ts
src/modules/settings/print-template-assignment/print-template-assignment.service.ts
src/modules/settings/print-template-assignment/print-template-assignment.save.spec.ts
src/modules/settings/employeeMaster/staff-advance-ledger.service.ts
src/modules/settings/employeeMaster/staff-advance-ledger.service.spec.ts
src/modules/master/charge-master/charge-master.service.ts
src/modules/master/charge-master/charge-master.service.spec.ts
src/modules/master/widget-master/widget-master.service.ts
src/modules/master/widget-master/widget-master.service.spec.ts
src/modules/master/batch-prefix/batch-prefix.service.ts
src/modules/master/batch-prefix/batch-prefix.service.spec.ts
src/modules/Inventory/items-master/items-master.service.ts
src/modules/Inventory/items-master/items-master.service.spec.ts
src/modules/Inventory/tax-rate-master/tax-rate-master.service.ts
src/modules/Inventory/tax-rate-master/tax-rate-master.service.spec.ts
src/modules/purchase/suppliers/suppliers.service.ts
src/modules/purchase/suppliers/suppliers.service.spec.ts
src/modules/sales/sale-agent/sale-agent.service.ts
src/modules/sales/sale-agent/sale-agent.service.spec.ts
src/modules/sales/area/area.service.ts
src/modules/sales/area/area.service.spec.ts
src/modules/accountsModule/tenderMaster/dto/save-tender-master.dto.ts
src/modules/accountsModule/tenderMaster/tender-master.service.ts
src/modules/accountsModule/tenderMaster/tender-master.service.spec.ts
src/modules/accountsModule/ledgerShippingAddress/ledger-shipping-address.service.ts
src/modules/accountsModule/ledgerShippingAddress/ledger-shipping-address.service.spec.ts
src/modules/accountsModule/ledgerBankAccount/ledger-bank-account.service.ts
src/modules/accountsModule/ledgerBankAccount/ledger-bank-account.service.spec.ts
src/modules/fixed/device-list-master/device-list-master.service.ts
src/modules/fixed/device-list-master/device-list-master.service.spec.ts
src/modules/items-cust-rates-master/items-cust-rates-master.service.ts
src/modules/items-cust-rates-master/items-cust-rates-master.service.spec.ts
src/modules/sales/posting/transport-band.service.ts
src/common/txn-status-log/txn-status-log.helper.ts
src/modules/accountsModule/accountLedgerMasters/README.md

Note: some of these (transport-band, txn-status-log helper) may belong to the purchase commit instead; check `git diff --cached` for each. Not confirmed which agent staged which file.

--- Commit S2: purchase fidelity backend ---

Subject:
Purchase: supplier dues, shortage claim close, PO terms, GRN validate / override / transport, migration 20261029130000

Body:
- migration 20261029130000: menu 297 Supplier Dues, PRET_ return reasons, pur_terms + po_terms_id + dropdown 73, pur_claim_settlement settles a return claim OR a shortage close of kind CLAIM, GRN Override verb, vehicle / driver / memo columns, psr_supervisor_id FK
- new routes /purchase/supplier-dues/{open,settle,settle-undo}, /purchase/terms/{create,get,delete}, /purchase/returns/validate, GRN validate
- shortage close kind CLAIM; GRN post and invoice post accept overrideWarnings + overrideReason (logged WARN_OVERRIDE)
- invoice agent / bill-from / shared transport band; PO termsId, agent and estimated charges
- GRN /get returns names, check, billed, charges and totals for the screen
- specs: supplier dues, grn checks, invoice checks, PO status, shortage close ref
- Error_Messages.json 2026-10-29.4 (seed NOT run yet)

Paths:
prisma/migrations/20261029130000_purchase_fidelity_db_needs/migration.sql
prisma/purchase/purchase.prisma
prisma/schema.prisma   (generated: rebuild with npm run prisma:schema:build; confirm it matches the split files)
prisma/seed/Db_Table_Master.sql
prisma/seed/Dropdown_Columns.sql
prisma/seed/Dropdown_Details.sql
prisma/seed/Menu_Master.sql
prisma/seed/Stock_Reason_Master.sql
prisma/seed/Error_Messages.json
dbml/awesome.dbml
src/modules/purchase/purchase.module.ts
src/modules/purchase/common/purchase-bill-from.ts
src/modules/purchase/dues/purchase-supplier-dues.ts
src/modules/purchase/dues/purchase-supplier-dues.spec.ts
src/modules/purchase/terms/purchase-terms.ts
src/modules/purchase/grn/purchase-grn.ts
src/modules/purchase/grn/purchase-grn-checks.ts
src/modules/purchase/grn/purchase-grn-checks.spec.ts
src/modules/purchase/invoice/dto/save-purchase-invoice.dto.ts
src/modules/purchase/invoice/purchase-invoice.controller.ts
src/modules/purchase/invoice/purchase-invoice.service.ts
src/modules/purchase/invoice/purchase-invoice-checks.ts
src/modules/purchase/invoice/purchase-invoice-checks.spec.ts
src/modules/purchase/lookup/purchase-lookup.ts
src/modules/purchase/order/purchase-order.ts
src/modules/purchase/order/purchase-order-status.spec.ts
src/modules/purchase/return/purchase-return.ts
src/modules/purchase/shortage/purchase-shortage.ts
src/modules/purchase/shortage/purchase-shortage-close-ref.spec.ts
src/modules/stocks/stock-adjustment/dto/save-stock-adjustment.dto.ts   (only if it is part of this work: it shows modified in dist; not confirmed)

Do NOT add: dist/**, logs/error.log, dist/tsconfig*.tsbuildinfo.

========================================================
SHARE (not under git): nothing to commit. Files written today by this note-writer:
purchase/notes-163-server-changes-2026-10-29.md, purchase/status-purchase-fidelity-2026-10-29.md, party/status-party-master-rules-2026-10-29.md, this file.
