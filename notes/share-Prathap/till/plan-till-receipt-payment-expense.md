# Till — bill-wise Receipt & Payment in the session, and Expense by tender (REV 1, 2026-10-07)

> **User asks, 2026-10-07:** *"bill wise payment and receipt also will include in till
> session and one more form we need to add thats expense by tender way"* — and *"in billwise
> pay and receipt all moved to acc_tender_detail"*.
>
> **A separate plan, per house rule.** `TILL_DESIGN.md` and `47_till_management.sql` are not
> edited. This file says which parts of them it overrides; `../48_till_receipt_payment_expense.sql`
> (run after 47) carries the schema side.
>
> Mockups: `till_expense_voucher_ui_mockup.png` (the new form) and
> `till_money_docs_in_session_ui_mockup.png` (F11 menu rev 2, Payment on a till, cockpit tender
> table rev 2).

## 0 · What this overrides

| In | Section | Now |
|---|---|---|
| TILL_DESIGN.md | §5.4 expected cash | **§2 here** — receipts, payments and expenses are terms of the formula, all read from `acc_tender_detail` |
| TILL_DESIGN.md | §5.5 money moves where the money moves | §2.3 adds receipt, payment, expense rows |
| TILL_DESIGN.md | §5.6 shipped rules | PAID_OUT → **EXPENSE**; new **CASH_PAYMENT** |
| TILL_DESIGN.md | §5.7 accounting (Paid-out row) | **removed** — an expense is an ExpV voucher (§4); a supplier/staff is a Payment |
| TILL_DESIGN.md | §7.4 changes to existing modules | §3 here (receipt, payment, voucher register) |
| TILL_DESIGN.md | §8 S2 F11 menu, S3 Paid out mode, S5 tender table | F11 rev 2, S3 loses Paid out, S5 gains Payments + Expenses columns (§6) |
| 47 | `till_cash_movement` kinds, `till_variance` split, rule/reason/event CHECKs | 48 §1–§6 |
| mockups | `till_cash_movement_ui_mockup.png` panel "Paid Out" | superseded by the Expense voucher mockup |

## 1 · The decision in one line

**Every rupee that crosses a till drawer is a row in `acc_tender_detail` carrying the session** —
sales, returns, order advances, re-tenders (already), and now **bill-wise receipts, bill-wise
payments and expense vouchers**. The till never needs to know which module wrote a row; it sums
rows by session, tender and source. Till-internal moves (float, pickup, drop, hand-over, paid-in)
stay in `till_cash_movement`.

Why **PAID_OUT goes**: with an Expense voucher and a bill-wise Payment both able to take cash out
of the drawer, a third way (a movement with a ledger) would let the same tea bill be booked three
ways and every expense report would have to union three sources. One path per fact:

| Cash out of the drawer for… | Document | Why |
|---|---|---|
| an expense (tea, courier, repair, electricity…) | **Expense voucher (ExpV)** — new | expense ledgers, optional GST bill, any tender |
| a supplier's bill, a COD delivery | **bill-wise Payment** (menu 100) | the party ledger and its bills must be settled |
| staff advance / salary advance | **Payment** to the staff ledger | it is a receivable from a person |
| sending cash to the safe / bank | till movement (DROP / PICKUP / REMIT) | unchanged |

`PAID_IN` stays (scrap sale, misc cash income without a party) — it is rare and has no party side.

## 2 · Expected cash — the formula, rev 2 (overrides TILL_DESIGN §5.4)

`TillLedgerService.expected(session, tender)` — one query over `acc_tender_detail`
(`td_session_id = S`, not voided, not deleted; index `ix_td_session` from 48) plus
`till_cash_movement`:

| term | rows | sign |
|---|---|---|
| open | `tss_float_counted` (cash only) | + |
| sales | `td_src_doc_type` SALE_BILL · SALES_ORDER (advance) DR | + |
| refunds | SALE_RETURN CR (and order-advance refunds) | − |
| **receipts** | **RECEIPT DR** (menu 99) and voucher-register receipt rows DR | + |
| **payments** | **PAYMENT CR** (menu 100) and voucher-register payment rows CR | − |
| **expenses** | **EXPENSE CR** (ExpV) | − |
| moved in | TOP_UP + PAID_IN | + |
| moved out | PICKUP + DROP | − |

`till_variance` gains `tvr_payment_amount` and `tvr_expense_amount`; `ck_tvr_expected` now reads
open + sales − refunds + receipts − payments − expenses + in − out (48 §2).

### 2.1 Which tenders are counted at close

Only rows of tenders whose `ttm_close_mode` is DENOM or SLIPS are counted. For **payments and
expenses** only the **CASH** rows can touch the drawer:

- a payment / expense by **bank transfer, UPI from the company account, cheque or company card**
  is a CR row on a non-drawer tender — it is listed in the cockpit as *"paid from bank —
  information"* and **never** enters a counted expectation (a card-slip count must not be reduced
  because the store paid a bill by NEFT);
- a **receipt** by card / UPI / cheque *does* enter that tender's expectation (the customer's card
  slip, cheque or UPI credit is real money taken at this counter).

### 2.2 The one refusal that keeps this formula true

On a till device in session, **a Journal (`Jrl`) or Contra (`Con`) with a leg on the till cash
ledger is refused** — `TILL_CASH_LEDGER_DIRECT`. Cash enters or leaves a drawer only through a
document that writes a tender row, or a till movement. (Back-office devices are unaffected.)

### 2.3 Money moves where the money moves — the new rows

| Document | Session it books into | Cash from / to |
|---|---|---|
| Receipt (menu 99) posted on a till device | the live session | drawer |
| Payment (menu 100) posted on a till device | the live session | drawer — **CASH_PAYMENT** approval over threshold |
| Expense voucher on a till device | the live session | drawer for the CASH rows |
| any of the three on a back-office device | none | the branch default **safe** (`till.backoffice_cash_from = SAFE`) → shows in the Cash Office safe book; or refused (`REFUSE`) |
| cancel of any of the three | allowed while its session is OPEN; after the session CLOSED → refused `TILL_SESSION_CLOSED` (the cash really moved — correct by a new document or an accounts journal, not by un-happening it) | |

## 3 · Receipt and Payment — what changes (they already write `acc_tender_detail`)

Server (both services already pass `avhSessionId` to the tender rows and legs —
`payment-posting.service.ts` 321/348/668, `voucher-register.service.ts` 552):

1. `TillSessionService.assertLive()` on `/receipts/post`, `/payments/post` and the voucher
   register's receipt/payment types when the request comes from a device with
   `till.money_docs_in_session` and any tender is CASH. Same refusals as the bill
   (`TILL_SESSION_REQUIRED` / `_NOT_OPEN` / `_NOT_YOURS` / `_WRONG_DEVICE`).
2. Stamp `avh_session_id` = the live session on the header (and so on legs + tender rows).
3. Payment, CASH tender, cash part over the rule threshold → `TILL_APPROVAL_REQUIRED`
   (CASH_PAYMENT; shipped 1,000.00, Supervisor, counter).
4. **40A(3)** — `StatutoryService.limit(company, 'CASH_PAYMENT_LIMIT_40A3', date)` (seeded 10,000,
   WARN): cash tenders of Payment + Expense vouchers to the same payee on the same date are summed;
   above it the post WARNs with the code, value and date (company override can make it REFUSE, or
   35,000 for goods-carriage hire).
5. **269ST** on receipts: cash received from one person in a day ≥ 2,00,000 — the existing
   `CASH_TXN_LIMIT_269ST` (REFUSE) now also guards `/receipts/post`, summing that party's cash
   receipts + cash sales of the day.
6. `till_event` RECEIPT_POSTED / PAYMENT_POSTED; `tss_receipt_count` / `tss_payment_count` frozen
   at close.

Client (Qt):

- `ReceiptEntry` / `PaymentEntry` send `avhSessionId = AppSession::sessionId()` (today they send
  nothing). They gain the **till strip** (S2). On a till device the CASH tender row reads
  *"Cash — drawer C01"*; on a back-office device *"Cash — safe S1"*.
- F11 rev 2 adds **Receipt (bill-wise)** and **Payment (bill-wise)** — they open menu 99 / 100 as
  a NEW entry over the bill (the re-tender pattern: the half-typed bill is not touched).

## 4 · The new form — Expense Voucher (ExpV), "expense by tender"

### 4.1 What it is

An expense paid by one or more **tenders**: several expense lines on the left, how it was paid on
the right, the two must be equal. On a till it is the cashier's petty-cash form (replaces PAID_OUT);
in the back office it is the accountant's quick expense entry.

3.0 had `EPOSGo/expenseentry` (`app_utils.fx_epos_expenses`): one ledger, one amount, one tender,
notes, and a delete. Kept from it: speed and the tender choice. Not carried: one line only, delete
instead of cancel, no GST, no approval, no session control.

### 4.2 Storage — no new table

| Part | Where |
|---|---|
| header | `acc_voucher_header`, type **ExpV** (48 §7): date = business date, `avh_session_id`, `avh_party_id` (optional supplier ledger), `avh_doc_refno` / `avh_doc_date` (supplier invoice), `avh_remarks` (paid to / notes), `avh_draft_lines` (draft payload) |
| expense lines | `acc_vouchers` legs DR, one per line: `av_ledger_id` (expense ledger), `av_cost_centre_id`, `av_remarks` (description); input-tax legs DR on the INPUT_CGST/SGST/IGST roles when a GST bill is claimed |
| tenders | `acc_tender_detail`, `td_src_doc_type = 'EXPENSE'`, `td_dr_cr = 'CR'`, `td_session_id`, ref no / last 4 / UTR / cheque no as for a payment; cheque rows go to the issued-cheque register like Payment's |
| GST bill | the GSTR2 register rows a PurA writes (`vchr_gst_register = GSTR2`, INPUT side) — supplier GSTIN, invoice no/date, place of supply, taxable, rate, ITC eligible |
| reason | quick-pick from `till_reason` category EXPENSE (its default ledger fills the line) |

### 4.3 Rules

- **Lines = tenders**, to the paisa (Decimal end to end). No round-off line; ROUND_OFF role only if
  the GST bill itself carries one.
- **GST bill** optional: tick *"With GST bill"* → supplier GSTIN (validated, GSTIN fetch button as
  on the masters), invoice no + date, POS; per line taxable + rate (`tax_rate_master`) + *ITC* tick.
  RCM, TDS and import of services are **not** in this form — use PurA / Payment.
  `accounts.expense_gst_bill_above` WARNs when a large expense has no bill.
- **Party** optional: when a supplier ledger is picked the expense is still paid now (no bill-wise
  outstanding) — for credit purchases use PurA + Payment.
- **Approvals:** EXPENSE rule on the **total** (shipped 500.00, Supervisor, counter); the cash part
  also counts toward 40A(3).
- **Session:** on a till device CASH rows are the drawer and need a live session; non-cash rows
  (bank, UPI-out, cheque, company card) are allowed and are information-only at close (§2.1).
- **Lifecycle:** DRAFT → POSTED → CANCELLED (house rule; no approve step). Cancel only while the
  session is OPEN (§2.3). The number is issued at post and never reused.

### 4.4 Routes — new family `/api/v1/expenses/*` (no existing route is called)

`POST /expenses/create` (draft) · `POST /expenses/validate` (always 200: derived legs, refusals,
warnings incl. 40A(3), approval needed) · `POST /expenses/post` · `GET /expenses/get` ·
`POST /expenses/cancel` · `GET /expenses/quick-reasons` (EXPENSE reasons + default ledgers) ·
`GET /expenses/ledger-pick` (expense-nature ledgers of the company + shared). Keys
`companyId · branchId · accYear · voucherId`. List = a registered grid "EXPENSE VOUCHERS" on
TxnMainView. Print = `EXPENSE_VOUCHER` template.

### 4.5 Errors

`EXPENSE_TOTAL_MISMATCH` (lines ≠ tenders) · `EXPENSE_LEDGER_NOT_EXPENSE` (a party or cash/bank
ledger as a line) · `EXPENSE_GST_INCOMPLETE` · `TILL_SESSION_REQUIRED` (cash on a till with no
session) · `TILL_SESSION_CLOSED` (cancel after close) · `TILL_APPROVAL_REQUIRED` (EXPENSE) ·
`STATUTORY_40A3` (WARN, or REFUSE by company override).

## 5 · Accounting

| Document | Dr | Cr |
|---|---|---|
| Receipt, cash at a counter | Till cash (session S) | customer ledger, bill-wise |
| Payment, cash at a counter | supplier / staff ledger, bill-wise | Till cash (session S) |
| Expense, cash at a counter | expense ledger(s) [+ input GST] | Till cash (session S) |
| Expense, UPI / bank / cheque | expense ledger(s) [+ input GST] | that bank ledger (no session effect at close) |
| Any of the three in the back office, cash | as above | **Safe** (default safe's ledger) — in the safe book |

The till-ledger invariant of TILL_DESIGN §5.7 is unchanged: the session's till-cash legs still net
to `tss_float_left` after close, because every one of these documents carries the session on its
legs.

## 6 · Screens

| Screen | Change |
|---|---|
| S2 F11 menu | rev 2: **1 Receipt (bill-wise) · 2 Payment (bill-wise) · 3 Expense** · 4 Paid in · 5 Drop · 6 No sale · 7 X read · 8 Suspend · 9 End shift · supervisor: Pickup · Top-up |
| Receipt (99) / Payment (100) | till strip; CASH row named "drawer C01" / "safe S1"; CASH_PAYMENT approval strip; 40A(3) warning strip |
| **Expense voucher** (new menu, own .ui on NexDialog) | header · lines grid · tenders grid · GST bill panel · approval strip · posting preview (mockup) |
| S3 Cash Movement | Paid out mode removed |
| S5 Cockpit | tender table gains **Payments** and **Expenses** columns, a *paid from bank (information)* row, and the Movements tab becomes **Money documents** (receipts, payments, expenses, movements) |
| S6 Sessions list | + Receipts / Payments / Expenses counts (optional columns) |
| S9 Safe book | back-office cash receipts/payments/expenses appear as safe rows |

## 7 · Build order (slots into TILL_DESIGN §13)

1. With phase 1: session stamp + `assertLive` on receipt/payment; expected formula rev 2.
2. With phase 2: Expense voucher (server + Qt form + grid + print), F11 rev 2, PAID_OUT removed.
3. With phase 3: CASH_PAYMENT and EXPENSE approvals; 40A(3) / 269ST checks.

## 8 · Validated (2026-10-07)

48 applied on the throwaway PG 18 cluster on top of 47 (live DDL of `acc_tender_detail`,
`acc_vouchers`, `statutory_limits` added): applies with `ON_ERROR_STOP`, re-runs with no change.
Seeds: ExpV with GSTR2/INPUT and its number series, CASH_PAYMENT rule, EXPENSE rule renamed,
11 EXPENSE reasons (2 retired: supplier, staff advance), 3 settings, the 40A(3) row. Probes:
a close variance with receipts 1,200 − payments 1,800 − expenses 250 adds up (11,920); refused —
an expectation that ignores a payment, an OPEN-stage row with an expense, a PAID_OUT movement /
rule / reason; allowed — PAID_IN with no safe, an EXPENSE_POSTED event. The 47 regression probes
still give their 10 refusals. **Nothing applied to 192.168.0.106; no NestJS or Qt code written.**
