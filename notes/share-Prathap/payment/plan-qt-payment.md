# Payment voucher — Qt client plan (menu 100)

**For Viji. Written 2026-09-14; code after `plan-backend-payment.md` §2 is
run and the `/payments/*`, `/cheque-books/*` routes answer.** House rules in
`../openingbal/plan-qt-opening-balance.md` §0 (own `.ui` on `NexDialog`, no
`NexCrud*` / `TxnMainView`; delegate `cellEdited`; Ctrl+Enter; scope captured
once; doubles not labels). **The receipt screen is coded first; this one is
the receipt with the signs flipped and three additions** — so most of
`../receipt/plan-qt-receipt.md` is reused as classes, not copied. The screen
is `payment_ui_mockup.png` band for band.

## 1. File set

```
src/modules/accounts/payment/
  payment_entry.h / .cpp / .ui           // NexDialog; pages Entry / List (F8)
  payment_bill_delegate.h / .cpp         // BILLS WE OWE: Pay, Disc recd, R/off; PDC issued read-only
  payment_tender_delegate.h / .cpp       // INSTRUMENTS: tender combo, amount, mode/ref, beneficiary (popup), cheque (book + date), PDC chip
  payment_other_delegate.h / .cpp        // OTHER LEDGERS: role | ledger, Dr/Cr, amount, settles, against
  payment_entity.h                       // PaymentHeader (payeeKind, payeeId, favouring…), OwedBill, HeldCredit, TenderRow (+beneficiary, chequeBookId), OtherLine
  payment_beneficiary_dialog.h / .cpp / .ui   // the NEFT/RTGS/IMPS/UPI row's popup: mode, name, account, IFSC, UPI, UTR (3.0 financialvoucherentry fields)
  payment_cheque_dialog.h / .cpp / .ui   // the CHEQUE row's popup: book (combo of active books of the tender's bank with free leaves), next leaf shown read-only, date, favouring, crossed
  cheque_book_dialog.h / .cpp / .ui      // "Cheque books": list (NexApiTableView on the books grid) + open a new book form; also opened from menu 52
src/modules/accounts/common/             // EXTRACTED from the receipt when this lands — shared by both vouchers
  voucher_allocation_engine.h / .cpp     // the receipt's engine with Direction { In, Out }
  voucher_identity.h                     // the identity struct, both directions
  voucher_credit_delegate.h / .cpp       // the credits band (Apply cell) — identical both sides
  voucher_header_edit_dialog.h / .cpp / .ui   // the four-field POSTED header edit + remark
  tender_master_row.h                    // TenderMasterRow moved out of sale_bill_tender.h
src/core/api/endpoints/accounts_end_points.h   // ApiEndpoints::Payments::{OpenItems, PartyContext, Create, Reject, Post, Get, List, UpdateHeader, Cancel}; ApiEndpoints::ChequeBooks::{List, Get, Create, Close}
```

## 2. The `.ui` — page "Entry"

1. **Header band:** `txtPaymentNo` · `datePayment` · `cmbPaidBy` (employee)
   · **`cmbPayeeKind`** (plain `QComboBox`: Supplier / Customer (refund) /
   Other ledger — NEW) · **payee picker that swaps with the kind**
   (`QStackedWidget` of three: `FetchLineEdit` + popup grid 100 POPUP-
   SUPPLIERS for a supplier — reads `sup_id`, `sup_name`, and the new hidden
   the picked supplier id IS the ledger id; `NexDropdownSingle` id 39 for a customer;
   `NexDropdownSingle` id 43 for a ledger — a cash/bank pick is refused at
   once with "use Contra (104)") · **`txtFavouring`** (NEW; default
   `party.favouringName` from `/open-items`) · `txtTheirRef` /
   `dateTheirRef` · `txtNarration` · `txtOurRef` · `lblStatus` pill.
2. **Payee summary line** from `/open-items` `summary` + `party` (group,
   bill by bill, TDS section + rate, PAN present / MISSING in red, cash
   discount rule, bills we owe, credits we hold, our PDCs out).
3. **Panels:** Last payments · Our cheques pending (`/party-context`).
4. **BILLS WE OWE:** `txtAmount` · `btnAutoAllocate` · `btnClear` ·
   `tblBills` (layout "PAYMENT - BILLS", id to issue): # · Bill date · Bill
   ref · Type chip (OPENING / PURCHASE / JOURNAL / INTEREST) · Due · Days ·
   Pending · PDC issued · **Pay** · **Disc recd** (seeded `discSuggested`,
   editable) · **R/off** · After · note; hidden keys. Empty for Customer /
   Other ledger kinds (the band collapses with a one-line reason).
5. **CREDITS WE HOLD:** `tblCredits` (shared delegate): date · ref · type
   chip (PURCHASE_RETURN / ADVANCE / OPENING(DR); for a Customer refund the
   customer's ADVANCE / SALES_RETURN items) · Held · **Apply** · Against
   bill · note. A payment with NO instruments that only applies a credit is
   refused on the client too ("that is a journal").
6. **INSTRUMENTS — money OUT:** `tblTenders` (layout "PAYMENT - TENDERS"):
   # · Tender (combo of masters with `allowInReturn`; CASH pops the drawer)
   · From (the master's ledger — our bank / cash, read-only) · Amount ·
   Mode / ref (BANK: NEFT/RTGS/IMPS/UPI + UTR, opens
   `PaymentBeneficiaryDialog`; CHEQUE: "leaf <next> (auto)" read-only,
   opens `PaymentChequeDialog`) · Beneficiary (name · account · IFSC, from
   the dialog; snapshot from `party.bank` by default) · Leaf / inst. date ·
   PDC chip · Voucher (after post) · note. The cheque row shows the leaf it
   WILL take, labelled "(auto)"; the number is never typed — the server
   takes it at post and the screen repaints the real one from the answer.
7. **OTHER LEDGERS:** `tblOther` (layout "PAYMENT - OTHER LINES"): Role
   (TDS_PAYABLE, DISCOUNT_RECEIVED [read-only Σ], BANK_CHARGES [seeded from
   MDR, read-only], ROUND_OFF, INTEREST_PAID, BALANCES_WRITTEN_BACK, any
   ledger) · Ledger · Dr/Cr · Amount · Settles bill · Against · note. TDS is
   seeded from `party.tdsRate` (PAN missing → 20 %, painted amber) on
   ALLOCATED + ON ACCOUNT, re-seeded whenever either moves unless the
   operator has overridden it (then a "manual" chip, and the override is
   kept until the party changes). Σ TDS this quarter is a hint label
   (`WHERE av_role = 'TDS_PAYABLE'` — a later report).
8. **Identity strip:** PAID + DEDUCTIONS + CREDITS APPLIED = ALLOCATED + ON
   ACCOUNT + ADDITIONS · DIFFERENCE. `VoucherIdentity` with Direction::Out:
   DEDUCTIONS = CR settling lines (TDS, discount received, balances written
   back, R/off CR); ADDITIONS = DR non-settling lines (interest paid, R/off
   DR); BANK_CHARGES excluded. Post disabled while ≠ 0. ON ACCOUNT > 0 shows
   "→ Advances to Suppliers + an ADVANCE bill" in the strip's note.
9. **Actions:** Save draft · **no Approve / Reject** (withdrawn 2026-09-15)
   · Post & Print · Edit header · Cancel payment · Ledger statement (F9,
   hidden until the view exists) · Cheque books (opens `ChequeBookDialog`)
   · List F8 · New · Close.
10. **Footer hint bar** as the mockup: Ctrl+Enter posts ·
    F8 list · Alt+A · Alt+T · Alt+L · F9 · no button takes focus.

**Page "List":** `NexApiTableView` on the payment grid; From / To, status,
payee kind, search; Enter loads with the row's own scope. PDC vouchers show
under their payment.

## 3. Behaviour — what differs from the receipt

- **Payee kind first.** Changing the kind clears the payee and the bands;
  Supplier → bills + credits; Customer → credits only (the customer's CR
  items), no TDS, no discount, ON ACCOUNT must be 0 (the strip says
  "refund = the credits applied; nothing else"); Other ledger → both bands
  collapsed, ONE implied leg = the payee for the gross (the operator types
  the amount in `txtAmount`; the instruments must equal gross − TDS), TDS
  seeded when the ledger is TDS-applicable.
- **Discount received** is seeded per bill only when `datePayment −
  billDate ≤ party.creditDays` — the client seeds, the operator decides,
  the server checks nothing about the rule (it only requires the
  DISCOUNT_RECEIVED line = Σ discounts sent).
- **Cheque row:** picking CHEQUE needs an active book of that tender's
  bank with free leaves — `GET /cheque-books/list?bank_ledger_id` on the
  fly; none → "no cheque book for Kvb Current A/c — open one" with the
  Cheque books button. Instrument date > payment date → PDC chip and "own
  voucher dated <date>" in the note; the strip still counts it as PAID.
- **Bank row:** beneficiary defaults from `party.bank`; UTR optional; the
  row's status chip "PENDING until UTR + bank date" after post (P9).
- **Post:** identity 0 → save draft if dirty → `POST /payments/post` with
  `allocations[]` (amount = money + TDS + credits per bill, discount and
  roundOff separate), `creditsApplied[]` (with `againstBillId` when
  pinned), `otherLineBills[]` (TDS pins), `writeoffApprovedBy`,
  `onAccount`. The answer carries the leaf numbers and `chb_next_leaf` —
  repaint the cheque rows, then print: the **payment advice** always, the
  **cheque(s)** when `accounts.cheque_print_after_post` (the print runner,
  §7; a cheque print also calls `POST /issued-cheques/print` so the
  register counts it).
- **Cancel:** confirm naming the PDC vouchers and the leaves that become
  VOID; the server's refusals (a cheque PRESENTED, a NEFT SETTLED, the
  ADVANCE spent) shown as is.
- **Load / Edit header:** as the receipt (no Approve / Reject — withdrawn).
- **Customer refund guard:** Apply > held refused in the cell; Post
  refused when `creditsApplied` is empty.

## 4. Settings read (`AppSession`):
`accounts.cheque_print_after_post`, `accounts.epayment_default_mode`,
`accounts.writeoff_approval_above`, `accounts.receipt_bill_sort` (reused for
the owed list's order). `accounts.tds_rates` is a server-side TABLE and is NOT read by the client —
`party.tdsRate` comes resolved.

## 5. Endpoint table

| call | when |
|---|---|
| `GET /payments/open-items?payee_kind&payee_id&company_id&on_date` | payee chosen; F5 |
| `GET /payments/party-context` | payee chosen |
| `GET /cheque-books/list?bank_ledger_id` | a CHEQUE row is added; the Cheque books dialog |
| `POST /cheque-books/create`, `/close` | the dialog |
| `POST /payments/create` | Save draft; first half of Post |
| `POST /payments/post` | Post |
| `POST /issued-cheques/print` | after a cheque print (count + log) |
| `GET /payments/get`, `/list` | load; page List |
| `PUT /payments/update-header`, `POST /payments/cancel` | POSTED actions |
| `GET /tender-masters/list` | once per open |

## 6. Tests

1. Kind = Supplier → both bands; Customer → credits only, TDS hidden;
Other ledger → bands collapsed, gross − TDS = instruments; 2. cash/bank
ledger pick refused with the Contra message; 3. discount seeded inside
credit days only; 4. TDS re-seeds on allocation change until overridden;
PAN missing → 20 % amber; 5. CHEQUE row with no book → the open-a-book
prompt; with a book → "leaf 000452 (auto)"; 6. PDC cheque → chip, counted
as PAID, note names the date; 7. identity: the worked example lands at
DIFFERENCE 0 with PAID 1,19,775 · DEDUCTIONS 225 · CREDITS 5,000 · ALLOCATED
1,25,000; 8. Post payload: pb00017 amount 19,900 + discount 100; 9. the
answer repaints the real leaf and voucher numbers; 10. refund > held refused;
11. Cancel confirm names the leaves; 12. List page + drill-through with the
row's own acc year.

## 7. The one shared prerequisite: a print runner

No client screen prints today (`quotation_entry.cpp` TODO). Every accounts
screen needs one: `src/shared/print/print_runner.h/.cpp` — `PrintRunner::
print(purposeCode, keys, parent)`: resolve the template
(`ApiEndpoints::PrintAssignments::Resolve` with company / branch / device /
purpose / output mode), fetch the dataset rows through the printing
module's data endpoint, render (HTML/QTextDocument for advices and slips,
a positioned-field renderer for the cheque's `chb_format` layout), then
`QPrinter` / PDF / preview per `ptaOutputMode`. Built once, before the
receipt is coded, used by receipt (voucher), cheques (slip), payment
(advice + cheque), issued cheques (cheque re-print). Its own small plan
when we start it; it is out of the five screen plans deliberately.
