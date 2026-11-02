---
name: issued-cheques-design
description: notes (55) issued cheques / cheque books were DESIGNED here (2026-09-26) because register_payment_cheques.md and plan-backend-payment.md are not on this machine; names may differ from what the client expects
metadata:
  node_type: memory
  type: project
  originSessionId: 3b4d1528-fc7f-40d0-a068-0ea6ddc6abc3
  modified: 2026-09-26T13:17:48.663Z
---

Built 2026-09-26 (migration 20260926190000_payment_cheques, module
src/modules/accountsModule/issuedCheques/, helper vouchers/cheque-book.helper.ts).
The spec docs notes (55) cites (`register_payment_cheques.md` §2/§4.9/§4.10/§5.2,
`../payment/plan-backend-payment.md` P4/P9/P11b) do not exist anywhere on disk;
the user said "design it yourself". Every invented name is listed in the
vouchers module README, section "notes (55)".

**Why:** when the client (or the missing docs) turn up with different names —
`acc_cheque_book` columns, `/cheque-books/*` vs `/vouchers/cheque-books/*`,
error codes — this is a naming mismatch to reconcile, not a bug.

**How to apply:**
- An issued cheque's tender row names the BANK (td_tender_ledger_id) with tender type 5.
  `fn_is_cheques_in_hand_ledger` therefore counts only `td_dr_cr = 'DR'` rows and
  `fn_cheques_in_hand_reconcile` only `apd_tra_type = 'R'` — undo either and every
  bank a cheque was written on gets reconciled against the register (422 ACC_CHEQUES_OUT_OF_BALANCE).
- A cheque's amount is its INSTRUMENT leg, not the party line (53's TDS grosses the line).
- test/cheques-bounce-flow, receipt-backend-gaps, receipt-represented-cheque-guards
  fail (25 tests) with or without these changes — verified by A/B of the two functions.
See [[voucher-register-module]], [[books-reconcile-trial-check]].

**notes (58), 2026-09-28:** cheque books have their own menu, **263** "Cheque Books" under
Accounts (position 13.10, verbs VIEW/CREATE/EDIT only; migration 20260928190000 + the seed
row). Every `/cheque-books/*` route is judged on 263 (`CHEQUE_BOOKS_MENU_ID` in
cheque-books.service.ts), never on 52, even when Issued Cheques' button opens the dialog.
The register's `GET /vouchers/cheque-books` listing is untouched. VKPOS holds 263 on
localhost (a row ADDED to user_menus, not a full-replace save); tester1 gets it from the
e2e MENUS fixture.

