# Register cheques #2 — the Payment Voucher issues cheques (rev 1, 2026-09-26)

The user confirmed the picture `register_payment_cheques_ui_mockup.png` (Q1–Q8, "ok")
before this file was written. It is the pay-side mirror of #1
(`register_cheques.md`, notes 54, live), and it builds on the parts of
`../payment/` that were designed but never built. Those parts are named below
and **not repeated**:

- `../payment/plan-backend-payment.md`:
  - §2.4 `acc_cheque_book` (P4);
  - §2.5 the beneficiary / print columns (P9);
  - §2.6 the `ck_apd_cleared` loosening (P11b);
  - §4.9 the cheque-book routes;
  - §4.10 the issued-cheque routes;
  - §5.2 steps 6 and 13 (the leaf lock);
- `../payment/plan-qt-issued-cheques.md`: the menu 52 client plan and `ChequeBookDialog`.

> **In one line:** a supplier line's CHEQUE is OUR cheque. It names a bank account and
> one of its cheque books; the server credits that bank, takes the book's next
> leaf at Post, and writes an ISSUED row (`apd_tra_type 'P'`) that Issued Cheques
> (52) carries on from.

## 1. Decisions (all confirmed on the PNG)

| # | Decision |
|---|---|
| Q1 | The instrument sits on each supplier line, as in #1. A returned cheque reopens only that supplier's bills. |
| Q2 | A CHEQUE or BANK line credits **the bank account chosen in the dialog**. It does not credit the CHEQUE tender's ledger: Cheques In Hand is the *receiving* side. CASH credits the CASH tender's Cash In Hand. |
| Q3 | **The leaf is the book's next one, taken at Post** (`takeNextLeaf … FOR UPDATE`, inside the post). The dialog shows it as "next". No leaf is typed. A spoilt leaf is VOIDED in 52, and the next cheque takes the next leaf. |
| Q4 | **The bank is credited at Post**, the Tally way. *Presented* stamps a bank date (reconciliation) on the bank leg and posts nothing (P11b). The statement balance belongs to Bank Reconciliation. |
| Q5 | **A post-dated cheque** posts as its own voucher on the cheque date, with its TDS deducted on that date. Its leaf is taken **now**, because the cheque is signed today. |
| Q6 | **Cancel is refused** once any cheque of the voucher has left ISSUED (HELD). Stop it or return it unpaid in 52 first. A voided or stopped leaf is never reused. |
| Q7 | **What must exist first:**<br>• the cheque-book table, its routes, and `ChequeBookDialog` (opened from 52);<br>• P11b;<br>• `/issued-cheques/*` and the menu 52 screen.<br>Printing waits for the print module. |
| Q8 | **Favouring** defaults to the supplier's name and is editable. **A/c payee** crossing is on by default. Both are kept on the cheque row for printing. |

## 2. The wire (proposed; the client codes to the live Swagger)

### 2.1 Rules and lookups

- **`/vouchers/types`:** `instruments: true` on **PmtV** as well (the same column as #1).
- **`GET /vouchers/cheque-books?companyId`** (new, own URL): the books a cheque may be drawn on.
  Only active and **not finished** books are listed.

```jsonc
{ "books": [ { "chequeBookId": "…", "bankLedgerId": "…", "bankName": "Kvb Current A/c",
               "bookNo": "12", "leafFrom": "000451", "leafTo": "000500",
               "nextLeaf": "000458", "left": 43, "format": null } ] }
```

- **`/vouchers/instruments`:** unchanged. BANK / UPI / CHEQUE / CASH come from it. On a
  payment, the screen asks for a bank account on BANK / UPI / CHEQUE lines.

### 2.2 The payload: `lines[].instrument` on a PmtV supplier line

```jsonc
{ "tenderId": "<CHEQUE>",
  "bankLedgerId": "<Kvb Current A/c>",     // required on CHEQUE / BANK / UPI of a payment
  "chequeBookId": "<book 12>",             // required on a CHEQUE of a payment; NO leaf is sent
  "instrumentDate": "2026-09-26",          // after header.date = post-dated (Q5)
  "refNo": null,                           // BANK / UPI: the UTR / reference
  "favouring": "Run Supplier A",           // Q8, defaults to the party name when absent
  "acPayee": true }                        // Q8
```

The receipt keys from #1 (`bankName`, `cheque{drawerName…}`) stay the receiving side's.
A payment instrument that sends them is refused.

**New refusals:**
- `VCH_BOOK_REQUIRED`;
- `VCH_BOOK_FINISHED` ("book 000451–000500 is finished — open the next");
- `VCH_BOOK_BANK`: the book is not of `bankLedgerId`;
- `VCH_BANK_REQUIRED`.

### 2.3 `/validate`

- **One generated CR leg per instrument line:**
  - on `bankLedgerId` for CHEQUE / BANK / UPI, and on the CASH tender's ledger for cash;
  - `source INSTRUMENT`, and `fromRows` = the line;
  - `instrument.nextLeaf` = the book's next leaf *as of now*, shown and not promised. Two cheques on one book show `000458` and `000459`.
- Post-dated and TDS per line work exactly as in #1 and notes 53. The PDC voucher carries that supplier's TDS leg.

### 2.4 `/post`

For each CHEQUE line, in the post's transaction (§5.2 steps 6 and 13 of the payment plan):

1. `takeNextLeaf(chequeBookId)`, `FOR UPDATE`. Refuse `VCH_BOOK_FINISHED` when the book has none left.
2. Write the leaf into the `acc_tender_detail` row: `td_ref_no` = the leaf, `td_beneficiary_name` = favouring (P9).
3. Write an `acc_pdc_register` row: `apd_tra_type 'P'`, HELD, `apd_instrument_no` = the leaf,
   `apd_bank_ledger_id` = the bank, `apd_party_id` = the supplier, `apd_voucher_id` (the PDC voucher for a post-dated cheque),
   the favouring, and the A/c payee flag. **A/c payee needs a column** (`apd_ac_payee boolean NOT NULL DEFAULT true`).
4. That line's allocations carry `abj_cheque_id`.

A BANK / UPI line writes its tender row (UTR in `td_ref_no`) and no register row.
If anything fails, the leaf rolls back with the transaction.

### 2.5 `/cancel` and `/get`

- **cancel:**
  - refused with `VCH_CHEQUE_MOVED` unless every cheque is HELD;
  - otherwise the HELD rows → CANCELLED, with **the leaves staying consumed**;
  - the post-dated voucher(s) are reversed too.
- **get:** as #1, plus each cheque's `leaf`, `chequeBookId / bookNo`, `favouring`, `acPayee`, and `pdcStatus`.

### 2.6 Issued Cheques (52) and cheque books

Build `../payment/plan-backend-payment.md` §4.9 and §4.10 as written, with two adaptations
for register vouchers:

- **Stop and returned unpaid on ONE cheque of a MANY voucher** reverse **that line
  only**: Dr bank / Cr that supplier, negative `abj` rows for `abj_cheque_id` = the cheque. The rest of the voucher stands.
  - Returned uses `ChqBnc`, as menu 51's bounce does.
  - Stop uses `ChqBnc` too, with the reason, so that neither steals a PmtV number.
- **Replace** raises a new PmtV voucher (one supplier line, the new leaf, `avh_against_voucher_id` →
  the original).
- **`/cheques/get` (or `/issued-cheques/get`) must name the voucher's type**, so 52's Enter
  opens the Voucher Register for a register voucher. The same ask as notes 54 #1.

## 3. The Qt client (once §2 answers on the box)

1. **The instrument dialog, payment shape.** The screen picks the shape from the party side, never from a type code:
   - Instrument;
   - Bank account (the bank ledgers that have a book; for BANK / UPI, any bank ledger);
   - Cheque book: "Book 12 · 000451–000500 · 43 left";
   - Leaf: read-only "000458 (next)";
   - Cheque date, Favouring, A/c payee;
   - for BANK / UPI: bank account + UTR;
   - the post-dated note.
2. **Instrument cell:** `CHEQUE  Kvb · book 12 · leaf 000458 · 26-09`, and PDC.
3. **Cheques card:** line · favouring · bank·book · leaf · date · amount · A/c payee · status.
4. **Cheque-book card:** the chosen bank's books (from / to / next / left).
5. **Issued Cheques (menu 52):** `plan-qt-issued-cheques.md` as written.
   - `cheque_register_entity.h` is shared with 51.
   - Enter opens `VoucherRegisterEntry::openFor` for a register voucher; the Payment (100) screen is not built.
   - The **Cheque books** button opens `ChequeBookDialog`. It is the cheque-book master: create a book, close a book, and see used / void / free.

## 4. Tests

- **Server:**
  - two suppliers by cheque on one book (leaves 458 and 459, in order), one by NEFT, one post-dated;
  - a finished book → `VCH_BOOK_FINISHED`;
  - two concurrent posts on one book get different leaves;
  - present one (a bank date on the bank leg, no voucher);
  - return one unpaid (only that supplier's bills reopen, the others stand);
  - stop one;
  - cancel a voucher with a presented cheque → `VCH_CHEQUE_MOVED`.
- **Client:**
  - `tests/register/run.sh paycheques`: the same case through the screen, up to Post, the ISSUED rows in 52, and cancel;
  - `tests/issued/`: the menu 52 actions.
