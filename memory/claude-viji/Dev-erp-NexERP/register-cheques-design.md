---
name: register-cheques-design
description: Cheques on the Voucher Register's Receipt Voucher (the user's #1): mockup rev 1 in share/accounts, decisions Q1-Q7 awaiting the user, and the facts it rests on
metadata:
  type: project
---

**2026-09-26:** the user said "ok" to Q1-Q7 on `share/accounts/register_cheques_ui_mockup.png`.
- Docs: `share/accounts/register_cheques.md` (contract §3, client plan §4, tests §5).
- Build list for Prathap: `notes-54-register-cheques.md`.
- **The Qt client is NOT coded:** it waits for the live routes (the verify-against-artefact rule).
- Contract facts:
  - The instrument keys mirror the Receipt's tender/cheque DTOs: `tenderId`, `refNo`, `instrumentDate`, `bankName`, `cheque{drawerName, bankBranch, ifsc, micr}`.
  - Collected-by = `avh_employee_id uuid[]`, which the Receipt already uses (`avhEmployeeId`).
  - The new Instrument grid column goes on the END of ui_table 39 (index 12), positioned after Group.

**Why:** the user receives cheques without knowing which bank they'll be deposited to, so the register needs a Cheques-in-Hand path. Today S16 refuses that ledger in the register, because menu 51 owns the cheque rows.

**Shape drawn:**
- An Instrument column per CUSTOMER line (Ctrl+Q dialog). The choices come from the tender masters.
- The Dr side is GENERATED from each tender's ledger: CHEQUE → "Cheques In Hannd", CASH → Cash In Hand, BANK/UPI → Kvb Current A/c on Acme.
- Each cheque becomes a HELD `acc_pdc_register` row, and menu 51 takes over unchanged.
- A post-dated cheque gets its own voucher on the cheque date (the Receipt's rule).
- Cancel is refused once a cheque is deposited.

**Q1-Q7 (all confirmed):** per-line instrument; generated debits; no bank at entry; PDC as its own voucher; the cancel guard; Payment Voucher next (#2, needs the cheque-book / issued-cheque routes from share/payment); an optional collected-by salesman.

Related: [[voucher-register-screen]], [[received-cheques-design]], [[receipt-design]].


**BUILT 2026-09-26 (notes 54 live, migration 20260926120000):**
- Client:
  - `voucher_instrument_dialog.*` (Ctrl+Q), the Instrument column (ui_table 39 col 12 at position 4; widths now sum 100);
  - the Cheques card, `lblPostDated`, Collected-by (dd 38 → `employeeIds`);
  - the post-dated amber tint through delegate roles (UserRole chip, UserRole+1 post-dated);
  - `applyLoaded` folds in `pdcVouchers[]`;
  - menu 51's Enter asks `/vouchers/get` and routes to `VoucherRegisterEntry::openFor` when `rules.inRegister`.
- Live names:
  - `/vouchers/instruments` → `data.tenders[]` + `typeName`;
  - a generated leg links by `fromRows`;
  - `/get` has `instruments[]` + `pdcVouchers[]`;
  - `/validate` has `derived.postDated[]` and `isPostDated/postsOn`.
- Test `run.sh cheques` 23/23. Test posts rcv00020–22 all cancelled.
- Asks left in notes 54: `typeCode` on `/cheques/get receiptVoucher`, and `employeeIds` on the `/get` header.


**#2 Payment Voucher cheques: CONFIRMED 2026-09-26 (Q1-Q8 "ok").**
- Docs: `share/accounts/register_payment_cheques.md`.
- Build list: `notes-55-payment-voucher-cheques.md` (13 items).
- The client waits for the server.
- Key contract points:
  - The PmtV instrument is `{tenderId, bankLedgerId, chequeBookId, instrumentDate, refNo, favouring, acPayee}`; NO leaf is sent.
  - New route `GET /vouchers/cheque-books`; a new column `apd_ac_payee`.
  - In menu 52, stop and returned reverse ONE line through ChqBnc.
  - The cheque-book master = `ChequeBookDialog` opened from menu 52 (no new menu).
- File: `share/accounts/register_payment_cheques_ui_mockup.py/.png`.
- Shape: the same per-supplier Instrument column.
  - A CHEQUE credits the BANK ACCOUNT chosen in the dialog, NOT the CHEQUE tender's Cheques In Hand (that is the receiving side).
  - The leaf is the book's next, taken at Post.
  - Presented = a bank date only (P11b).
  - A PDC is its own voucher (its TDS on that date too).
  - Cancel is refused once presented.
  - Favouring + A/c payee.
- Depends on `share/payment`, none of it on the box:
  - `acc_cheque_book` (P4) + the cheque-book routes + a master screen;
  - the P11b CHECK;
  - `/issued-cheques/*` + the menu 52 screen.
- `acc_pdc_register` already allows `apd_tra_type` 'P'.


**#2 PmtV cheques: register side BUILT 2026-09-26** (notes 55 items 1-10 live).
- Client:
  - the dialog's paying shape (`Context.paying` from partySide DR): bank account, book, next leaf, favouring, A/c payee, UTR;
  - `ChequeBook` + `loadBooks` (`/vouchers/cheque-books` → `data.books[]`);
  - `grpBooks`, the paying columns on the Cheques card, "Paid by".
- Test `run.sh paycheques` 15/15. pmv00036 and pmv00038 are cancelled.
- NOT live yet: `/cheque-books/*` and `/issued-cheques/*`. The code is in the tree, but the server answers 404; restart asked in notes 55.
- So menu 52 (Issued Cheques) and ChequeBookDialog are still to code.
- `/get` returns instrument `bankLedgerId` null (asked).
