# Payment voucher — the complete flow, table by table, column by column

**Revision 1, 2026-09-14** — the mirror of `../receipt/receipt_flow.md` rev 3,
read against the 3.0 screens (`NextLvl/3.0/pos/Accounts/supplierpayment.cpp`
"Bill wise Payment", `issuedcheques.cpp`, `financialvoucherentry.ui`),
TallyPrime's Payment (F5: bank allocations, cheque range, favouring name,
post-dated, cheque printing, payment advice, e-payments, auto TDS) and
GoFrugal's payables (cheque preparation → print → issue → reconcile, all
transfer modes, TDS, debit notes adjusted). On the user's decision:
ONE screen for every single-payee payment, ONE phase, no multi-ledger
voucher, no batch. Every table, column, CHECK and index below was read off
the live database at 192.168.0.106/ERP on 2026-09-14; where this text says
NEW the DDL is in `plan-backend-payment.md` §2.

**On row counts:** a TEST database. Its SCHEMA is authoritative; its DATA
only shows the shape of a thing. `acc_bill_balance` live has only SALES DR
rows — no purchase module yet — so the first bills this screen settles are
the OPENING CR rows `../openingbal/` creates for creditors.

Companion files — `README.md`. The receipt this mirrors is `../receipt/`;
the cheque register it shares is `../cheques/`. **What is NOT restated here
is inherited from the receipt flow:** the allocation engine (§4.4 there), the
post-dated model (§5 there), the header-totals trigger and the bill-balance
trigger (receipt plan §2.1 / §2.4). Where the payment differs, this file says
so; where it says "as the receipt", it means exactly that with the signs
flipped.

**The whole thing in one line:** a payment is money paid to ONE payee —
a supplier (bill-wise), a customer (a refund of a credit we hold), or any
other ledger (rent, salary, a tax remittance) — split across instruments,
allocated due-date-first against the bills we owe and the credits we hold,
every non-money line (TDS we deduct, discount we receive, bank charges,
interest, round-off) a role-tagged leg, the excess held as an ADVANCE we can
spend on a later bill — one voucher for today's money, one more per
post-dated cheque dated the cheque, our cheques tracked leaf by leaf.

---

## 0. The tables involved

| table | partitioned (LIVE) | written by a payment | role |
|---|---|---|---|
| `accounts.acc_voucher_types` | no | no — needs one ROW (`Pmt`) | the type |
| `accounts.acc_voucher_seq` | no | bump | number series per type / branch / device |
| `accounts.acc_voucher_header` | yes | **yes — one, plus one per post-dated cheque** | the payment |
| `accounts.acc_vouchers` | yes | **yes** | the legs; `av_role` tags the non-money ones |
| `accounts.acc_tender_detail` | yes | **yes** — `td_dr_cr = 'CR'` | one row per instrument; the beneficiary snapshot (NEW columns) |
| `accounts.acc_bill_balance` | yes | ADVANCE (DR) row for on-account | CR bills settled; DR credits applied |
| `accounts.acc_bill_adjustment` | yes | **yes** — `abj_dr_cr = 'DR'` on a CR bill | every settlement event; the credit pairs |
| `accounts.acc_pdc_register` | yes | **yes** — `apd_tra_type = 'P'` | one row per cheque WE issue |
| `accounts.acc_cheque_book` | **NEW** | `chb_next_leaf` advanced | the leaf the cheque takes |
| `purchase.suppliers` | no | **no change needed** | `sup_id` IS `led_id` — no bridge (2026-09-15) |
| `sales.customers` | no | **no change needed** | `cus_id` IS `led_id` — no bridge (2026-09-15) |
| `accounts.acc_ledger_master` | no | no | payee; `led_is_tds_applicable`, `led_tds_deductee_type`, `led_tds_nature_of_payment`, `led_pan_no` drive TDS |
| `accounts.acc_ledger_bank_accounts` | no | no | the beneficiary's bank details (exists live) |
| `accounts.acc_ledger_role` / `acc_ledger_map` | no | no — needs four ROWS | `DISCOUNT_RECEIVED`, `INTEREST_PAID`, `BALANCES_WRITTEN_BACK`, `ADVANCE_PAID` (NEW) |
| `public.txn_status_log` | no | one row per state change | the trail |

`acc_voucher_cheques` is retired (receipt R1) — nothing here writes it.

---

## 1. What the receipt design already settled — and this one inherits

| fact | where |
|---|---|
| `acc_bill_balance` and `acc_pdc_register` are partitioned live | receipt flow §1.1 / §1.2 |
| the bill-balance trigger is not deployed; the plan's `fn_abl_recompute` + `fn_abl_regularise_pdc` replace it | receipt plan §2.1 |
| header totals derived from legs by trigger | receipt plan §2.4 |
| `av_role` on `acc_vouchers`; `avh_draft_lines` on the header | receipt plan §2.6 |
| settlement modes `TDS`, `CLAIM` added to `ck_abj_mode` | receipt plan §2.5 |
| one cheque table; a post-dated cheque is its own voucher dated the cheque; bills settle on maturity | receipt R1 / §5 |
| ~~approval~~ | **withdrawn 2026-09-15** — DRAFT → POSTED → CANCELLED |
| lowercase dotted settings under `accounts.*` | receipt plan §2.8 |

If `../receipt/plan-backend-receipt.md` §2 has not been run, nothing here can
be. This design adds to it; it does not repeat it.

---

## 2. `acc_voucher_types` — one row

| column | value |
|---|---|
| `vt_code` | `Pmt` |
| `vt_name` | Payment |
| `vt_nature` | `PAYMENT` (the enum already has it) |
| `vt_is_bank_voucher` | true |
| `vt_needs_party` | true — every payment has a payee, even the "other ledger" kind |
| `vt_prefix` / numbering | as the receipt's, per branch / device through `acc_voucher_seq` |

`ChqBnc` (JOURNAL, from the cheques plan) is reused for a cheque of ours
returned unpaid. No `Contra` type is created here (P15) — a cash-to-bank
movement is menu 104's, later.

---

## 3. `acc_voucher_header` — one row per payment, one more per post-dated cheque

Exactly the receipt's §3 with `avh_voucher_type_id` = `Pmt`. The columns
that carry payment-specific meaning:

| column | value |
|---|---|
| `avh_party_id` | the payee's ledger — which is the payee id itself, supplier / customer / ledger alike (2026-09-15) |
| `avh_doc_amount` | Σ instruments (money OUT) — derived, never typed |
| `avh_adjust_amount` | Σ settling rows of THIS voucher (money + TDS + discount + credits) |
| `avh_doc_refno` / `avh_doc_date` | THEIR reference — supplier statement / advice no; `ux_avh_doc_refno` once per party per year |
| `avh_employee_id[1]` | who released / handed over |
| `avh_remarks` | narration; `avh_usr_refno` our own reference |
| `avh_draft_lines` | the other-ledger lines while DRAFT (receipt §2.6) |
| `avh_against_voucher_id` | on a PDC voucher → the payment; on a reversal → the original |
| `avh_src_module / _doc_type / _doc_id` | NULL — a payment is its own source |

Status walk: **DRAFT → POSTED → CANCELLED** (approval withdrawn 2026-09-15).
`ck_avh_no` fires at Post — a DRAFT never holds a number.

---

## 4. The payment, table by table — the worked example

Balu Traders (Sundry Creditors, bill by bill, TDS 194Q 0.1 %, PAN present,
cash discount 0.5 % within 7 days). Bills we owe: PB/2025/0088 OPENING(CR)
60,000 due 22-03; PB/2025/0102 OPENING(CR) 45,000 due 14-04; pb00017 PURCHASE
20,000 dated 08-09 due 08-10. We hold DN/2026/0007 PURCHASE_RETURN 5,000.
Paid 14-09-2026: NEFT 74,775 (bank charge 5) + our cheque 000451 for 45,000
dated 20-09.

The identity the screen and the server both enforce, to the paisa:

    PAID 1,19,775 + DEDUCTIONS 225 + CREDITS APPLIED 5,000
      = ALLOCATED 1,25,000 + ON ACCOUNT 0 + ADDITIONS 0

DEDUCTIONS = TDS 125 (75 on 0088 + 50 on pb00017) + discount received 100.
Bank charges 5 are NOT in the identity — they are the bank's cut of OUR money,
a split of the bank leg (receipt §4.5's rule, same reasoning).

### 4.1 `acc_tender_detail` — one row per instrument, `td_dr_cr = 'CR'`

| column | row 1 | row 2 |
|---|---|---|
| `td_src_doc_type` | `PAYMENT` (already in `ck_td_src_doc_type`) | `PAYMENT` |
| `td_src_doc_id` / `td_src_acc_year` | the payment header | the payment header |
| `td_row_no` | 1 | 2 |
| `td_tender_id` / `td_tender_type_id` | the BANK tender master (`ttm_allow_in_return` = true) | the CHEQUE tender master |
| `td_tender_ledger_id` | Kvb Current A/c — OUR bank, from the master | Kvb Current A/c |
| `td_dr_cr` | **CR** — money out | **CR** |
| `td_amount` | 74,775 | 45,000 |
| `td_mdr_amt` | 5 — the transfer charge; becomes the `BANK_CHARGES` split | 0 |
| `td_ref_no` | UTR — **may be NULL at post** (P9) | leaf no `000451` |
| `td_instrument_date` / `td_is_pdc` | — | 20-09-2026 / true (COMPUTED: instrument date > voucher date) |
| `td_bank_name` | — | KVB (ours) |
| `td_settle_status` | **PENDING** until UTR + bank date are written (P9) | SETTLED at issue |
| `td_beneficiary_name` (NEW) | Balu Traders & Co — snapshot of `sup_cheque_pre_name` / `acc_ledger_bank_accounts` | Balu Traders & Co (the favouring name) |
| `td_beneficiary_account` (NEW) | 347600231 | — |
| `td_beneficiary_ifsc` (NEW) | SBIN0012809 | — |
| `td_voucher_id` | pmt00009 | pmt00010 (the PDC voucher) |

The beneficiary trio is a SNAPSHOT: the e-payment file the bank takes needs
what was true on the day, not what the ledger says next month. Source is
`acc_ledger_bank_accounts` for the payee ledger (default row), editable on the
row for a one-off. `td_beneficiary_*` are the whole e-payment hook — the
export screen is a later plan and reads these columns and `td_settle_status`.

### 4.2 `acc_vouchers` — the legs, and `av_role`

One DR party per voucher for what it settles; one CR per instrument; one leg
per other-ledger line; the credit pair's ledger half.

**Voucher 1 — pmt00009, dated 14-09 (today's money):**

| # | ledger | Dr/Cr | amount | `av_role` | note |
|---|---|---|---|---|---|
| 1 | Balu Traders | DR | 80,000 | — | bills 0088 + pb00017: NEFT 74,775 + TDS 125 + discount 100 + DN 5,000 |
| 2 | Kvb Current A/c | CR | 74,780 | — | what actually leaves the bank: 74,775 + its charge 5 |
| 3 | Bank Charges | DR | 5 | `BANK_CHARGES` | the split of leg 2 — outside the identity |
| 4 | TDS Payable | CR | 125 | `TDS_PAYABLE` | deducted by US; remitted to the government later (Form 26Q is `WHERE av_role = 'TDS_PAYABLE'`) |
| 5 | Discount Received | CR | 100 | `DISCOUNT_RECEIVED` | seeded from `sup_cash_disc_perc` on pb00017 |
| 6 | Purchase Return (the debit note's holding ledger) | CR | 5,000 | `NOTE_ADJUST` | the credit we held is spent; the party is debited inside leg 1 |

Σ DR = 80,005 = Σ CR (74,780 + 125 + 100 + 5,000). Leg 6 is the ledger half
of the credit PAIR (§4.4): a credit's value sits in a HOLDING ledger until it
is applied — the debit note's ledger for a note, **`ADVANCE_PAID`** ("Advances
to Suppliers", NEW asset role) for an advance we paid — and applying it moves
it onto the party: DR party / CR holding. `ck_av_self` is why the pair is
never DR party / CR party. This is the receipt's row 7 with the signs flipped
(`../receipt/receipt_flow.md` §4.2, corrected today).

**Voucher 2 — pmt00010, dated 20-09, `avh_against_voucher_id` = pmt00009
(our post-dated cheque):**

| # | ledger | Dr/Cr | amount | `av_role` |
|---|---|---|---|---|
| 1 | Balu Traders | DR | 45,000 | — |
| 2 | Kvb Current A/c | CR | 45,000 | — |

**If ON ACCOUNT had been > 0** (we paid more than we owe): one more leg on
the voucher that carries the excess — DR Advances to Suppliers
(`ADVANCE_PAID`) for the excess, and the party leg smaller by the same — plus
the ADVANCE (DR) bill of §4.3. The purchase bill's adjust panel spends it
later with the mirror pair (DR party / CR Advances to Suppliers).

**"Other ledger" payee (lane B):** no bills, no credits; leg 1 is DR the
payee ledger (rent, salary, GST payable …) for the gross; TDS Payable CR when
the ledger is TDS-applicable; CR bank/cash for the net. Same shape, no `abj`
rows.

**Customer refund (lane C):** the credit LEAVES. Legs: DR the holding
ledger the credit sits in (`ADVANCE_RECEIVED` "Customer Advances Received"
for an advance, the credit note's liability ledger for a SALES_RETURN note)
/ CR bank or cash. The customer's own ledger does not move — it was never
credited for the holding (receipt §4.2). One `abj` row closes the customer's
CR item (`abj_dr_cr = 'DR'`, `abj_adj_type = 'ALLOCATION'`, mode by
instrument) with `abj_voucher_id` = this payment (§4.4). No TDS ever on a
refund; no discount; no on-account — a refund larger than the credit is
refused.

`av_opp_ledger_id`: the party on each instrument / other leg; NULL on the
party leg when more than one thing faces it. `av_recon_date`: written on the
bank leg by Issued Cheques → Presented (bank date) or by the e-payment
confirmation — the bank-book hook, not written at post.

### 4.3 `acc_bill_balance` — bills settled, credits applied, credit created

Read side (what the screen lists for a supplier):

```sql
-- BILLS WE OWE: the CR side of the party, open, across years
SELECT … FROM accounts.acc_bill_balance
 WHERE abl_party_id = :party AND abl_company_id = :company
   AND abl_dr_cr = 'CR' AND abl_status IN ('OPEN','PARTIAL') AND NOT abl_is_deleted
   AND abl_bill_type IN ('PURCHASE','OPENING','JOURNAL','INTEREST')
 ORDER BY abl_due_date NULLS LAST, abl_doc_date;          -- accounts.receipt_bill_sort reused
-- CREDITS WE HOLD: the DR side
   … abl_dr_cr = 'DR' AND abl_bill_type IN ('PURCHASE_RETURN','ADVANCE','OPENING')
```

No branch filter (`abl_branch_id` is where the bill was raised; the payment
settles it from anywhere — the receipt's rule). `abl_acc_year` travels with
every row because the table is partitioned.

Write side after the example posts (`fn_abl_recompute` moves the caches):

| bill | `abl_alloc_amount` | `abl_disc_amount` | pending | status | when |
|---|---|---|---|---|---|
| PB/2025/0088 | 60,000 (54,925 money + 75 TDS + 5,000 note) | 0 | 0 | CLOSED | 14-09 |
| pb00017 | 19,900 (19,850 money + 50 TDS) | 100 | 0 | CLOSED | 14-09 |
| PB/2025/0102 | 0 until 20-09, then 45,000 | 0 | 45,000 → 0 | OPEN → CLOSED | matures 20-09 (`fn_abl_regularise_pdc`) |
| DN/2026/0007 | 5,000 | 0 | 0 | CLOSED | 14-09 |

Credit created — only when ON ACCOUNT > 0: one row, `abl_bill_type =
'ADVANCE'`, `abl_dr_cr = 'DR'` (we hold a claim on the supplier),
`abl_src_module 'ACCOUNTS'`, `abl_src_doc_type 'PAYMENT'`, `abl_src_doc_id`
= the voucher carrying the excess, `abl_voucher_id` the same, `abl_branch_id`
= the payment's branch (NOT NULL), `abl_doc_refno` = the payment no,
`abl_doc_date` = that voucher's date (the cheque date for a PDC's excess).

For a customer refund the read side flips: list the customer's **DR-side
credits only** — `abl_dr_cr = 'CR'` on the customer (ADVANCE received,
SALES_RETURN credit note) with pending > 0 and **not already applied**; the
payment closes them. Refused against an open DR bill (P12).

### 4.4 `acc_bill_adjustment` — every settlement event

The receipt's §4.4 with `abj_dr_cr = 'DR'` on a CR bill. One MIXED row per
bill for instant money, one row per cheque per bill (`abj_cheque_id`,
`abj_is_post_dated`), one row per settling deduction per bill, one PAIR per
credit applied.

| # | bill | type | mode | amount | cheque | post-dated | against | voucher | date |
|---|---|---|---|---|---|---|---|---|---|
| 1 | 0088 | ALLOCATION | MIXED | 54,925 | — | f | — | pmt00009 | 14-09 |
| 2 | 0088 | ALLOCATION | TDS | 75 | — | f | — | pmt00009 | 14-09 |
| 3 | 0088 | NOTE_ADJUST | — | 5,000 | — | f | DN/2026/0007 | pmt00009 | 14-09 |
| 4 | DN/2026/0007 | NOTE_ADJUST | — | 5,000 | — | f | 0088 | pmt00009 | 14-09 |
| 5 | pb00017 | ALLOCATION | MIXED | 19,850 | — | f | — | pmt00009 | 14-09 |
| 6 | pb00017 | ALLOCATION | TDS | 50 | — | f | — | pmt00009 | 14-09 |
| 7 | pb00017 | DISCOUNT | — | 100 | — | f | — | pmt00009 | 14-09 |
| 8 | 0102 | ALLOCATION | CHEQUE | 45,000 | 000451 | **t** | — | pmt00010 | **20-09** |

Rows 1–7 `abj_dr_cr = 'DR'` (a CR bill is settled by a DR movement); row 4
`'CR'` (the DR credit is relieved by a CR movement). Row 8 waits for maturity
— `fn_abl_recompute` excludes post-dated rows dated after today; the daily
`fn_abl_regularise_pdc` counts it on 20-09. TDS rows are pinned to a bill by
the operator or spread pro-rata by the engine (`otherLineBills[]`).

### 4.5 The OTHER LEDGERS band — every non-money line is a role

| role | side | seeded from | settles the bill? | in the identity as |
|---|---|---|---|---|
| `TDS_PAYABLE` | CR | ledger's `led_tds_deductee_type` + `led_tds_nature_of_payment` → the **`accounts.tds_rates` TABLE** (plan §2.9, effective-dated); **PAN missing → 20 %** (206AA); on the on-account part too (P5) | yes — mode TDS | DEDUCTION |
| `DISCOUNT_RECEIVED` (NEW) | CR | Σ of the bill grid's Disc recd column (pre-filled from `sup_cash_disc_perc` when paid within `sup_credit_days`); read-only in this band | yes — DISCOUNT rows | DEDUCTION |
| `ROUND_OFF` | DR or CR | Σ of the bill grid's R/off column | yes | DEDUCTION or ADDITION by sign |
| `BANK_CHARGES` | DR | `td_mdr_amt` of every bank row | **no** — the split of the bank leg | outside |
| `INTEREST_PAID` (NEW) | DR | typed | no — a new INTEREST bill is the supplier's to raise; here it is an addition on top | ADDITION |
| `BALANCES_WRITTEN_BACK` (NEW) | CR | typed — a small creditor balance we will never pay | yes — WRITEOFF rows, `approvedBy` per `accounts.writeoff_approval_above` | DEDUCTION |
| any ledger | either | typed; role NULL, `av_role` NULL | operator's flag | by side |

`ADVANCE_PAID` is not a band line — it is the on-account leg the server writes.

### 4.6 `acc_pdc_register` — one row per cheque WE issue

Same table, same CHECKs as the received side, read the other way:

| column | value |
|---|---|
| `apd_tra_type` | **'P'** |
| `apd_instrument_no` | the LEAF number, taken from `acc_cheque_book` at post — never typed |
| `apd_instrument_date` | the cheque date; post-dated when > payment date |
| `apd_drawer_name` | the FAVOURING name on this side (`td_beneficiary_name`) |
| `apd_bank_ledger_id` | OUR bank — set at ISSUE (the received side sets it at deposit) |
| `apd_party_id` | the payee ledger |
| `apd_tender_id` | the CHEQUE tender row |
| `apd_voucher_id` | the payment / PDC voucher — always set at post (P3) |
| `apd_posting_mode` | ON_RECEIPT always — an issued cheque credits the bank the day it is written; no setting on this side |
| `apd_status` | HELD = issued, not presented (the register shows it as ISSUED) |
| `apd_printed_on` / `apd_print_count` (NEW) | NULL / 0 until Print |
| `apd_clear_date`, `apd_clear_voucher_id` | Presented: bank date / **stays NULL** — see P11b |
| `apd_bounce_*`, `apd_charge_voucher_id` | Returned unpaid — the `ChqBnc` voucher |
| `apd_replaced_by_id` | Stop → Replace |

`ck_apd_cleared` live demands `apd_clear_voucher_id IS NOT NULL` at CLEARED.
Presenting OUR cheque posts nothing (the bank was credited at issue), so
**P11b:** loosen the CHECK to R rows (`plan-backend-payment.md` §2.6). The
payment voucher is already in `apd_voucher_id`.

---

## 5. Cheques — leaves, post-dated, printing

### 5.1 `acc_cheque_book` (NEW) — one row per physical book

`chb_id`, `chb_company_id`, `chb_bank_ledger_id` → BANK ledger,
`chb_book_no` (the bank's series), `chb_leaf_from` / `chb_leaf_to` (fixed
width, numeric compare), `chb_next_leaf`, `chb_format` (the printing-schema
layout for this bank / book), `chb_is_active`, audit. `ux_chb_bank_book`,
`ck_chb_range` (from ≤ next ≤ to + 1). Full DDL in the plan §2.4.

**Post takes the next leaf inside the payment's transaction:** `SELECT
chb_next_leaf FROM acc_cheque_book WHERE … FOR UPDATE`, write it into the
register row and `td_ref_no`, advance `chb_next_leaf`. A leaf is CONSUMED by
an issue or a void and never given back — a spoilt print is a VOID register
row (amount 0, `apd_status = 'CANCELLED'`, no voucher), a lost cheque is STOP
+ Replace. `chb_next_leaf = chb_leaf_to + 1` means the book is finished; the
screen says so and the operator opens the next book (`cheque-books/create`).
An operator may pick a DIFFERENT active book of the same bank on the
instrument row; the number itself is never typed.

### 5.2 Post-dated — the receipt's §5.2 exactly

Instrument date > payment date ⇒ its own header dated the cheque
(`avh_against_voucher_id` = the payment), its own legs (DR party / CR bank),
its own `abj` rows (`abj_is_post_dated`), and the bills it settles show
"PDC issued" until maturity. `avh_voucher_date` of the PDC voucher must be
in an OPEN, unlocked year. ~~Approval (P10)~~ withdrawn; this covered the payment AND its PDC
vouchers in one act.

### 5.3 Printing — P13

Post & Print prints the **payment advice** always (one dataset, the header +
bills settled + instruments) and the **cheque** for every CHEQUE row on the
bank's format (`chb_format`) — the printing schema (17), nothing
cheque-specific in this screen. `accounts.cheque_print_after_post` (default
true) decides whether the cheque print opens by itself. Print writes
`apd_printed_on = now()`, `apd_print_count + 1`, a `txn_status_log` row
'PRINTED'; a re-print is counted, never silent.

### 5.4 The register's actions — Issued Cheques, menu 52

The mirror of `../cheques/received_cheques_flow.md` §3. One register update +
one `txn_status_log` row per transition; a voucher only where money moves.

| action | from → to | writes |
|---|---|---|
| **Print** (F6) | no status change | §5.3 |
| **Presented** (F7) | HELD → CLEARED (shown PRESENTED) | `apd_clear_date` = bank date, `apd_remarks`; **`av_recon_date`** = bank date on the payment's bank leg (the bank-book hook); NO voucher; `apd_clear_voucher_id` stays NULL (P11b) |
| **Returned unpaid** (F8) | HELD → BOUNCED (shown RETURNED) | ONE `ChqBnc` voucher: CR party (the amount — we owe them again) / DR bank (the debit reversed) + DR `BANK_CHARGES` / CR bank for our bank's charge; negative `abj` rows for every row with `abj_cheque_id` = this (the bills REOPEN); an ADVANCE (DR) bill created from this cheque's excess is reversed with the C4 cascade mirrored (anything that spent it is un-spent, then the ADVANCE soft-deleted); `apd_bounce_*`, `apd_bounce_voucher_id = apd_charge_voucher_id` = the ChqBnc; the leaf stays consumed. "Charge to party" is n/a — OUR bank refused OUR cheque |
| **Stop** (F3, had a payment) | HELD → CANCELLED (shown STOPPED) | reversal voucher (`Pmt`, mirrored legs, `avh_against_voucher_id` back), negative `abj` rows, ADVANCE undone; refused once PRESENTED |
| **Void** (F3, blank spoilt) | a NEW register row: amount 0, no payment, no voucher, CANCELLED | the leaf is consumed; the book's next moves on |
| **Replace** | STOPPED or RETURNED → REPLACED + a new HELD row | the new row takes the book's next leaf, `apd_replaced_by_id` on the old; a NEW voucher `Pmt` DR party / CR bank (dated the new cheque when post-dated) with fresh allocations through the engine against the bills the old one settled (or on account) |
| **History** (Ctrl+H) | — | `txn_status_log` for the leaf |

Due buckets computed as the received side's (FUTURE / DUE TODAY / OVERDUE /
STALE +3 months). The summary tie: **bank book uncleared = ISSUED, NOT
PRESENTED + POST-DATED, OUT.**

---

## 6. Cancel, header edit — money is never edited

As the receipt's §6: a POSTED payment is cancelled by a reversal per voucher
(the payment AND each PDC voucher — `Pmt`, mirrored legs including `av_role`,
one negative row per `abj` row, ADVANCE bills soft-deleted, refused if
spent), register rows → CANCELLED (the leaf VOID — never re-used), tender rows
soft-deleted, originals → CANCELLED with the reason and
`avh_reversal_voucher_id`. **Refused** when any issued cheque is PRESENTED or
later, when the NEFT is SETTLED (UTR confirmed) — stop / return the instrument
first — or when the on-account ADVANCE has been spent.

Header-only edit on a POSTED payment: `avh_remarks`, `avh_usr_refno`,
`avh_doc_refno`, `avh_doc_date`, `avh_employee_id`, with `editRemark` to the
audit trail — nothing else, a 400 on anything else.

~~Approval (P10)~~: withdrawn 2026-09-15. Post runs from a DRAFT. The
paragraph that stood here described DRAFT → APPROVED → POSTED. Reject returned it
to DRAFT with the reason. Off: Post does both.

---

## 7. Rules the DATABASE enforces

`ck_avh_no`, `ck_avh_balanced` (on derived totals, receipt plan §2.4),
`ck_av_self`, `ck_td_src_doc_type` (PAYMENT is in it), `ck_abl_settled`,
`ck_abj_against`, `ck_abj_mode` (+TDS, CLAIM), `ck_abj_cheque_mode` /
`ck_abj_pdc`, the `ck_apd_*` lifecycle (with the P11b change), `ux_apd_*`
(one register row per leaf per bank), `ux_chb_bank_book`, `ck_chb_range`,
`ux_avh_doc_refno` (their reference once per party per year), the FK from
the party id (already the ledger id), partition keys everywhere
(`*_acc_year` NOT NULL).

## 8. Rules ONLY the service can enforce

The identity (§4); per-bill `amount + discount + writeoff ≤ pending` NOW
under a row lock; a credit applied `≤ pending` NOW; the leaf taken under a
row lock on the book; TDS present when the ledger is TDS-applicable (server
re-seeds and refuses a disagreement); PAN-missing rate; a cash/bank ledger
refused as payee ("use Contra"); a customer refund only against unapplied CR
items; a payment with no instruments refused ("that is a journal"); every
date in an OPEN, unlocked year; `writeoffApprovedBy` above the threshold;
Cancel's refusals (§6); Presented / Stop refusals by state; the number
allocator inside the transaction.

## 9. Deferred, with the hook

| deferred | hook that exists |
|---|---|
| e-payment bank file export + bulk UTR confirm | `td_beneficiary_*`, `td_settle_status`, `av_recon_date` |
| bank reconciliation / bank-book screen | `av_recon_date`, `apd_clear_date` |
| multi-ledger vouchers (Journal 103), Contra 104, GST expense 163 | own screens; this screen refuses with the menu name |
| batch / multi-party payment run (GoFrugal's "load and adjust many parties") | `/payments/open-items` is company-wide already |
| TDS return / challan | `WHERE av_role = 'TDS_PAYABLE'` |
| supplier ageing / creditors statement | `acc_bill_balance` CR side, the receipt's §6b.3 shape |

## 10. Decisions — all taken on the PNGs

P1 one screen, three payee kinds; cash/bank payee → Contra. P2
no bridge column (P2 withdrawn). P3 issued cheques post at issue; PDC = own voucher dated the
cheque; no ON_CLEARING. P4 `acc_cheque_book`, leaf under a row lock, consumed
by issue or void; `apd_printed_on` / `apd_print_count`. P5 TDS seeded from
the ledger via the `accounts.tds_rates` TABLE (plan §2.9), PAN missing → 20 %, on advances too,
operator may override. P6 discount received seeded from `sup_cash_disc_perc`
within `sup_credit_days`. P7 roles `TDS_PAYABLE` · `DISCOUNT_RECEIVED` ·
`BANK_CHARGES` · `ROUND_OFF` · `INTEREST_PAID` · `BALANCES_WRITTEN_BACK` ·
any ledger; **`ADVANCE_PAID` holds the on-account (added while writing —
the receipt's `ADVANCE_RECEIVED` mirrored).** P8 the identity, bank charges
outside it. P9 e-payment hook only. **P10 re-taken 2026-09-15: no approval.** P11 the six
register actions. P11b `ck_apd_cleared` loosened to R rows. P12 customer
refund against unapplied CR items only. P13 payment advice always, cheque on
the bank's format. P14 `Pmt` new, `ChqBnc` reused. P15 out: multi-ledger,
batch, bank reconciliation, day-close, 163, 104, 215. One phase.

## 11. Prerequisites — collected

Receipt plan §2 and cheques plan §2 run first. Then `plan-backend-payment.md`
§2: the `Pmt` type row; four roles + map rows; ~~`sup_ledger_id`~~ withdrawn;
`acc_cheque_book`; `td_beneficiary_name / _account / _ifsc`;
`apd_printed_on / apd_print_count`; the P11b CHECK; four settings; menus 100
visible / 49 retired / 52 visible; a grid for the payment list, one for the
issued register, one for cheque books; two print datasets (payment advice,
cheque); `acc_ledger_bank_accounts` rows for the payees who are paid by
transfer.
