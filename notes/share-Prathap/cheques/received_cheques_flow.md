# Received cheques — the complete flow, table by table, column by column

**Revision 1, 2026-09-14.** Every table, column, CHECK and index below was
read off the live database at 192.168.0.106/ERP on 2026-09-14. The lifecycle
is the one `ck_apd_*` already constrains; what the schema does not decide is
in §10 as decisions C1–C10, all TAKEN (confirmed by the user on the PNGs
before this document was written).

**References:** 3.0 `NextLvl/3.0/pos/Accounts/receivedcheques.cpp` +
`chequedeposit` + `chequebounce` (status filters, F5 Deposit / F7 Clear /
F8 Bounce / F3 Cancel / F6 Print / Ctrl+H History, bounce charge BY the bank
and charge TO the party); TallyPrime (deposit-slip print, Cheque Register,
Post-Dated Summary, bounce = reversal + bank charge, bank date for
reconciliation); GoFrugal (receive → deposit → acknowledge clearance →
reconcile e-statement; bounce control; "a debit note will automatically be
created against the party").

Companion files in this folder — see `README.md`. The receipt that creates
every row this screen moves is `../receipt/` (rev 3).

**The whole thing in one line:** the receipt puts a cheque in the register
and (under ON_RECEIPT) into Cheques In Hand; this screen takes it to the bank
(no voucher), lets the bank clear it (one contra voucher) or bounce it (one
journal voucher that reverses the money, reopens the bills and charges the
party), and re-issues it if the customer presents it again or replaces it —
never editing a row, always adding one.

---

## 0. The tables involved

| table | partitioned (LIVE) | written here | role |
|---|---|---|---|
| `accounts.acc_pdc_register` | **yes**, `apd_acc_year` (spec says no) | **yes — every action** | the register; `apd_tra_type = 'R'` only |
| `accounts.acc_voucher_types` | no | no — needs TWO rows | `ChqClr` (clearing), `ChqBnc` (bounce) |
| `accounts.acc_voucher_header` / `acc_vouchers` | yes | clear, bounce, re-issue, return | one voucher per step that moves money |
| `accounts.acc_bill_adjustment` | yes | bounce, return, re-issue | the negative reversal rows; fresh allocations on re-issue |
| `accounts.acc_bill_balance` | **yes** (spec says no) | bounce | soft-deletes the cheque's ADVANCE; adds a JOURNAL bill for the party's charge |
| `accounts.acc_ledger_role` / `acc_ledger_map` | no | no — needs one role | `BOUNCE_CHARGES_RECOVERED` |
| `public.txn_status_log` (07) | — | every transition | the History panel |
| `accounts.acc_tender_detail` | yes | no | `apd_tender_id` points at it; read for the instrument row |

Not written: `acc_voucher_cheques` (retired — receipt R1), `acc_opening_balance`,
`acc_voucher_doc_register`.

---

## 1. `acc_pdc_register` — the register, column by column

Read off the live table (partitioned by `apd_acc_year`; composite PK; every
reference to a row carries both `apd_id` and `apd_acc_year`):

    apd_id + apd_acc_year     PK; apd_acc_year = the FY the cheque was RECEIVED in
    apd_company_id / apd_branch_id (received at) / apd_tenant_id
    apd_tra_type              'R' — this screen; 'P' is menu 52 (ck_apd_tra_type)
    apd_party_id              → acc_ledger_master (fk_apd_party)
    apd_salesman_id           who collected it
    apd_instrument_type       CHEQUE | DD | PAY_ORDER | ECS | NACH | UPI_MANDATE (ck_apd_instrument)
    apd_instrument_no         ux_apd_instrument: (company, party, type, no, year) WHERE not CANCELLED
    apd_instrument_date       the date ON the cheque — maturity; the due list works from it
    apd_amount                > 0 (ck_apd_amount)
    apd_bank_name / _branch / apd_ifsc / apd_micr / apd_drawer_name   the DRAWER's bank
    apd_received_on           the receipt date
    apd_bank_ledger_id        OUR bank — NULL until DEPOSIT (fk_apd_bank_ledger)
    apd_posting_mode          ON_RECEIPT | ON_CLEARING (ck_apd_posting_mode) — copied from the setting at receipt
    apd_voucher_id / _acc_year   the receipt (current-dated) or the PDC voucher (post-dated) — ck_apd_posting: NOT NULL under ON_RECEIPT
    apd_tender_id             the acc_tender_detail row (no FK)
    apd_status                HELD | DEPOSITED | CLEARED | BOUNCED | RETURNED | CANCELLED | REPLACED (ck_apd_status)
    apd_status_on / _by       the LAST transition only
    apd_present_count         ≥ 0; +1 per deposit; 2 = re-presented
    apd_deposit_date / apd_deposit_slip_no     ck_apd_deposited: date NOT NULL at DEPOSITED / CLEARED / BOUNCED
    apd_clear_date / apd_clear_voucher_id / _acc_year   ck_apd_cleared: both NOT NULL at CLEARED
    apd_bounce_date / apd_bounce_reason        ck_apd_bounced: both NOT NULL at BOUNCED
    apd_bounce_charges        ≥ 0 — the BANK's charge (ck_apd_charges)
    apd_bounce_voucher_id / _acc_year          the reversal voucher
    apd_charge_voucher_id / _acc_year          the charge voucher — the SAME voucher (C2)
    apd_replaced_by_id / _acc_year             ck_apd_replaced: NOT NULL at REPLACED; self-FK across partitions; ck_apd_self
    apd_cancel_date / apd_cancel_reason        ck_apd_cancelled: reason NOT NULL at CANCELLED / RETURNED
    apd_remarks, apd_is_active, apd_is_deleted, audit columns
    ck_apd_seq                clear date ≥ deposit date; bounce date ≥ deposit date
    ck_apd_dates              instrument date within a window — loosened by the receipt plan §2.3

Indexes that the screen leans on: `ix_apd_due (company, instrument_date)
WHERE HELD/DEPOSITED` (the due list), `ix_apd_party`, `ix_apd_bank (company,
bank_ledger, instrument_date) WHERE HELD`, `ix_apd_bounced`, `ix_apd_branch
(company, branch, status)`, `ix_apd_voucher`, `ix_apd_replaced`.

The lifecycle the CHECKs already enforce — and this screen never leaves:

    HELD ──► DEPOSITED ──► CLEARED
                  └──────► BOUNCED ──► DEPOSITED again (re-present)   or   REPLACED (+ a new HELD row)
    HELD ──► RETURNED | CANCELLED | REPLACED

---

## 2. The two voucher types and the one role this screen needs

Live: six types, none for clearing or bounce; `VoucherNature` enum has
`CONTRA` and `JOURNAL`, so no enum change.

```sql
INSERT INTO accounts.acc_voucher_types
  (vchr_type_code, vchr_type_name, vchr_type_short, vchr_category, vchr_nature,
   vchr_numbering_mode, vchr_no_prefix, vchr_no_width, vchr_reset_freq,
   vchr_affects_accounts, vchr_affects_inventory, vchr_is_cash_voucher, vchr_is_bank_voucher,
   vchr_print_title, vchr_sort_order, vchr_tally_voucher_type_name, vchr_tally_base_voucher_type)
VALUES
  ('ChqClr', 'Cheque clearing', 'ChqClr', 'ACCOUNTING', 'CONTRA',
   'AUTO', 'chqclr', 5, 'YEARLY', true, false, false, true,
   'CHEQUE CLEARING', 280, 'Contra', 'Contra'),
  ('ChqBnc', 'Cheque bounce', 'ChqBnc', 'ACCOUNTING', 'JOURNAL',
   'AUTO', 'chqbnc', 5, 'YEARLY', true, false, false, false,
   'CHEQUE BOUNCE', 281, 'Journal', 'Journal');

INSERT INTO accounts.acc_ledger_role (alr_role, alr_label, alr_want_nature, alr_want_type)
VALUES ('BOUNCE_CHARGES_RECOVERED', 'Cheque bounce charges recovered from parties', 'Income', 'INCOME');
```

then one `acc_ledger_map` row for the company. `BANK_CHARGES` and the
`Rct` type come from the receipt plan §2. Cheques In Hand is the CHEQUE
tender master's `tnd_ledger_id` — the same ledger the receipt debited; this
screen reads it from the cheque's `apd_tender_id → td_tender_ledger_id`, so
a later master edit cannot move an old cheque.

---

## 3. The six actions, table by table

Worked example throughout, continuing `../receipt/`: on 14-09-2026 the
register holds, among others, **445123** (Sri Krishna Traders, 50,000, dated
20-09, HELD, PDC voucher rct00018), **445100** (same party, 20,000, dated
18-09, HELD), **221870** (Anand Stores, 18,500, DEPOSITED into KVB on 10-09,
slip D-118, it settled bil00022 12,500 + bil00025 6,000).

### 3.1 DEPOSIT — HELD → DEPOSITED — many cheques, one slip, NO voucher

The money is still promised, not received; Cheques In Hand does not move.

    per selected row (one transaction):
      apd_bank_ledger_id      = the own bank chosen (dropdown 25 BANK LEDGERS)
      apd_deposit_date        = the slip date  — refused if < apd_instrument_date ('post-dated: deposit on or after 20-09')
      apd_deposit_slip_no     = the slip number, shared by every cheque on the slip
      apd_present_count       += 1
      apd_status              = 'DEPOSITED', apd_status_on = now(), apd_status_by
      txn_status_log          one row per cheque: HELD → DEPOSITED, who, when, slip

Refused: a row not HELD (or BOUNCED — re-present is deposit again, §3.4); a
deposit date before the instrument date; a deposit date in a closed year.

**Deposit slip** — one print per `(apd_bank_ledger_id, apd_deposit_date,
apd_deposit_slip_no)`: the bank's own header, then cheque no · drawer bank ·
amount per line, total, count. A dataset in the printing schema (17) — the
same pipeline the invoice uses; nothing cheque-specific (C8). Tally's
*Cheque Deposit Slip*, GoFrugal's step 2 of 4.

### 3.2 CLEAR — DEPOSITED → CLEARED — the bank confirms — one contra voucher

**ON_RECEIPT** (the receipt already posted DR Cheques In Hand / CR party):

    acc_voucher_header  type ChqClr, avh_voucher_date = clear date, avh_party_id = the cheque's party,
                        avh_against_voucher_id / _acc_year = the receipt or PDC voucher (apd_voucher_id),
                        avh_src_module = 'ACCOUNTS', avh_src_doc_type = 'PDC', avh_src_doc_id = apd_id   (C10 — ux_avh_src: one clearing per cheque)
                        avh_doc_amount = apd_amount, avh_remarks = 'Clearing 445123 · slip D-121', status POSTED (last, after the legs)
    acc_vouchers        row 1  DR  own bank (apd_bank_ledger_id)   apd_amount   av_opp_ledger_id = Cheques In Hand
                                  av_recon_date = the BANK date typed on the panel (defaults to the clear date) — the bank book's hook
                        row 2  CR  Cheques In Hand               apd_amount   av_opp_ledger_id = own bank
    acc_pdc_register    apd_clear_date, apd_clear_voucher_id / _acc_year = this voucher, apd_status = 'CLEARED', status_on/by
    txn_status_log      DEPOSITED → CLEARED

Nothing on the bills: they were settled at receipt (current-dated) or on the
cheque date (post-dated, `fn_abl_regularise_pdc`). Clearing moves money
between OUR ledgers only.

**ON_CLEARING** (the receipt posted nothing for the cheque): the clearing
voucher IS the receipt for it — type `Rct`, DR own bank / CR party, plus the
allocations chosen on the panel (auto FIFO over the party's open bills,
editable — the same allocation engine and the same `abj` rows the receipt
writes, mode CHEQUE, `abj_cheque_id` set), `apd_voucher_id` = this voucher.
The panel therefore shows the bills band only under ON_CLEARING (and on a
re-present, §3.4).

Refused: a row not DEPOSITED; a clear date < deposit date (`ck_apd_seq`); a
second clearing (`ux_avh_src`).

### 3.3 BOUNCE — DEPOSITED → BOUNCED — one journal voucher, the bills reopen, the party is charged

Panel: bounce date (≥ deposit date), reason (picklist: *Funds insufficient ·
Payment stopped by drawer · Signature differs · Account closed · Post-dated
presented early · Stale · Other* + free text — `apd_bounce_reason` NOT NULL),
**charged by bank** (what the bank debited us), **charge to party** (what we
levy — defaults from a setting, may be zero).

**ON_RECEIPT** — one voucher `ChqBnc` dated the bounce date (C2):

| # | ledger | Dr/Cr | amount | `av_role` | `av_opp_ledger_id` |
|---|---|---|---|---|---|
| 1 | party | DR | 18,500 + 300 | — | NULL (two things face it) |
| 2 | Cheques In Hand | CR | 18,500 | — | party |
| 3 | Bank Charges | DR | 250 | `BANK_CHARGES` | own bank |
| 4 | own bank (`apd_bank_ledger_id`) | CR | 250 | — | Bank Charges |
| 5 | Bounce Charges Recovered | CR | 300 | `BOUNCE_CHARGES_RECOVERED` | party |

Σ DR 19,050 = Σ CR 19,050. `avh_against_voucher_id` = the receipt / PDC
voucher; `avh_src_*` NULL (a cheque can bounce twice — C10); `avh_remarks` =
'Bounced 221870 · Funds insufficient'.

Then, in the same transaction:

1. **The bills reopen** — for every `acc_bill_adjustment` row with
   `abj_cheque_id = this` (and `abj_cheque_acc_year`), one negative row:
   same `abj_bill_id / _acc_year`, same `abj_adj_type`, `abj_amount =
   −original`, `abj_reversal_of_id` = the original, `abj_reversal_reason` =
   the bounce reason, `abj_voucher_id` = the ChqBnc voucher, `abj_adj_date`
   = bounce date, `abj_settlement_mode = 'CHEQUE'`, `abj_cheque_id` kept
   (`ck_abj_cheque_mode` is satisfied by `abj_reversal_of_id`).
   `tr_abj_refresh_balance` reopens bil00022 to 12,500 and bil00025 to 6,000.
2. **The cheque's on-account credit is undone** (C4) — if the cheque's
   remainder became an ADVANCE bill (`abl_src_doc_type = 'RECEIPT'` and the
   PDC voucher, or the receipt, as `abl_voucher_id`, apportioned to this
   cheque by `abj_tender_id`): every `ADVANCE_ADJUST` pair that already
   consumed it is reversed (negative pairs, `abj_reversal_of_id`) — the
   invoices it was applied to reopen — and then the ADVANCE bill is
   soft-deleted (`abl_is_deleted`, `abl_is_active = false`). The alternative,
   refusing until someone reverses by hand, leaves a bounced cheque
   unrecorded while the bank has already debited us.
3. **The party's charge becomes receivable** (C3) — one `acc_bill_balance`
   row: `abl_bill_type = 'JOURNAL'`, `abl_dr_cr = 'DR'`, `abl_bill_amount` =
   300, `abl_doc_refno` = 'BNC/221870', `abl_doc_date` = bounce date,
   `abl_voucher_id / _type_id / _no / _date / _refno` = the ChqBnc voucher
   (`ck_abl_voucher`: not OPENING ⇒ set), `abl_src_module = 'ACCOUNTS'`,
   `abl_src_doc_type = 'PDC'`, `abl_src_doc_id = apd_id`, `abl_src_acc_year`,
   `abl_branch_id` = the cheque's, `abl_credit_days = 0`. It ages from the
   bounce date and the next receipt settles it like any bill — GoFrugal's
   "debit note against the party", as a bill rather than a document.
4. **The register** — `apd_status = 'BOUNCED'`, `apd_bounce_date`,
   `apd_bounce_reason`, `apd_bounce_charges` = 250, `apd_bounce_voucher_id =
   apd_charge_voucher_id` = the ChqBnc voucher (both, so either lookup finds
   it), `apd_status_on/by`; `txn_status_log` DEPOSITED → BOUNCED.

**ON_CLEARING** — nothing was posted at receipt, so steps 1–2 have nothing to
reverse; only step 3 (the party's charge, and the bank's charge as legs 3–4 in
a ChqBnc voucher without rows 1–2, 5 folded to the party leg) and step 4.

Refused: a row not DEPOSITED; bounce date < deposit date; a charge to the
party without the `BOUNCE_CHARGES_RECOVERED` mapping; a bank charge without
`BANK_CHARGES`.

### 3.4 RE-PRESENT — BOUNCED → DEPOSITED — the same cheque, again

The bounce took the in-hand entry out, so presenting again **re-issues** it
(C5), and Clear stays one shape:

    re-issue voucher   type Rct, dated the re-present date, avh_against_voucher_id = the ChqBnc voucher, avh_src NULL:
                       DR Cheques In Hand apd_amount / CR party apd_amount
    allocations        fresh acc_bill_adjustment rows, mode CHEQUE, abj_cheque_id = this, abj_is_post_dated = false —
                       auto FIFO over the party's open items (the reopened bills come first by due date), editable on the panel
    then the DEPOSIT writes of §3.1: apd_bank_ledger_id, apd_deposit_date, slip, apd_present_count = 2, status DEPOSITED
    txn_status_log     BOUNCED → DEPOSITED (re-presented)

The bounce charge bill (§3.3 step 3) stays open — it is the party's to pay.
Under ON_CLEARING there is nothing to re-issue: deposit again, present_count 2.

### 3.5 REPLACE — BOUNCED | HELD → REPLACED, plus a new HELD row

The customer gives a new cheque (often for amount + charge):

    old row     apd_status = 'REPLACED', apd_replaced_by_id / _acc_year = the new row (ck_apd_replaced), status_on/by
    new row     a fresh acc_pdc_register row: received today, its own instrument no / date / bank / amount,
                apd_acc_year = the current year, apd_tender_id NULL (it came through this screen, not a receipt),
                apd_status 'HELD', apd_voucher_id = its re-issue voucher
    re-issue    as §3.4 — Rct, DR Cheques In Hand / CR party for the NEW amount, fresh allocations (the reopened bills and,
                if the new amount covers it, the bounce-charge bill) — dated the new cheque's date if post-dated, exactly as
                the receipt does (own voucher, bills settle on maturity)
    from HELD   (never bounced): the old row's in-hand entry is reversed first (a return without charges, §3.6),
                then the new row is issued
    txn_status_log  on both rows

### 3.6 RETURN / CANCEL — HELD → RETURNED | CANCELLED — before the bank ever saw it

Handed back to the customer, or keyed in error. Refused once DEPOSITED — a
cheque with the bank is bounced or cleared, never cancelled.

    apd_status = 'RETURNED' | 'CANCELLED', apd_cancel_date, apd_cancel_reason (ck_apd_cancelled), status_on/by
    ON_RECEIPT   a reversal voucher (Rct, dated today, avh_against_voucher_id = the receipt / PDC voucher):
                 DR party / CR Cheques In Hand; negative abj rows as §3.3 step 1; the ADVANCE as §3.3 step 2 (C6)
    ON_CLEARING  the register row alone
    txn_status_log

### 3.7 What no action does

- Edit a register row's instrument no / date / amount / party — those came
  from the receipt; a wrong cheque is RETURNED and re-keyed on a new receipt.
- Delete a row. `apd_is_deleted` exists and stays false; CANCELLED is the
  deletion.
- Touch `acc_opening_balance`, `acc_voucher_doc_register`, or
  `acc_voucher_cheques`.

---

## 4. The register list and the due buckets

The list is a configured grid over `acc_pdc_register` joined to the party,
the tender row and the vouchers — `TxnMainView`-style, multi-select. Per row:
received on, party, cheque no, drawer bank, instrument date, **due bucket**,
amount, status, receipt · vouchers (drill-throughs), deposited (bank · date ·
slip), present count.

The due bucket is `vouchers.sql`'s, computed in the SELECT, never stored:

    FUTURE      apd_instrument_date > today
    DUE_TODAY   = today
    OVERDUE     matured, still HELD / DEPOSITED
    STALE       apd_instrument_date + 3 months < today — the bank will refuse it; Return or Replace

The summary strip: IN HAND (HELD), WITH THE BANK (DEPOSITED), DUE THIS WEEK,
MATURED-NOT-DEPOSITED, STALE, BOUNCED-OPEN, CLEARED THIS MONTH — and the tie
the screen keeps honest: **Cheques In Hand ledger balance = IN HAND + WITH
THE BANK** (under ON_RECEIPT). The spec's `vw_party_pdc_summary` is the
per-party version (it is not deployed — zero views live).

---

## 5. Rules the DATABASE enforces

| constraint | forces |
|---|---|
| `ck_apd_status` / `ck_apd_tra_type` / `ck_apd_instrument` / `ck_apd_posting_mode` | the vocabularies |
| `ck_apd_deposited` / `ck_apd_cleared` / `ck_apd_bounced` / `ck_apd_replaced` / `ck_apd_cancelled` | each status has its dates, voucher, reason |
| `ck_apd_seq` | clear / bounce date ≥ deposit date |
| `ck_apd_posting` | ON_RECEIPT rows name their voucher |
| `ck_apd_self` / `fk_apd_replaced_by` | a replacement points at a real, different row |
| `ux_apd_instrument` | one cheque number per party per year (CANCELLED excluded — a re-keyed cheque may reuse it) |
| `ux_avh_src` | one clearing voucher per cheque (C10) |
| `ck_abj_reversal_sign` / `ck_abj_cheque_mode` | reversals are negative and still name the cheque |
| `ck_abl_voucher` | the charge bill names the ChqBnc voucher |
| `ck_avh_balanced` (once derived — receipt plan §2.4) | every voucher here balances at POSTED |

## 6. Rules ONLY the service can enforce

1. The state machine of §1 — every action checks the current status under `FOR UPDATE` and refuses anything else by name.
2. Deposit date ≥ instrument date; clear / bounce / return dates in an OPEN, unlocked year.
3. Cheques In Hand comes from the cheque's own tender row, never from today's master.
4. Bounce step 2's cascade (C4) runs entirely or not at all — one transaction.
5. A charge to the party needs `BOUNCE_CHARGES_RECOVERED` mapped; a bank charge needs `BANK_CHARGES`; unmapped → refused by role name.
6. Re-present / replace allocations: per bill ≤ pending under the row lock, same as the receipt.
7. Every transition writes `txn_status_log` (C9); `apd_status_on/by` is the cache.
8. ON_CLEARING is honoured per ROW (`apd_posting_mode`), not per setting — a cheque received under one mode is finished under the same mode.

---

## 7. Deferred, with the hook

| what | hook |
|---|---|
| Bank reconciliation screen (GoFrugal step 4) | `av_recon_date` set at CLEAR — **deliberately out (user, 2026-09-14)** |
| Issued Cheques (menu 52) | same table, `apd_tra_type = 'P'`, cheque-book leaves, printing — own plan |
| Cheque printing (ours) | with 52 |
| Interest / penalty beyond the charge | the charge bill is a JOURNAL bill; interest is `INTEREST` bills (receipt §9) |
| Formal deposit-slip layouts per bank | dataset 17; one generic layout first |

## 8. Decisions — all taken

| # | decision | taken |
|---|---|---|
| C1 | clearing voucher | new type `ChqClr`, nature CONTRA |
| C2 | bounce | ONE voucher `ChqBnc` (JOURNAL): reversal + both charges; `apd_bounce_voucher_id = apd_charge_voucher_id` |
| C3 | party's charge | a JOURNAL DR bill, src PDC, role `BOUNCE_CHARGES_RECOVERED` (new) |
| C4 | bounce cascade | reverses adjustments that consumed the cheque's ADVANCE, then soft-deletes it |
| C5 | re-present / replace | re-issue: Rct DR Cheques In Hand / CR party + fresh FIFO allocations |
| C6 | return / cancel | bounce without charges (ON_RECEIPT); register only (ON_CLEARING); refused once DEPOSITED |
| C7 | granularity | deposit bulk (one slip); clear and bounce per cheque |
| C8 | deposit slip | printing-schema dataset keyed by (bank, date, slip no) |
| C9 | history | `txn_status_log` written on every transition |
| C10 | idempotency | clearing carries `src ('ACCOUNTS','PDC', apd_id)`; bounce / re-issue carry no src, link via `apd_*_voucher_id` and `avh_against_voucher_id` |

## 9. Prerequisites — collected

1. Everything in `../receipt/plan-backend-receipt.md` §2 (this screen assumes it: `Rct`, `av_role`, the balance and totals triggers, `BANK_CHARGES`, `fk_abj_cheque` → `acc_pdc_register`, `ck_apd_dates` loosened).
2. The two voucher-type rows (§2).
3. `BOUNCE_CHARGES_RECOVERED` role + one map row per company (§2).
4. A configured grid for the register (§4); a deposit-slip dataset (C8).
5. Menu 51 visible.
6. Settings: `accounts.bounce_charge_to_party` (default amount levied, may be 0); `accounts.bounce_reasons` (the picklist, JSON list).
