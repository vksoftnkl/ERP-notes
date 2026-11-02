# Receipt voucher — the complete flow, table by table, column by column

**Revision 3, 2026-09-14** — revision 2 read the 3.0 screen
(`NextLvl/3.0/pos/Accounts/customerreceipt.cpp`, "Bill wise Receipt") and
TallyPrime's Receipt (F6) against revision 1; revision 3 read GoFrugal's
receivables features against revision 2 and, on the user's decision, put
EVERYTHING into one phase (batch posting, bank reconciliation and day-close
deliberately left out). What changed is listed in §0a; every decision is in
§10 and every one is TAKEN. Every table, column, CHECK and index below was
read off the live database at 192.168.0.106/ERP on 2026-09-14. Where the spec
files in `schema/` and `vouchers.sql` disagree with the live database, the
LIVE database wins and the disagreement is called out (§1).

**On row counts:** a TEST database. Its SCHEMA is authoritative; its DATA
only shows the shape of a thing. 18 voucher headers, 0 legs, 4 bills, 0
adjustments, 0 PDC rows, 0 ledger-map rows.

Companion files in this folder — see `README.md`. The opening-balance work
this consumes is in `../openingbal/`.

**The whole thing in one line:** a receipt is money received from a party,
split across instruments, allocated against the party's open bills (and any
credit the party already holds), with every non-money line — TDS deducted,
discount, bank charges, TCS, claims — a role-tagged ledger leg; posted as one
voucher for the money that is here today and one more voucher, dated the
cheque, for each post-dated cheque.

---

## 0. The tables involved

| table | partitioned (LIVE) | written by a receipt | role |
|---|---|---|---|
| `accounts.acc_voucher_types` | no | no — needs one ROW | the `Rct` type |
| `accounts.acc_voucher_seq` | no | bump | number series per type / branch / device |
| `accounts.acc_voucher_header` | yes | **yes — one, plus one per PDC** | the receipt |
| `accounts.acc_vouchers` | yes | **yes** | the legs; `av_role` (new) tags the non-money ones |
| `accounts.acc_tender_detail` | yes | **yes** | one row per instrument |
| `accounts.acc_bill_balance` | **yes** (spec says no) | ADVANCE row for on-account | bills settled; credits applied; credit created |
| `accounts.acc_bill_adjustment` | yes | **yes** | one row per bill per instrument-that-can-fail; the credit pairs; the deduction rows |
| `accounts.acc_pdc_register` | **yes** (spec says no) | one per cheque | every cheque, post-dated or not — **R1: the only cheque table** |
| `accounts.acc_ledger_master` | no | no | party, instrument ledgers, TDS/TCS flags |
| `accounts.acc_tender_master` / `_types` | no | no | instruments and their ledgers |
| `accounts.acc_ledger_role` / `acc_ledger_map` | no | no | every non-money ledger is found by ROLE |

Not written: `acc_voucher_cheques` (retired by R1), `acc_voucher_doc_register`
(a receipt is not a GST document — until GST-on-advance, §9), `acc_opening_balance`.

### 0a. What revision 2 changes

| | rev 1 | rev 2 | from |
|---|---|---|---|
| credits the party holds | not adjustable in a receipt (R4) | **listed with the bills, adjustable** (`ADVANCE_ADJUST` / `NOTE_ADJUST`) | 3.0 "adjust Advance"; Tally *Agst Ref* against an advance |
| post-dated cheque | posted on the receipt date, one voucher | **its own voucher dated the cheque; bills settle on maturity** | Tally Ctrl+T; 3.0's per-bill *Cheque* column |
| TDS / TCS / bank charges / surcharge / claims | out of scope, or forbidden | **the OTHER LEDGERS band**, one leg per line, `av_role` tagged | 3.0 TDS + card surcharge; Tally Particulars lines |
| bill order | bill date | **due date, increasing** (setting) | Tally *Sorting Method for Pending Bills* |
| POSTED receipt | never touched | **header-only edit** (narration, refs, collected-by) | 3.0 edit with log; money never moves |
| screen context | none | last receipts + pending cheques panels, Ledger Statement, Save & Print | 3.0 grids 813 / 563 |
| collection import | — | hook only: `avh_src_doc_type = 'COLLECTION'` | 3.0 `processCollection` (menus 187/188 later) |
| **rev 3** approval | none | **none** — withdrawn 2026-09-15; DRAFT → POSTED → CANCELLED; right `um_can_approve` | GoFrugal collection approval; our menu 188 |
| **rev 3** prompt-payment discount | discount typed | `accounts.ppd_slabs` (days → %) SEEDS the Discount cell; override allowed | GoFrugal discount slabs |
| **rev 3** TCS basis | — | `accounts.tcs_basis` = RECEIPT (206C(1H)); the TCS row seeds on it | GoFrugal config 76 (sales / receipt) |
| **rev 3** party picker | a ledger | a CUSTOMER (dropdown 39 + `iarea_id`), the picked id IS the ledger id (R9 re-taken 2026-09-15 — no bridge column); a **Beat** picker narrows the list | 3.0 `comboBeat`; GoFrugal area/salesman outstanding |
| **rev 3** bank charges | "settles the bill" | a LEG SPLIT of the instrument — outside the identity (§4.5) | the drawing forced the correction |
| **rev 3** phases | six | ONE — list, cancel, header edit, print included; batch post dropped; bank reconciliation and day-close out of scope | user's call |

---

## 1. Where live and spec disagree — read this first

### 1.1 `acc_bill_balance` IS partitioned live

`09_acc_bill_balance.sql`: *"NOT partitioned … making it a partition key is
exactly the mistake this table exists to undo."* Live: `PARTITION BY LIST
(abl_acc_year)`, composite PK, every FK to it composite (`fk_abj_bill`,
`fk_av_doc`, `fk_abl_parent`, ×3 partitions). **Every bill reference travels
with its year**: `abj_bill_id + abj_bill_acc_year` (the year column exists
live, NOT NULL; the spec had no such column), `av_doc_id + av_doc_acc_year`.

### 1.2 `acc_pdc_register` IS partitioned live

`vouchers.sql`: *"NOT partitioned. A cheque … crosses FY boundaries."* Live:
partitioned by `apd_acc_year`, composite PK. `apd_id + apd_acc_year`
everywhere; a March cheque cleared in April is a 2025_2026 row whose
`apd_clear_voucher_id` points into 2026_2027 — legal.

### 1.3 The balance trigger is NOT deployed

`09`'s `fn_abl_recompute` / `fn_abj_refresh_balance` / `tr_abj_refresh_balance`
keep `abl_alloc_amount / abl_disc_amount / abl_writeoff_amount` in step with
the adjustment rows. Live: **no function, no trigger on any accounts table**;
`abl_pending_amount` (GENERATED from those caches) never changes. Prerequisite,
with two changes: the composite key, and the post-dated rule in §5.3.

### 1.4 Two cheque tables — DECISION R1, TAKEN: `acc_pdc_register`

`acc_voucher_cheques` (03; Postgres enums against the house rule; no FK to the
header live) is retired. `fk_abj_cheque` is repointed to
`(apd_id, apd_acc_year)`. `ck_apd_dates` (`instrument_date >= received_on`)
refuses a cheque dated before the day it arrived — written yesterday, handed
over today — and is loosened to a validity window. DDL in the plan §2.3.

### 1.5 `acc_ledger_map` is empty

Twenty roles exist; no company has mapped any. Every non-money leg on a
receipt (§4.5) resolves through a role; until the rows exist, those lines are
greyed on the screen and refused by the service.

Smaller: live header has `avh_doc_amount / avh_doc_refno / avh_doc_date`
(vouchers.sql names) where `03` had `avh_bill_*`; live has `avh_against_*` and
`avh_reversal_*`; no `avh_counter_id` (the counter is `avh_device_id`);
`ck_avh_device_type` also allows `DESKTOP`.

---

## 2. `acc_voucher_types` — one row

```sql
INSERT INTO accounts.acc_voucher_types
  (vchr_type_code, vchr_type_name, vchr_type_short, vchr_category, vchr_nature,
   vchr_numbering_mode, vchr_no_prefix, vchr_no_width, vchr_reset_freq,
   vchr_affects_accounts, vchr_affects_inventory, vchr_is_cash_voucher, vchr_is_bank_voucher,
   vchr_print_title, vchr_sort_order, vchr_tally_voucher_type_name, vchr_tally_base_voucher_type)
VALUES ('Rct', 'Receipt', 'Rct', 'ACCOUNTING', 'RECEIPT', 'AUTO', 'rct', 5, 'YEARLY',
        true, false, false, false, 'RECEIPT', 270, 'Receipt', 'Receipt');
```

Cash/bank flags both false — the tender rows decide. Type 5 `ARc` stays for
order advances (`ux_avh_src` keeps them apart). A DRAFT holds no number
(`ck_avh_no`); the allocator is whatever `/bills/create` uses on
`acc_voucher_seq`.

---

## 3. `acc_voucher_header` — one row per receipt, one more per PDC

Columns that carry meaning here:

    avh_voucher_type_id   → Rct
    avh_voucher_no / slno / refno   NULL while DRAFT; 'rct00017'
    avh_voucher_date      the RECEIPT date — or the CHEQUE date on a PDC voucher (§5.2)
    avh_src_module / _doc_type / _doc_id   NULL — a receipt IS the source; set to
                          'ACCOUNTS' / 'COLLECTION' / <collection id> when raised from a
                          field collection (menus 187/188, later); ux_avh_src then makes it
                          one receipt per collection
    avh_usr_refno         the operator's own reference
    avh_doc_refno / avh_doc_date   THEIRS — payment advice no / date; ux_avh_doc_refno
                          refuses the same advice twice for a party in a year
    avh_doc_amount        Σ instruments (money received)
    avh_adjust_amount     Σ settled on bills by this voucher (money + deductions + credits)
    avh_round_off         the ROUND_OFF line, if any
    avh_total_debit / _credit   DERIVED from the legs by trigger (plan §2.4) — never stamped
    avh_party_id          the LEDGER (NOT NULL); R5
    avh_opposite_ledger_id   the one instrument ledger on a single-instrument voucher, else NULL
    avh_employee_id[1]    collected by (R6); mandatory by setting (3.0 salesmanMandatory)
    avh_remarks           narration
    avh_against_voucher_id / _acc_year   on a PDC voucher: the receipt it belongs to
    avh_reversal_voucher_id / _acc_year  on CANCEL: the contra voucher
    avh_voucher_status    DRAFT | POSTED | CANCELLED — APPROVED stays legal in ck_avh_status but
                          NOTHING WRITES IT (approval withdrawn 2026-09-15); a DRAFT has no number and no legs
    avh_draft_lines       jsonb (NEW, plan §2.6) — the other-ledger lines of a DRAFT receipt;
                          cleared at post, when they become legs
    avh_status_on / _by, avh_posted_on, avh_cancel_reason, avh_user_id, avh_session_id,
    avh_device_type ('DESKTOP'), avh_device_id, avh_print_count, avh_whatsapp_status …

Constraints that shape the flow: `ck_avh_no`, `ck_avh_balanced` (real once
totals are derived), `ck_avh_cancel`, `ck_avh_src` (all three or none),
`ux_avh_doc_refno`, `ix_avh_party` (the party's receipt history — the *last
receipts* panel reads this), `ix_avh_daybook`.

---

## 4. The receipt, table by table

Worked example used throughout — **Sri Krishna Traders, 14-09-2026, Main
Store.** The party holds a 4,000 credit note (SALES_RETURN, CR) from last
week. Received: **Cash 5,000 + UPI 15,000 (bank credited 14,990, charges 10)
+ Cheque 50,000, KVB no. 445123 dated 20-09-2026**. Customer deducted **TDS
700**. Allocated, due date first, against opening bills SB/2025/0412 (48,600)
and SB/2025/0455 (pending 12,150; 500 settlement discount typed) and this
year's bil00031 (10,000, six days old — the PPD slab "2% within 7 days" seeds
a 200 discount). Remaining on account.

### 4.1 `acc_tender_detail` — one row per instrument

Same rows and the same `td*` keys the sale bill already sends;
`ck_td_src_doc_type` already allows `'RECEIPT'`.

    td_src_module = 'ACCOUNTS'   td_src_doc_type = 'RECEIPT'   td_src_doc_id = <voucher>
    td_row_no 1..n   td_doc_date = receipt date   td_party_ledger_id = party (NOT NULL)
    td_voucher_id    = the voucher this instrument's legs are in — the RECEIPT for instant
                       money, the PDC VOUCHER for a post-dated cheque (§5.2); NULL while DRAFT
    td_tender_id / td_tender_type_id / td_tender_ledger_id   snapshots from acc_tender_master
    td_dr_cr = 'DR'  (money IN)
    td_amount        what the customer paid by this instrument: 5000 / 15000 / 50000
    td_surcharge_perc / _amt / _ledger_id   card surcharge, when the master charges one (§4.5)
    td_total_amt     = amount + surcharge  (ck_td_total_amt)
    td_received_amt / td_change_amt   cash only
    td_mdr_amt       10 — the bank's cut on the UPI; NOT a surcharge (§4.5 BANK_CHARGES)
    td_ref_no        UTR / cheque no      td_bank_name, td_payer_vpa
    td_instrument_date   2026-09-20       td_is_pdc = instrument date > receipt date (COMPUTED)
    td_settle_status  'NA' cash | 'PENDING' when the master has a clearing ledger | 'NA' cheque
    td_settle_ledger_id / td_expected_settle_on   clearing snapshot

### 4.2 `acc_vouchers` — the legs, and `av_role`

One CR party per voucher; one DR per instrument; one leg per other-ledger
line; **`av_role`** (new column, plan §2.6) names the posting role that
produced a non-money leg so every statutory report is a WHERE clause.

**Voucher 1 — rct00017, dated 14-09 (the money that is here today):**

| # | ledger | Dr/Cr | amount | `av_role` | note |
|---|---|---|---|---|---|
| 1 | Cash In Hand | DR | 5,000 | — | `td_received_amt − td_change_amt` |
| 2 | Kvb Current A/c | DR | 14,990 | — | what actually hit the bank |
| 3 | Bank Charges | DR | 10 | `BANK_CHARGES` | the bank's cut; the customer still paid 15,000 |
| 4 | TDS Receivable | DR | 700 | `TDS_RECEIVABLE` | deducted by the customer; recoverable |
| 5 | Discount Allowed | DR | 700 | `DISCOUNT_ALLOWED` | 500 typed on bill 2 + 200 seeded on bill 8 |
| 6 | Sri Krishna Traders | CR | 25,400 | — | 5,000 + 15,000 + 700 + 700 + 4,000 (the credit note applied, §4.4) |
| 7 | Sales Return / credit-note liability | DR | 4,000 | `NOTE_ADJUST` | the holding ledger is relieved; the party is credited inside row 6 |

Σ DR = 5,000 + 14,990 + 10 + 700 + 700 + 4,000 = 25,400 = Σ CR. Row 7 is the
ledger half of the credit-note PAIR of §4.4 (`acc_bill_adjustment_payload.md`
§3 gives the two bill rows): the credit's value sits in a HOLDING ledger until
it is applied — the credit note's liability ledger here, `ADVANCE_RECEIVED`
("Customer Advances Received") for an advance — and applying it moves that
value onto the party: DR holding / CR party. `ck_av_self` is why the pair is
never DR party / CR party. *(Corrected 2026-09-14 evening: the earlier text
had a third leg that did not balance; the payment design mirrors this row —
`../payment/payment_flow.md` §4.2.)*

**Voucher 2 — rct00018, dated 20-09, `avh_against_voucher_id` = rct00017
(the post-dated cheque):**

| # | ledger | Dr/Cr | amount | `av_role` |
|---|---|---|---|---|
| 1 | Cheques In Hand | DR | 50,000 | — |
| 2 | Sri Krishna Traders | CR | 50,000 | — |

`av_opp_ledger_id`: the party on each instrument / other leg; NULL on the party
leg when more than one thing faces it. `av_doc_id` left NULL — the bill-wise
trail is `acc_bill_adjustment`'s. `av_recon_date` is the bank-book screen's,
later.

### 4.3 `acc_bill_balance` — bills settled, credits applied, credit created

**Who the party is.** The screen picks a CUSTOMER — dropdown 39, with a new
optional `iarea_id` parameter so that choosing a **Beat** (`sales.area_master`,
dropdown 13, the same picker the sale bill's `ddBeat` uses) narrows the list.
`sales.customers.cus_id` **is** the ledger id (R9 re-taken 2026-09-15 — no bridge column, plan §2.11), so the picked id already IS
`avh_party_id`; a customer with no control ledger is refused by name. A
receipt from a non-customer ledger (bank interest, a deposit refund) picks the
ledger directly and gets an empty bill panel (R5).

**Read — the party's open items, BOTH sides, due date first, across years:**

```sql
SELECT abl_id, abl_acc_year, abl_bill_type, abl_dr_cr, abl_doc_refno, abl_doc_date,
       abl_due_date, abl_bill_amount, abl_pending_amount, abl_status,
       GREATEST(CURRENT_DATE - abl_due_date, 0) AS days_overdue,
       -- 3.0's "Cheque" column: PDC held against this bill, not yet matured (§5.3)
       (SELECT COALESCE(SUM(j.abj_amount),0) FROM accounts.acc_bill_adjustment j
         WHERE j.abj_bill_id = b.abl_id AND j.abj_bill_acc_year = b.abl_acc_year
           AND j.abj_is_post_dated AND j.abj_adj_date > CURRENT_DATE
           AND NOT j.abj_is_deleted) AS pdc_held
  FROM accounts.acc_bill_balance b
 WHERE abl_company_id = :company AND abl_party_id = :party
   AND NOT abl_is_deleted AND abl_is_active
   AND abl_pending_amount > 0
   AND ( (abl_dr_cr = 'DR' AND abl_bill_type IN ('SALES','OPENING','INTEREST','JOURNAL'))
      OR (abl_dr_cr = 'CR' AND abl_bill_type IN ('ADVANCE','SALES_RETURN','OPENING')) )
 ORDER BY abl_dr_cr DESC, COALESCE(abl_due_date, abl_doc_date), abl_doc_refno;
```

DR rows are what the party owes; CR rows are what the party holds — an
advance from an earlier on-account receipt, a credit note, an OPENING credit.
The screen shows CR rows in their own band under the bills; applying one is
§4.4. No branch filter — outstanding is company-wide. The sort is *due date
increasing* (Tally's default), switchable to bill date by setting.

**Write — only when money is left over: on-account = a new ADVANCE bill**
(Tally's *Advance* — a credit the party holds, adjustable later by the sale
bill's panel or the next receipt):

    abl_company_id / abl_branch_id (receipt's) / abl_tenant_id / abl_acc_year (receipt year)
    abl_party_id / abl_bill_type = 'ADVANCE' / abl_dr_cr = 'CR'
    abl_src_module = 'ACCOUNTS'  abl_src_doc_type = 'RECEIPT'  abl_src_doc_id = <voucher 1>  abl_src_acc_year
    abl_voucher_id / _type_id / _no / _date / _refno = voucher 1   (ck_abl_voucher: not OPENING ⇒ set)
    abl_doc_refno = 'rct00017'   abl_doc_date = receipt date   abl_credit_days = 0
    abl_bill_amount = <on account>   alloc/disc/writeoff = 0 (trigger-owned)
    abl_narration = 'On account — receipt rct00017'

Tally's *On Account* (unreferenced, no bill row) is **not** offered: an
unreferenced credit is what ageing cannot see (R11).

### 4.4 `acc_bill_adjustment` — every settlement event

Granularity, from `09` and kept: instant instruments taken together share one
row per bill (mode `MIXED`); every cheque gets its own row per bill (a bounce
reverses exactly that); every deduction that settles a bill gets a row; a
credit applied is a PAIR of rows.

**Order of application** (deterministic, server-side, the screen previews it):
credits held → deductions that settle → instant money → cheques (current-dated,
then post-dated by date), each walking the bills in the list order. Cheques
land on the LAST bills, so a bounce reopens the newest debt and the oldest
stays settled by money that cleared.

Worked example — bill 1 SB/2025/0412 pending 48,600; bill 2 SB/2025/0455
pending 12,150; bill 8 bil00031 pending 10,000:

| # | bill | `abj_adj_type` | amount | mode | `abj_against_bill_id` | tender / cheque | post-dated | date |
|---|---|---|---|---|---|---|---|---|
| 1 | bill 1 | NOTE_ADJUST | 4,000 | CREDIT_NOTE | the credit note | — | f | 14-09 |
| 2 | credit note | NOTE_ADJUST | 4,000 | CREDIT_NOTE | bill 1 | — | f | 14-09 |
| 3 | bill 1 | ALLOCATION | 700 | **TDS** | — | — | f | 14-09 |
| 4 | bill 1 | ALLOCATION | 20,000 | MIXED | — | — | f | 14-09 |
| 5 | bill 1 | ALLOCATION | 23,900 | CHEQUE | — | td 3 / apd | **t** | **20-09** |
| 6 | bill 2 | ALLOCATION | 11,650 | CHEQUE | — | td 3 / apd | **t** | **20-09** |
| 7 | bill 2 | DISCOUNT | 500 | DISCOUNT | — | — | f | 14-09 |
| 8 | bill 8 | ALLOCATION | 9,800 | CHEQUE | — | td 3 / apd | **t** | **20-09** |
| 9 | bill 8 | DISCOUNT | 200 | DISCOUNT | — | — | f | 14-09 (PPD-seeded) |

Bill 1: 4,000 + 700 + 20,000 + 23,900 = 48,600 → CLOSED on 20-09. Bill 2:
11,650 + 500 = 12,150 → CLOSED on 20-09. Bill 8: 9,800 + 200 = 10,000 →
CLOSED on 20-09. Cheque used 45,350 of 50,000; the remaining **4,650 goes on
account** (the ADVANCE bill of §4.3 — dated 20-09, because that money exists
on 20-09; see §5.2). `avh_adjust_amount` on voucher 1 = 25,400 (rows
1,3,4,7,9); on voucher 2 = 45,350.

Bank charges (row 3 of the legs) settle nothing extra: the customer paid
15,000 and 15,000 is what row 4 carries for the UPI part; the 10 is our
expense. TDS *does* settle: the customer discharged 700 of the bill by paying
it to the government on our behalf — `ALLOCATION` with mode `TDS`, so
`fn_abl_recompute` folds it into `abl_alloc_amount` unchanged.

Every row: `abj_company_id`, `abj_branch_id` (settled at), `abj_acc_year`
(the VOUCHER's year), `abj_bill_acc_year` (the BILL's — §1.1), `abj_party_id`,
`abj_row_no` per bill, `abj_dr_cr = 'CR'` on a DR bill and `'DR'` on the
credit-side row of a pair (`acc_bill_adjustment_payload.md` §3),
`abj_adj_date` = the voucher's date, `abj_voucher_id / _acc_year` = the
voucher the leg is in (voucher 1 or the PDC voucher), `abj_voucher_line_id` =
the party CR leg, `abj_settlement_ledger_id` = the instrument or role ledger,
`abj_user_id`, `abj_session_id`.

`ck_abj_against` holds: ALLOCATION / DISCOUNT / WRITEOFF name no opposite bill;
NOTE_ADJUST / ADVANCE_ADJUST must. `ck_abj_cheque_mode` / `ck_abj_pdc`: the
cheque rows carry `abj_cheque_id + abj_cheque_acc_year`. `ck_abj_settlement_mode`
needs **`'TDS'` and `'CLAIM'` added** (plan §2.5).

**The rule the database cannot check:** per bill, Σ of this receipt's rows ≤
`abl_pending_amount` NOW, with the bill rows locked (`SELECT … FOR UPDATE`)
inside the posting transaction. `ck_abl_settled` is the backstop.

### 4.5 The OTHER LEDGERS band — every non-money line is a role

A receipt in Indian trade carries lines that are neither an instrument nor a
bill. Rather than a field per case, the screen has one band, and every line is
`{ role | ledger, drCr, amount, settlesBill, againstBill?, narration }`:

| line | side | settles the bill | role | seeded when |
|---|---|---|---|---|
| TDS deducted by the customer | DR | yes | `TDS_RECEIVABLE` (new) | `led_is_tds_applicable` — rate from `led_tds_deductee_type` |
| Bank charges / MDR | DR | **no — a leg split**: the customer's 15,000 is already the instrument; the bank kept 10 of it | `BANK_CHARGES` (new) | `td_mdr_amt > 0` on an instrument |
| Settlement discount | DR | yes | `DISCOUNT_ALLOWED` | the bill grid's Discount column, mirrored here read-only |
| Write-off / short payment | DR | yes, needs `abj_approved_by` | `WRITE_OFF` | the bill grid's W/off column |
| Customer claim / deduction (damage, freight, scheme) | DR | yes | `CLAIMS_ALLOWED` (new) | typed |
| Round-off | either | yes | `ROUND_OFF` | receipt of 4,999.60 for 5,000 |
| TCS collected on receipt (206C(1H)) | CR | no — money on top of the bill | `TCS_PAYABLE` | `led_is_tcs_applicable` AND `accounts.tcs_basis = 'RECEIPT'`. **On the SALES basis this row never appears** — the TCS is already inside the bill, and bill-wise TCS outstanding / collection come from `abl_tcs_amount` and the view `accounts.v_bill_tcs` (plan §2.13). The per-buyer 50L threshold runs on cumulative RECEIPTS from that buyer, not on TCS collected |
| Card surcharge recovered | CR | no | `SURCHARGE_RECOVERED` (new) | the tender master's `tnd_surcharge_*` |
| Interest on overdue received | CR | no (or settles an INTEREST bill) | `INTEREST_INCOME` (new) | typed |
| any other ledger | either | as ticked | role NULL, `av_role` NULL | typed |

Two shapes only — **DR-other that settles the bill** (a deduction the
customer made: TDS, discount, write-off, claim) and **CR-other** (money on top
of the bill: TCS, surcharge, interest) — plus one shape that is neither: a
**leg split** (bank charges), which changes what the bank got, not what the
customer paid, and so stays OUTSIDE the identity the strip enforces:

```
Σ instruments + Σ deductions + Σ credits applied  =  Σ allocated to DR bills + on account + Σ additions
```

**Prompt-payment discount (rev 3).** `accounts.ppd_slabs` is a JSON list of
`{days, perc}` (e.g. `[{"days":7,"perc":2},{"days":15,"perc":1}]`). For every
DR bill whose age at the receipt date falls inside a slab, open-items returns
`ppdSuggested`, and the screen SEEDS the Discount cell with it — visibly, as
a seeded value the operator may change or clear. The server accepts whatever
the operator left, and never re-seeds at post. The slab is a suggestion; the
DISCOUNT row is the fact.

`settlesBill` lines write `acc_bill_adjustment` rows (`ALLOCATION` with mode
`TDS` / `CLAIM` / `BANK`; `DISCOUNT`; `WRITEOFF`); `againstBill` blank = spread
pro-rata over the bills allocated in this receipt (server, deterministic —
TDS per invoice matters for 26AS matching, which is why the link exists).
Every line becomes one leg with `av_role` set. Reserved, not built: an
`acc_tds_register` (deductor, section, rate, certificate, quarter, 26AS
match) keyed by `(av_id, av_acc_year)`; GST on advance (§9).

### 4.6 `acc_pdc_register` — one row per cheque, every cheque

    apd_company_id / apd_branch_id (received at) / apd_tenant_id / apd_acc_year (receipt year)
    apd_tra_type = 'R'   apd_party_id   apd_salesman_id = avh_employee_id[1]
    apd_instrument_type = 'CHEQUE'   apd_instrument_no = '445123'   apd_instrument_date = 2026-09-20
    apd_amount = 50,000   apd_bank_name / _branch / _ifsc / _micr / apd_drawer_name (received from)
    apd_received_on = receipt date   apd_bank_ledger_id = the OWN bank (set at deposit)
    apd_posting_mode = company setting, ON_RECEIPT in this plan (R2)
    apd_voucher_id / _acc_year = the voucher carrying the cheque's legs — voucher 1 for a
                                  current-dated cheque, the PDC voucher for a post-dated one
    apd_tender_id = td row   apd_status = 'HELD'   apd_status_on / _by   apd_present_count = 0

`ux_apd_instrument` refuses the same number from the same party twice in a
year. Deposit / clear / bounce / replace / cancel are menu 51's (3.0
`receivedcheques`: Deposit F5, Clear F7, Bounce F8, Cancel F3, Print F6,
History) — defined by this table and §6.

---

## 5. Cheques — current-dated, post-dated, and the posting mode

### 5.1 Current-dated (instrument date ≤ receipt date)

Part of voucher 1: DR Cheques In Hand / CR party on the receipt date;
allocation rows not post-dated; register row HELD. Clearing later moves
Cheques In Hand → Bank (menu 51).

### 5.2 Post-dated — Tally's model, kept exactly

Tally (Ctrl+T): *"the transaction affects the books on the post-dated cheque
date"*; until then it shows in the Post-Dated Summary as *Pending* and becomes
*Regularized* when the date arrives. 3.0 showed the same fact as the *Cheque*
column on each bill — money promised, not yet effective.

So a post-dated cheque is **its own voucher**, numbered at post like any other
(rct00018), `avh_voucher_date = apd_instrument_date`, `avh_against_voucher_id`
= the receipt, legs DR Cheques In Hand / CR party, status POSTED. Because
every statement, day book and trial balance is date-filtered, it is invisible
until 20-09 and appears on 20-09 with no further action. The receipt (voucher
1) carries `avh_doc_amount` = instant money only; the PDC voucher carries the
cheque.

On-account money that comes from the cheque (14,450 in the example) is an
ADVANCE bill dated the cheque date, voucher = the PDC voucher: the credit
exists when the money does.

### 5.3 Bills settle on maturity — the post-dated rule in the trigger

The allocation rows for the cheque are written at post with `abj_adj_date` =
cheque date and `abj_is_post_dated = true` (`ck_abj_pdc` needs the cheque id —
present). `fn_abl_recompute` **excludes rows with `abj_is_post_dated AND
abj_adj_date > CURRENT_DATE`**, so the bill stays open and the screen shows
the amount as *PDC held*. A daily job re-runs the recompute for every bill
with a row maturing that day (plan §2.1 `fn_abl_regularise_pdc`; partial index
on `(abj_adj_date) WHERE abj_is_post_dated`). That is Tally's *Regularized*,
and it costs one index and one cron line.

### 5.4 The posting mode — R2

| | ON_RECEIPT (this plan) | ON_CLEARING (with the Received Cheques screen, menu 51) |
|---|---|---|
| legs | as above, on receipt / cheque date | none until the bank clears; party stays outstanding |
| allocations | at post (post-dated ones mature per §5.3) | written at clearing with the clear date |
| `apd_voucher_id` | the voucher | NULL until cleared |
| bounce | reversal voucher + negative rows; bill reopens | register row → BOUNCED, nothing to reverse |

`ACCOUNTS.PDC_POSTING_MODE` in `app_settings`, company-scoped, default
`ON_RECEIPT`. This plan refuses `ON_CLEARING` outright rather than half-doing it.

---

## 6. Cancel, header edit, bounce — and why money is never edited

**Cancel** (any time after POSTED), one transaction: status CANCELLED + reason;
a reversal voucher per voucher (the receipt AND each PDC voucher), mirrored
legs, `avh_against_voucher_id` back, `avh_reversal_voucher_id` set; one
negative `acc_bill_adjustment` row per original row (`abj_reversal_of_id`) —
the trigger reopens the bills; the ADVANCE bill soft-deleted — refused if any
of it has since been adjusted; each register row HELD → CANCELLED with reason
— **refused once DEPOSITED or later** ("the cheque is with the bank; bounce
or clear it first" — 3.0's flag-11 refusal, same shape); tender rows
soft-deleted. 3.0's *delete with a remark* is this, with a trail instead of a
hole.

**Header-only edit** on a POSTED receipt: `avh_remarks`, `avh_usr_refno`,
`avh_doc_refno / _date`, `avh_employee_id` — nothing that touches a figure,
a leg, a bill or a cheque. `avh_modified_*` stamped; the edit remark is the
audit screen's (3.0 `editingLog`). Anything else: cancel and re-enter (R3).

**Bounce** belongs to menu 51: reversal voucher for the cheque's legs only,
negative rows for the rows carrying that `abj_cheque_id`, register row →
BOUNCED (`apd_bounce_voucher_id`), optional bounce-charges debit note
(`apd_charge_voucher_id`). The cash and UPI of the same receipt stand.

**Draft**: edited freely — header + tenders + other-lines only, no number, no
legs, no adjustments, no register rows (R10: allocations are recomputed on
reopen; the order in §4.4 makes that one keystroke).

**Approval — REMOVED 2026-09-15 (R14 re-taken).** There is none. The lifecycle
is **DRAFT → POSTED → CANCELLED**: a DRAFT touches no bill, writes no leg and
holds no number, so it already IS the unapproved state, and a second gate in
front of Post is the same decision taken twice (user: *"remove approve option
because we have posted its a double work"*). **Post always runs from a DRAFT**
— `/create` saves the draft, `/post` turns it into a voucher in one
transaction. `APPROVED` remains legal in `ck_avh_status` and nothing writes it;
`um_can_approve` is not being added. The write-off approver (`abj_approved_by`,
`accounts.writeoff_approval_above`) is a different thing and is unaffected: it
is a name recorded on the line, not a state on the document.

---

## 7. Rules the DATABASE enforces

| constraint | forces |
|---|---|
| `ck_avh_no` / `ck_avh_balanced` / `ck_avh_cancel` / `ck_avh_status_on` / `ck_avh_posted_on` | numbered when not DRAFT; Dr = Cr at POSTED (real once derived); lifecycle columns |
| `ck_avh_src` | src triple all-or-nothing — NULL for a keyed receipt, all three for a collection-raised one |
| `ux_avh_doc_refno` | one receipt per payment advice per party/year |
| `ck_av_amount` / `ck_av_dr_cr` / `ck_av_self` / `ux_av_voucher_row` | legs positive, sided, no self-contra, unique rows |
| `fk_av_role` (new) | a role on a leg is a real role |
| `ck_td_total_amt` / `ck_td_pdc` / `ck_td_cash_change` / `ck_td_card_last4` | tender arithmetic; a PDC has a date |
| `ck_abj_against` | who may and may not name an opposite bill |
| `ck_abj_cheque_mode` / `ck_abj_pdc` | a cheque settlement names its cheque |
| `ck_abj_writeoff_approval` / `ck_abj_reversal_sign` | write-offs signed; reversals negative |
| `ck_abl_settled` | never over-settled — the backstop behind the row lock |
| `ck_abl_voucher` / `ux_abl_doc_refno` | the ADVANCE bill names its voucher; its refno unique per party/type/year |
| `ux_apd_instrument` / `ck_apd_posting` / `ck_apd_dates` (loosened) | one cheque number per party/year; ON_RECEIPT names its voucher; date within a window |

## 8. Rules ONLY the service can enforce

1. The identity in §4.5, to the paisa, or no post.
2. Per bill, allocation ≤ pending under `FOR UPDATE`; per credit, applied ≤ pending, same lock.
3. Every role-tagged line has a mapped ledger; unmapped → refused by name.
4. A write-off has an approver; a claim above a setting threshold too.
5. Date in an OPEN, unlocked year — for the receipt AND for each PDC voucher's date.
6. `td_is_pdc` computed, never trusted; a PDC row's date within the `ck_apd_dates` window.
7. Totals derived: `avh_total_*` by trigger, `avh_doc_amount` from tenders, `avh_adjust_amount` from adjustments.
8. The register row and the tender row agree (amount, number, date).
9. A receipt with no instruments and only credits applied is refused — "that is a journal".
10. Salesman mandatory when the setting says so (3.0 `salesmanMandatory`).
11. Post runs from a DRAFT; there is no approval state to refuse (approval withdrawn 2026-09-15).
12. The PPD seed is computed server-side from the slab and the bill's age at the receipt date; the operator's override is what posts.
13. The TCS row is seeded only when the party is TCS-applicable AND `accounts.tcs_basis = 'RECEIPT'`; on SALES it must NOT appear, and `tcsAmount` / `tcsPending` show on the bill row instead.
14. There is no customer→ledger resolution to fail: `cus_id` IS `led_id` (R9).

---

## 9. Deferred, with the hook that makes it cheap later

| what | hook |
|---|---|
| TDS register / 26AS | `av_role = 'TDS_RECEIVABLE'` + `againstBill` per invoice → `acc_tds_register(av_id, av_acc_year, …)` |
| TCS threshold (50L per buyer per FY) | cumulative **receipts** from the buyer in the year decide it (not TCS collected); computed, not stored — plan §2.14 |
| GST on advance (services) | CR Output GST lines on the on-account part + a doc-register row (GSTR-1 11A); `abl_bill_type = 'ADVANCE'` already carries the base |
| Interest on overdue | `abl_bill_type = 'INTEREST'` + `abl_parent_bill_id` exist; an interest debit-note screen raises them, this screen settles them |
| Field collection (menus 187/188) | `avh_src_doc_type = 'COLLECTION'`; the collection screen creates receipt DRAFTs |
| Formal receipt print | `avh_print_count`; a dataset in the printing schema (17) |
| WhatsApp / SMS | `avh_whatsapp_status / avh_sms_status` |
| Bank reconciliation | `av_recon_date` / `av_is_reconciled` on every instrument leg — **screen deliberately NOT designed (user, 2026-09-14)** |
| Day-close lock | none — **deliberately out (user, 2026-09-14)**; `fy_lock_date` is the only lock |
| Batch posting of many receipts | **dropped (user, 2026-09-14)** — a collection screen posts N receipts as N calls |
| Multi-currency | out — no currency anywhere |

## 10. Decisions

| # | decision | taken |
|---|---|---|
| R1 | cheque table | **`acc_pdc_register` only**; repoint `fk_abj_cheque`; loosen `ck_apd_dates` |
| R2 | PDC posting mode | company setting, default ON_RECEIPT; this plan ON_RECEIPT only; PDC = own voucher dated the cheque (§5.2) |
| R3 | edit a POSTED receipt | header-only fields; money = cancel + re-enter |
| R4 | credits applied inside a receipt | **yes** — listed with the bills, `ADVANCE_ADJUST` / `NOTE_ADJUST` pairs |
| R5 | party picker | PARTY ledgers; any ledger allowed with an empty bill panel |
| R6 | collected-by | one salesman → `avh_employee_id[1]`; mandatory by setting |
| R7 | on-account remainder | always an ADVANCE bill (Tally *Advance*); never unreferenced |
| R8 | TDS / TCS / surcharge / charges / claims | the other-ledgers band, roles + `av_role` (§4.5) |
| R9 | customer ↔ ledger bridge | **RE-TAKEN 2026-09-15: none needed** — `sales.customers.cus_id` IS `acc_ledger_master.led_id`, and `purchase.suppliers.sup_id` likewise (plan §2.11) |
| R10 | draft allocations | nowhere; recomputed on reopen |
| R11 | Tally's unreferenced *On Account* | not offered |
| R12 | pending-bill sort | due date increasing; setting to switch to bill date |
| R13 | field collection import | own screen later; hook only |
| R14 | approval before post | **RE-TAKEN 2026-09-15: none.** A DRAFT is the unapproved state; Post runs from it directly. No setting, no `um_can_approve` |
| R15 | TCS basis | `accounts.tcs_basis` = RECEIPT (default) \| **SALES — the basis for the clients who need bill-wise TCS outstanding (user, 2026-09-15)**. The two mechanisms never both apply: RECEIPT = a `TCS_PAYABLE` leg on the receipt; SALES = `abl_tcs_amount` on the bill, collection a slice of the allocation (plan §2.13) |
| R16 | prompt-payment discount | `accounts.ppd_slabs` seeds the Discount cell; override allowed; never decides |
| R17 | party picker | a CUSTOMER via dropdown 39 + `iarea_id`; Beat (dropdown 13) narrows it; the picked id IS the ledger — **R9 re-taken, no bridge** |
| R18 | phasing | ONE phase — list, cancel, header edit, print included; batch post dropped; bank reconciliation and day-close out |

## 11. Prerequisites before any code — collected

1. `fn_abl_recompute(uuid, char(9))` with the post-dated exclusion + trigger + `fn_abl_regularise_pdc()` daily (plan §2.1).
2. `fn_avh_refresh_totals()` trigger on `acc_vouchers` (plan §2.4) — fixes notes (4).
3. The `Rct` type row (§2).
4. R1: repoint `fk_abj_cheque`; loosen `ck_apd_dates` (plan §2.3).
5. `acc_vouchers.av_role` + FK (plan §2.6).
6. Roles: `TDS_RECEIVABLE`, `BANK_CHARGES`, `SURCHARGE_RECOVERED`, `CLAIMS_ALLOWED`, `INTEREST_INCOME`; map these + `DISCOUNT_ALLOWED`, `WRITE_OFF`, `ADVANCE_RECEIVED`, `TCS_PAYABLE`, `ROUND_OFF` for the company (§1.5).
7. `ck_abj_settlement_mode` += `'TDS'`, `'CLAIM'` (plan §2.5).
8. Settings (`public.app_setting_def`, lowercase dotted keys like the live `system.regional`): `accounts.pdc_posting_mode`, `accounts.receipt_bill_sort`, `accounts.receipt_salesman_mandatory`, `accounts.writeoff_approval_above`, `accounts.tcs_basis`, `accounts.ppd_slabs`.
9. Menu 99 visible; 48 retired (one screen).
10. ~~`um_can_approve` / `ctx.canApprove`~~ — WITHDRAWN 2026-09-15 with the approval step.
11. ~~`cus_ledger_id`~~ — withdrawn; instead declare the identity that already holds: FK `customers.cus_id → acc_ledger_master.led_id` (plan §2.11).
12. Dropdown "CUSTOMERS BY AREA" — dropdown 39's SELECT with an optional `iarea_id` (plan §2.12).
13. `acc_voucher_header.avh_draft_lines jsonb` (plan §2.6).
