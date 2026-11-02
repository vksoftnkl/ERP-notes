# Payment (menu 100) — Qt client plan, REVISION 2

**For Viji. 2026-09-28.** Supersedes `plan-qt-payment.md` (rev 1, 2026-09-14). Code once
`plan-backend-payment-rev2.md` is live.

**The rule (user, 2026-09-28): "do as like receipt".** The template is the Receipt screen AS
BUILT: `src/modules/accounts/receipt/` (≈7,900 lines) and `tests/receipt/` (four suites). It is
not rev 1, and **not the Voucher Register**: no bill-wise popup, no side lock, no party-first
grid. Wherever this file is silent, the payment does exactly what `ReceiptEntry` does, with the
money going out.

---

## 1. File set: the receipt's, renamed

| receipt | payment |
|---|---|
| `receipt_entry.h/.cpp/.ui` | `payment_entry.h/.cpp/.ui` (`PaymentEntry : NexDialog`) |
| `receipt_entity.h` | `payment_entity.h` (`PaymentHeader`, `OwedBill`, `HeldCredit`, `TenderRow` + `chequeBookId / favouring / acPayee / beneficiary`, `OtherLine`, `PaymentRoles`) |
| `receipt_identity.h` | `payment_identity.h` (the same equation, direction OUT) |
| `receipt_allocation_engine.h/.cpp` | **shared**: give it `Direction { In, Out }` rather than copying 226 lines |
| `receipt_bill_delegate.*`, `receipt_tender_delegate.*`, `receipt_delegate_base.*` | `payment_bill_delegate.*`, `payment_tender_delegate.*`; **the base is reused** |
| `receipt_cheque_dialog.*` | `payment_cheque_dialog.*`: OUR cheque. Book (the open books of the tender's bank, from `/vouchers/cheque-books`), the next leaf read-only "(auto)", date, favouring, A/c payee. Beneficiary for a BANK row. EnterNavigation, no default button |
| `ApiEndpoints::Receipts` | `ApiEndpoints::Payments`: the same 11 routes under `/payments`, the same avh-prefixed `VoucherKey` |
| `TxnDocTypes::receipt()` (grid 108) | `TxnDocTypes::payment()` (the new list grid), registered on menu **100** the way 99 is (`openForMenu`, `txnEntryFirst`) |
| `tests/receipt/{run,run-post-new,run-features,run-full}.sh` | `tests/payment/…` the same four suites, mirrored |

## 2. The screen: the receipt's bands, what changes on each

| band (receipt) | payment |
|---|---|
| Header: no, date, beat → customer (dd 13 → 54), collected by (38), their ref / date, narration | **payee**: any party ledger (dd 43 filtered to party groups, or a supplier dropdown; decide at build), no beat. **Paid by** in place of collected by. + **Favouring** (default `party.favouringName`) |
| Party band: mobile/address, TOTAL BALANCE / CHEQUES OUT / AFTER THIS RECEIPT | the same from `/payments/party-context`: TOTAL WE OWE / OUR CHEQUES OUT / AFTER THIS PAYMENT; TDS section + rate + source (NO_PAN in amber) |
| Bills grid (credits first, then bills; ui_table 32) | the same grid: credits = DR items we hold (advance paid, debit note); bills = CR bills we owe. Columns: Pay, **Disc recd** (seeded by slab), Write-off, R/off. New ui_table "PAYMENT - BILLS" |
| Tenders grid (instruments then other-line roles; ui_table 33) | the same. CHEQUE row → payment cheque dialog (book + auto leaf); BANK row → beneficiary. Roles: TDS_PAYABLE (seeded, override chip), BANK_CHARGES (from MDR), INTEREST_PAID, BALANCES_WRITTEN_BACK, any ledger. New ui_table "PAYMENT - TENDERS" |
| Identity strip, balanced tone | the same equation: PAID + DEDUCTIONS + CREDITS = ALLOCATED + ON ACCOUNT + ADDITIONS |
| Last receipts / cheques-out panels | last payments / our cheques out |

## 3. Behaviour: the receipt's, with these deltas only

- **Money-ledger payee** refused at the pick: "use Contra".
- **TDS** seeded from `party.tdsRate` on allocated + on account, and re-seeded as they move
  unless overridden. The server re-seeds and refuses a disagreement (show its sentence).
- **Cheque row:** no open book for that bank → "no cheque book for <bank> — open one" (menu 263).
  The leaf is shown "(auto)" and never typed; after post, repaint the real leaf from
  `data.cheques[]`.
- **Post-dated cheque** chip and note, exactly as the receipt's PDC.
- **Discount/write-off/round-off travel on the allocation only** (`linesThatTravel()`, the
  receipt's lesson).
- **On account** → the confirm "keep it on account? It becomes an advance to <supplier>", as the
  receipt's.
- **Everything else is the receipt's, as built:**
  - draft remembers allocations with the clamp on reopen (notes 30);
  - posted amend behind the setting (R20);
  - delete-draft vs cancel;
  - Ctrl+PgUp/PgDn;
  - the duplicate chip;
  - Problem → focus the cell;
  - F7 paid detail;
  - Alt+A auto allocate;
  - the F8 picker;
  - scope captured once.
- **Popups** follow the house rule (EnterNavigation: Enter moves on, no default button, keypad
  Enter included).

## 4. Tests: the receipt's four suites, OUT

Each is posted for real and then cancelled or deleted, as the receipt's are:

- `run`: layout, one bill by NEFT;
- `run-post-new`: draft → post → reopen, delete draft;
- `run-features`: TDS seed / override / NO_PAN, discount slab, credit applied, on account →
  advance, PDC voucher, duplicate chip, adjacent;
- `run-full`: DB assertions per row written (legs, abj DR rows, ADVANCE bill, apd 'P' row +
  leaf + book next leaf), cancel reverses and returns nothing to the book, amend.

## 5. Order

1. Backend live (rev-2 §1–§5).
2. Endpoints + entity.
3. Clone the screen.
4. Deltas (§3).
5. Registrations (ui_tables, menu 100).
6. The four suites.
