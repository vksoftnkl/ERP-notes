---
name: bank-details-popup
description: Supplier/Ledger bank accounts — NexBankGrid is a read-only list; Add/Edit/Enter/double-click open NexBankAccountDialog (.ui); delete as before
metadata:
  type: project
---

2026-10-22, user: "edit only in a popup, stop editing in the table cell, show only the account details". NexBankGrid (src/shared/widgets, used by SupplierEntry and LedgerEntry) now has NoEditTriggers and row selection. Buttons are Add / Edit / Delete; Enter or a double-click on a row edits it. The popup `NexBankAccountDialog` (nex_bank_account_dialog.{h,cpp,ui}) has every lba* field: Account No / Holder / Bank required, IFSC upper case, a type combo, and a Default checkbox (locked on for a single-mandatory Bank-profile ledger). Ticking Default clears the others. The Default cell in the table is display-only. Accounts still save with the ledger payload (fieldValue). The delegate's cell combo editor was removed. Test: tests/bankgrid/run.sh (19/19, offline).
