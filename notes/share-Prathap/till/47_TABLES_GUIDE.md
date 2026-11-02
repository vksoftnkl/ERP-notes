# 47_till_management.sql — table guide

**For:** reading file 47 table by table: what each table is, why it exists, what every column is
used for, and how the tables connect. **Date:** 2026-10-07.

**Diagrams in this folder:**
- `47_tables_map.png`: all 16 tables, their nature, their key columns and the links between them.
- `47_session_flow.png`: one session (C01-261007-02) from day open to HO review, showing which rows each step writes.

**Where the rules come from:** `TILL_DESIGN.md`. File 47 is §6 of it. Later files change a few things, flagged inline:
- **[48]** = `48_till_receipt_payment_expense.sql`
- **[49]** = `49_noncash_tender_control.sql`
- **[rev2]** = the review of 2026-10-07 (`plan-till-review-rev2.md`), whose SQL changes are already in 47 / 48 / 49.

---

## 0 · Read this first

### 0.1 The 16 objects at a glance

| # | Table | Prefix | Nature | One line |
|---|---|---|---|---|
| 1 | `till_safe` | tsf_ | master | the store safe / cash office the drawers empty into |
| 2 | `till_counter` | tcn_ | master | a billing lane with one cash drawer |
| 3 | `till_reason` | trs_ | master | the controlled list of reasons, per category |
| 4 | `till_denomination` | tdn_ | master | the notes and coins a count offers |
| 5 | `till_approval_rule` | tar_ | master | **when** an event needs a second person |
| 6 | `till_approval_authority` | taa_ | master | **who** may approve what, up to how much |
| 7 | `acc_tender_types` (existing) | ttm_ | master, +2 columns | **how** each tender is closed |
| 8 | `till_business_day` | tbd_ | transaction | one trading date per branch: the store Z |
| 9 | `till_session` | tss_ | transaction, **the hub** | one cashier × one counter × one stretch of time |
| 10 | `till_count` | tct_ | transaction | every time money is counted (all attempts kept) |
| 11 | `till_count_line` | tcl_ | transaction | the lines of a count: per note or per tender |
| 12 | `till_cash_movement` | tcm_ | transaction | every non-sale cash move, one voucher each |
| 13 | `till_variance` | tvr_ | transaction | per session × tender: the gap and what was decided |
| 14 | `till_approval` | tap_ | transaction | one exception request, decided once |
| 15 | `till_event` | tev_ | log, insert only | the append-only till journal |
| 16 | `till_session_review` | tsr_ | log, cloud only | HO's verdict on a session or a day |

(16 new tables plus 2 columns on an existing one.)

**[49]** adds two more tables: `acc_settlement_import` (one card / UPI payout, or one bank-statement period) and
`acc_settlement_line` (one provider line and what it matched). It also adds `tnd_statement_format` on
`acc_tender_master` and `tvr_rows` on `till_variance`; see `plan-noncash-tender-control.md`.

### 0.2 What "nature" means here

- **Master:** set up once, changed rarely, and **not partitioned**. `company_id NULL` = shared by every
  company, on purpose (the shipped reasons, denominations and rules are rows like that).
- **Transaction:** written while the store trades. **LIST-partitioned by `acc_year`**, so the
  primary key is always `(id, acc_year)` and a foreign key to one of them carries both columns.
  The year roll is the existing `fn_create_year_partitions` sweep. That is why the tables live in
  schema `accounts` and not in a new schema.
- **Log:** insert only. `till_event` has exactly one writing service. `till_session_review` is the
  only table written at the **cloud**; everything else is written by the **store** server.

### 0.3 House rules the whole file keeps

1. **No function, no trigger, no view.** NestJS (TillSessionService, TillLedgerService …) writes
   every row. The DB keeps CHECKs, GENERATED columns, FKs and indexes only.
2. **Amounts are positive.** Direction is a column (`tcm_kind`, `tvr_variance` sign, Dr/Cr in the
   voucher), never a negative amount.
3. **Offline:** every row has one writing site. That is why many links are **id columns without a
   foreign key**: rows of an offline store reach the cloud in any order, and an FK would reject them.
   The service checks those ids when it posts.
4. **GENERATED columns** (`tss_*_variance`, `tvr_variance`, `tcl_amount`, `tbd_cash_variance`) are
   computed by PostgreSQL and must never be in a sync push.
5. **Frozen totals:** the close writes counts and sums onto the session / day **once** and never
   recomputes them. The live figures come from the source rows.

### 0.4 Columns every table has (not repeated below)

| Column | Use |
|---|---|
| `*_id` | uuid v7 primary key (`uuidv7()` default) |
| `*_company_id`, `*_branch_id` | the scope: which company / store the row belongs to |
| `*_tenant_id` | reserved for multi-tenant cloud; NULL today |
| `*_acc_year` | the partition key on transaction tables, e.g. `2026-2027` |
| `*_is_active` | masters: switched off = not offered, history kept |
| `*_is_deleted` | soft delete; every unique index is `WHERE is_deleted = false` |
| `*_sync_date` | NULL = not yet pushed to the cloud |
| `*_created_on/by`, `*_modified_on/by` | audit stamps |
| `*_remarks` / `*_notes` | free text |
| `*_print_count` | how many times the slip / Z was printed |
| `*_device_id` | the device (fixed.device_master) the action came from |

---

## 1 · Masters

### 1.1 `till_safe` (tsf_): the safe / cash office

**Purpose:** where drawer cash goes: drops, pickups and the close hand-over. It is also where a float
comes from. A branch can have several safes; one is the default.
**Links:** `tsf_ledger_id` → `acc_ledger_master` (FK). `till_counter.tcn_safe_id`, `till_cash_movement.tcm_safe_id`
and `till_count.tct_safe_id` point here.

| Column | Use |
|---|---|
| `tsf_code`, `tsf_name` | e.g. S1 / Main safe; code unique per branch (case-insensitive) |
| `tsf_ledger_id` | the ledger this safe posts to: the **SAFE_CASH** role ledger. Every TFlt / TDrp / CRem voucher hits it |
| `tsf_insured_limit` | 0 = none. Above it, day close asks for a remittance (CRem) |
| `tsf_is_default` | the branch default safe; unique index allows one per branch |

### 1.2 `till_counter` (tcn_): the lane and its drawer

**Purpose:** the **drawer** is what is accountable, not the PC. A dead PC is replaced by re-pointing
`tcn_device_id`, and the history stays on the counter.
**Links:** `tcn_device_id` → `fixed.device_master` (FK, SET NULL). `tcn_safe_id` → `till_safe` (FK).
`till_session.tss_counter_id` points here (FK).

| Column | Use |
|---|---|
| `tcn_code`, `tcn_name` | C01, EXP1, RET; becomes the first part of the session number |
| `tcn_kind` | POS · EXPRESS · RETURNS_DESK · SERVICE_DESK · CASH_OFFICE · MOBILE · SELF_CHECKOUT |
| `tcn_drawer_mode` | DRAWER (fixed) · TRAY (insert travels with the cashier) · NONE (cashless lane: a session still opens, nothing is counted by note) |
| `tcn_device_id` | the PC bolted to it (Desktop/Mobile only, never a Web device); NULL = any device may claim it; one device drives one counter (unique). Claim rule: `plan-till-counter-claim.md` |
| `tcn_safe_id` | where its drops go; NULL = the branch default safe |
| `tcn_default_float` | float offered at session open |
| `tcn_cash_alert_limit` | cash in drawer above this shows "pickup due" (0 = off) |
| `tcn_cash_block_limit` | above this, billing stops until a pickup (0 = off); must be ≥ the alert limit |
| `tcn_z_last_no` | running Z number, never reset. The session close writes it +1 into `tss_z_no` |
| `tcn_requires_session` | false = this counter may bill without an open session |
| `tcn_sort_order` | order in lists |

### 1.3 No `till_operator`: the till reads the user and the employee

**Dropped 2026-10-07** (`plan-till-operator-to-user.md`). Every fact the till needs about a person already
has a home, so there is no till profile table:

| Need | Where it lives |
|---|---|
| sign-in and approval PIN | `user_master.usr_pin_hash` (set through the user API, notes 95) |
| which employee this user is | `user_master.usr_employee_id` |
| where a RECOVER shortage is debited | `employee_master.emp_loan_ledger_id`: the employee's **staff advance** ledger, created with the employee (notes 95) |
| may this user open a till | a menu right (`user_menus`) on the Open Till menu |
| what this user may approve | `till_approval_authority` (§1.7) |
| the float | the counter's default; a different float at open is a FLOAT_MISMATCH approval |

### 1.4 `till_reason` (trs_): controlled reasons

**Purpose:** 3.0 hard-coded six strings in the client. Here every "why" is a row, grouped by category,
with its own rules. Shipped rows have company NULL.
**Links:** `trs_ledger_id` → ledger (FK). Used by `till_cash_movement`, `till_variance` and `till_approval`
(FKs), and by `till_session.tss_force_reason_id` and `till_event` (ids).

| Column | Use |
|---|---|
| `trs_category` | VARIANCE · FLOAT_MISMATCH · PAID_OUT **[48: → EXPENSE]** · PAID_IN · PICKUP · NO_SALE · SUSPEND · FORCE_CLOSE · REOPEN · SESSION_VOID · MOVEMENT_VOID · DAY_REOPEN · REPRINT · VOID_BILL · VOID_LINE · PRICE_OVERRIDE · REFUND · **[49] NONCASH** |
| `trs_code`, `trs_name` | COUNT_ERROR / Counting error; unique per company + category |
| `trs_ledger_id` | PAID_OUT **[48: EXPENSE]** / PAID_IN: the default counter-ledger |
| `trs_needs_note` | the screen insists on a note |
| `trs_needs_ref` | the screen insists on a reference (supplier bill no) |
| `trs_max_amount` | a cap of the reason itself (0 = none); the approval rule still applies |

### 1.5 `till_denomination` (tdn_): notes and coins

**Purpose:** the rows of a cash count. 3.0 hard-coded them in the `.ui`. INR ₹500 … ₹1 is shipped.
**Links:** `till_count_line.tcl_denomination_id` → here (FK).

| Column | Use |
|---|---|
| `tdn_currency` | INR |
| `tdn_value`, `tdn_kind` | 20 NOTE and 20 COIN are two rows (unique on value + kind) |
| `tdn_label` | what the count screen shows: ₹500, ₹20 coin |
| `tdn_bundle_qty` | notes per strapped bundle (100); 0 = not bundled. Lets a count enter "3 bundles" |
| `tdn_valid_to` | a withdrawn note stops being offered after this date |

### 1.6 `till_approval_rule` (tar_): **when** a second person is needed

**Purpose:** one row per event per scope says whether that event needs approval, and at what level.
**Resolution:** branch row > company row > shipped row (both NULL), then the latest `tar_effective_from`
on or before the business date.
**Links:** `till_approval.tap_rule_id` (id), and a copy of the row is kept in `tap_rule_snapshot`.

| Column | Use |
|---|---|
| `tar_company_id`, `tar_branch_id` | both NULL = shipped; company only = company override; both = branch override |
| `tar_event_code` | FLOAT_MISMATCH · CASH_VARIANCE · NONCASH_VARIANCE · PAID_OUT **[48: EXPENSE, + CASH_PAYMENT]** · PAID_IN · PICKUP · TOP_UP · NO_SALE · CASH_LIMIT_OVERRIDE · SUSPEND_LONG · FORCE_CLOSE · SESSION_REOPEN · SESSION_VOID · MOVEMENT_VOID · RECOUNT · DAY_CLOSE_EXCEPTION · DAY_REOPEN · SAFE_VARIANCE · REMITTANCE · **COUNTER_RELINK** (counter claim), plus reserved VOID_BILL … RETENDER for the void module · **[49] + NONCASH_WRITE_OFF, SETTLEMENT_RESOLVE** |
| `tar_mode` | NEVER · ALWAYS · OVER_AMOUNT · OVER_COUNT · OVER_PERCENT |
| `tar_threshold_amount / _count / _percent` | the limit the mode compares against (count is per session) |
| `tar_channel` | COUNTER (PIN on the cashier's screen) · REMOTE (inbox / mobile) · EITHER |
| `tar_min_role` | the lowest authority level that may approve |
| `tar_two_person` | needs two different approvers |
| `tar_allow_self` | the requester may approve their own request |
| `tar_blocks_till` | true = the counter waits; false = record now, review later |
| `tar_expire_minutes` | a PENDING request expires after this (0 = never) |
| `tar_effective_from` | a rule change takes effect from this date; history stays |

**Shipped defaults (examples):**
- PAID_OUT over 500, SUPERVISOR **[48: EXPENSE over 500; + CASH_PAYMENT over 1,000]**.
- NO_SALE over 3 per session.
- CASH_VARIANCE over 10.
- FORCE_CLOSE always, STORE_MANAGER.
- COUNTER_RELINK always, SUPERVISOR at the counter.
- DAY_REOPEN always, AREA_MANAGER, channel **EITHER [rev2]** (was REMOTE: no reopen with the internet down).
- SAFE_VARIANCE two-person.

### 1.7 `till_approval_authority` (taa_): **who** may approve

**Purpose:** a grant per approver: the level, which events, up to what amount, from where. This table is
the only place a user's approval level lives.
**Links:** `taa_user_id` = `user_master.usr_id`. When an approval is decided, the decider must hold a
matching, valid grant.

| Column | Use |
|---|---|
| `taa_user_id` | the approver |
| `taa_company_id`, `taa_branch_id` | branch NULL = every branch of the company (area manager); company NULL = group-wide (HO finance) |
| `taa_role` | the level this grant gives: SUPERVISOR … HO_FINANCE |
| `taa_event_code` | NULL = every event |
| `taa_max_amount` | NULL = no ceiling. A supervisor up to 500, a manager up to 5,000 … |
| `taa_can_remote` | may approve from the inbox / mobile, not only by PIN at the counter |
| `taa_valid_from`, `taa_valid_to` | a temporary grant (holiday cover) |

### 1.8 `acc_tender_types` (existing, ttm_): two columns added

**Purpose:** each tender type says how it is closed at the till.

| Column | Use |
|---|---|
| `ttm_close_mode` | **DENOM**: counted note by note (CASH). **SLIPS**: slips counted + amount + batch (CARD, CHEQUE, VOUCHER). **STATEMENT**: nothing to count at the till, reconciled later with the provider statement (UPI, WALLET, BANK). **NONE**: not money in the drawer (CREDIT, TEMP_CR, LOYALTY, RRN) |
| `ttm_handover` | the instrument physically goes to the safe at close (cash, cheques, vouchers) |

---

## 2 · The business day

### 2.1 `till_business_day` (tbd_): one per branch per trading date

**Purpose:** every session hangs off a day. **Day close** is the store's Z and the moment the existing
`SALES_DAY_CLOSED` refusal starts: no re-tender, no cash cancel, no back-dated money documents.
The business date is **not** the calendar date. With `till.day_cutoff` 04:00, a sale at 00:40
belongs to yesterday.
**Links:** `till_session.tss_day_id` → here (FK). `till_cash_movement.tcm_day_id`, `till_event.tev_day_id` and
`till_session_review.tsr_day_id` are ids. Unique per (company, branch, date).

| Column | Use |
|---|---|
| `tbd_business_date` | the trading date. **[rev2]** `ck_tbd_year`: `tbd_acc_year` must be this date's April–March year; `uq_tbd_id_date` lets the session FK carry the date |
| `tbd_status` | OPEN → CLOSING (no new session may open; checklist runs) → CLOSED |
| `tbd_opened_on/by` | who opened it (the first session, or a manager when `till.day_auto_open` is false) |
| `tbd_closing_on` | when the manager started the close |
| `tbd_closed_on/by` | must be set exactly when the status is CLOSED (CHECK) |
| `tbd_z_no` | store Z number, running per branch, never reset |
| `tbd_reopen_count` | DAY_REOPEN count (area manager, remote) |
| `tbd_session_count`, `tbd_bill_count`, `tbd_net_sales` | **frozen at close**: the day's totals |
| `tbd_cash_expected`, `tbd_cash_counted` | **frozen**: sum over its sessions |
| `tbd_cash_variance` | GENERATED = counted − expected |
| `tbd_noncash_variance` | **frozen**: card / UPI gaps of the day |
| `tbd_safe_opening`, `tbd_safe_expected`, `tbd_safe_counted` | the safe for the day; counted NULL = no safe count (setting allowed it) |
| `tbd_remitted` | cash sent to the bank that day (CRem) |
| `tbd_exception_count` | approvals + force closes + no-sales in the day |
| `tbd_close_approval_id` | the DAY_CLOSE_EXCEPTION approval, when the checklist was not green |

---

## 3 · The session: the hub

### 3.1 `till_session` (tss_): one cashier, one drawer, one stretch of time

**Purpose:** the row every `*_session_id` in the product was always meant to point at:
- sales documents: sale bill, return, DC, quotation, order;
- money and posting rows: `acc_tender_detail`, `acc_voucher_header`, `acc_bill_adjustment`, `acc_temp_credit`;
- holds, loyalty and status: `txn_hold`, `loyalty_ledger`, `txn_status_log`.

Those columns stay FK-less (partitioned tables of other years; an offline push must never fail on
ordering). The service checks the id when it posts.

**Lifecycle:** OPEN ⇄ SUSPENDED → COUNTING → PENDING_APPROVAL → CLOSED, or VOIDED (opened in error,
nothing written). CLOSED → OPEN again only by SESSION_REOPEN (approval), and then it is counted again.

**Links:**
- **Foreign keys:** `tss_day_id` → day and `tss_counter_id` → counter.
- **Id columns (no FK):** operator → user; previous session; open/close count; the three approval ids;
  the force reason.
- **Pointing here:** `till_variance` (FK); count, movement, approval, event and review (ids).

**Unique:** one live session (OPEN / SUSPENDED / COUNTING) per counter. One live session per
**operator** is a service rule only, because two offline sites could each open one.

| Column | Use |
|---|---|
| `tss_day_id`, `tss_business_date` | the day it belongs to |
| `tss_counter_id`, `tss_device_id` | the lane and the device that opened it (**required**); money posts only from that device. Moved only by COUNTER_RELINK |
| `tss_operator_id` | user_master.usr_id: the accountable cashier |
| `tss_session_no` | printed number counter-date-seq, e.g. C01-261007-02 |
| `tss_day_seq` | the nth session of this counter on this date |
| `tss_status` | the lifecycle above |
| **open** | |
| `tss_opened_on` | start time |
| `tss_float_mode` | ISSUED (counted out of the safe) · CARRIED (left in the drawer by the previous session) · NONE |
| `tss_prev_session_id` | CARRIED / hand-over: whose drawer this was (required when CARRIED) |
| `tss_float_issued` | what the safe or the previous close says is in the drawer |
| `tss_float_counted` | what the cashier counted at open: **this is the opening figure for expected cash** |
| `tss_float_variance` | GENERATED = counted − issued |
| `tss_open_count_id` | the OPEN count |
| `tss_float_approval_id` | the FLOAT_MISMATCH approval |
| **during** | |
| `tss_suspend_count` | breaks / idle locks so far |
| `tss_suspended_on` | set exactly while SUSPENDED (CHECK) |
| **close** | |
| `tss_billing_ended_on` | the moment it stopped taking money |
| `tss_count_mode` | BLIND (cashier never sees expected; default) · OPEN |
| `tss_count_place` | COUNTER · CASH_OFFICE (drawer handed over sealed) |
| `tss_close_count_id` | the FINAL count; every attempt stays in `till_count` |
| `tss_count_attempts` | how many counts it took |
| `tss_counted_on/by`, `tss_witness_by` | who counted, and the second person if `till.close_witness` |
| `tss_closed_on/by` | set exactly when CLOSED or VOIDED (CHECK) |
| `tss_close_approval_id` | CASH_VARIANCE / NONCASH_VARIANCE / FORCE_CLOSE |
| `tss_force_closed`, `tss_force_reason_id` | a supervisor closed it for an absent cashier; reason required |
| `tss_reopen_count` | times reopened; the Z number is not reused |
| `tss_z_no` | = counter's `tcn_z_last_no` + 1 at close |
| **frozen at close** | |
| `tss_bill_count`, `tss_return_count`, `tss_receipt_count`, `tss_no_sale_count`, `tss_void_count`, `tss_void_line_count` | activity counts **[48: + payment, expense counts]** |
| `tss_net_sales` | net sales of the session |
| `tss_cash_expected`, `tss_cash_counted`, `tss_cash_variance` (GENERATED) | the cash result |
| `tss_noncash_expected`, `tss_noncash_counted`, `tss_noncash_variance` (GENERATED) | card / slips result |
| `tss_handed_over` | cash that went to the safe at close |
| `tss_float_left` | cash left in the drawer for the next session. **CHECK: handed_over + float_left = cash counted** |
| `tss_variance_status` | NONE · WITHIN_TOLERANCE · PENDING · ACCEPTED · RECOVER · INVESTIGATE; an index finds closed sessions nobody has decided |

---

## 4 · Counting

### 4.1 `till_count` (tct_): every time money is counted

**Purpose:** 3.0 deleted and re-inserted the count on every close, so a recount erased the first.
Here every attempt is its own row and stays. `tct_expected` is stored **even when the count is blind**,
so an auditor can see what the system expected at that moment.
**Links:**
- Ids it holds: `tct_session_id` (session) or `tct_safe_id` (a safe count); one of the two is required. `tct_movement_id` is the movement being counted.
- Pointing here: `till_count_line` (FK, cascade); `till_session.tss_open_count_id` / `tss_close_count_id`; `till_cash_movement.tcm_count_id`.

| Column | Use |
|---|---|
| `tct_kind` | OPEN · CLOSE · RECOUNT · HANDOVER · SURPRISE · FLOAT_ISSUE · PICKUP · DROP_VERIFY · SAFE |
| `tct_session_id` / `tct_safe_id` | whose money: a drawer or a safe |
| `tct_movement_id` | DROP_VERIFY / FLOAT_ISSUE / PICKUP: the movement counted |
| `tct_attempt_no` | 1, 2, … |
| `tct_is_final` | the attempt the close used; one final OPEN and **one final close-side count (CLOSE or RECOUNT) [rev2]** per session (unique) |
| `tct_is_blind` | the counter did not see the expected figure |
| `tct_counted_on/by`, `tct_witness_by` | who; the witness must be a different person (CHECK) |
| `tct_total_counted` | the total of its lines |
| `tct_expected` | what the system expected at count time |

### 4.2 `till_count_line` (tcl_): the lines of a count

**Purpose:** one line per denomination for cash. Other tenders get one line each, so two UPI QR codes
are two lines.
**Links:** `tcl_count_id` → `till_count` (FK, delete cascade). `tcl_tender_type_id` → `acc_tender_types` (FK).
`tcl_denomination_id` → `till_denomination` (FK). `tcl_tender_id` = `acc_tender_master` (id).

| Column | Use |
|---|---|
| `tcl_row_no` | order on screen |
| `tcl_tender_type_id`, `tcl_tender_id` | which tender this line is |
| `tcl_denomination_id` | cash lines only |
| `tcl_face_value` | the note value (0 when not a denomination line; CHECK keeps these consistent) |
| `tcl_qty` | pieces (cash) or slips (card) |
| `tcl_entered_amount` | typed total: slips, loose coins |
| `tcl_batch_ref` | card EDC batch no / cheque bundle ref |
| `tcl_amount` | GENERATED: face × qty for denominations, else the entered amount |

---

## 5 · Cash movements

### 5.1 `till_cash_movement` (tcm_): every non-sale cash move

**Purpose:** any money that moves through a drawer or the safe and is not a sale. Each POSTED row has
exactly one accounting voucher; cancelling it is a VOIDED status plus a reversal voucher, never a delete.

| Kind | From → To | Voucher | Dr / Cr |
|---|---|---|---|
| FLOAT_ISSUE, TOP_UP | safe → till | TFlt | Till cash / Safe |
| PICKUP, DROP, CLOSE_HANDOVER | till → safe | TDrp | Safe / Till cash |
| PAID_OUT **[48: retired → Expense voucher]** | till → ledger | TPOut | expense or party / Till cash |
| PAID_IN | ledger → till | TPIn | Till cash / income or party |
| REMIT | safe → bank / in transit | CRem | Cash in transit / Safe |
| SAFE_TRANSFER | safe → safe | CRem | other safe / this safe |
| EXCHANGE | till ↔ safe | none | change for a big note: nets to zero, counted both ways |

**"Till cash"** is the branch CASH tender's ledger (`acc_tender_master.tnd_ledger_id`). There is no
separate till ledger: a sale and a payout hit the same one, and the session is the sub-ledger via
`avh_session_id`.

**Links:**
- **Foreign keys:** `tcm_safe_id` → safe, `tcm_ledger_id` → ledger, `tcm_reason_id` → reason.
- **Id columns (no FK):** session, day, to-safe, count, approval, `tcm_voucher_id` → `acc_voucher_header`.

| Column | Use |
|---|---|
| `tcm_kind` | the table above |
| `tcm_doc_no`, `tcm_doc_date` | numbered from `acc_voucher_seq` for its voucher type; date = the business date |
| `tcm_day_id`, `tcm_session_id` | session is required except for REMIT / SAFE_TRANSFER (CHECK) |
| `tcm_safe_id`, `tcm_to_safe_id` | the safe side; to-safe only for SAFE_TRANSFER (CHECK) |
| `tcm_amount` | > 0 |
| `tcm_ledger_id` | PAID_OUT **[48: retired]** / PAID_IN counter-ledger; REMIT bank / transit (required for those, CHECK) |
| `tcm_reason_id` | required for PAID_OUT **[48: retired]** / PAID_IN / PICKUP |
| `tcm_ref_no`, `tcm_ref_date` | supplier bill no, deposit slip no |
| `tcm_party_name` | who took / gave the cash (free text) |
| `tcm_count_id` | the note-by-note count of what moved |
| `tcm_bag_no`, `tcm_seal_no` | tamper-evident bag for drops and remittances |
| `tcm_cit_agency` | the cash-in-transit company for REMIT |
| `tcm_done_by`, `tcm_witness_by` | who handled it; the witness must be someone else; **a PICKUP always has one [rev2] (`ck_tcm_pickup_witness`)** |
| `tcm_approval_id` | the approval that let it through |
| `tcm_voucher_id`, `tcm_voucher_acc_year` | the posted voucher (both or neither, CHECK); an index finds POSTED rows with no voucher |
| `tcm_status` | POSTED · VOIDED |
| `tcm_void_reason_id`, `tcm_voided_on/by`, `tcm_void_approval_id` | all required when VOIDED (CHECK) |

---

## 6 · Variance

### 6.1 `till_variance` (tvr_): the gap, per session × tender, and the decision

**Purpose:** for each tender of a session, what was expected, what was counted, and what was decided
about the difference. Stage **OPEN** = the float mismatch at open (cash only). Stage **CLOSE** = the
close. A row is never edited after it posts. A changed decision is a REVERSED row plus a new one.
**Links:** `tvr_session_id` → session (FK). `tvr_tender_type_id` → tender types (FK). `tvr_reason_id` → reason (FK).
Approval, voucher and `tvr_reverses_id` are ids. Only one live (non-reversed) row per session + stage + tender.

**Expected cash, the one formula (CHECK `ck_tvr_expected`):**
`expected = open + sales − refunds + receipts + moved_in − moved_out`
**[48: − payments − expenses]**

| Column | Use |
|---|---|
| `tvr_stage` | OPEN (float) · CLOSE |
| `tvr_tender_type_id`, `tvr_tender_id` | which tender |
| `tvr_open_amount` | the counted float (cash only: **[rev2] `ck_tvr_cash_terms`** keeps open / moved_in / moved_out at 0 on any other tender) |
| `tvr_sales_amount` | sale tenders taken (`acc_tender_detail`, Dr, this session) |
| `tvr_refund_amount` | refunds paid out |
| `tvr_receipt_amount` | receipts / advances / temp-credit collections at this counter |
| `tvr_moved_in` | TOP_UP + PAID_IN |
| `tvr_moved_out` | PICKUP + DROP + PAID_OUT **[48: PICKUP + DROP only]** |
| `tvr_payment_amount`, `tvr_expense_amount` | **[48]** bill-wise payments and expense vouchers out of this tender; the formula subtracts both |
| `tvr_txn_count`, `tvr_slip_count` | how many transactions / slips |
| `tvr_expected`, `tvr_counted` | the two figures |
| `tvr_variance` | GENERATED = counted − expected (negative = short) |
| `tvr_tolerance` | the tolerance that applied, as a snapshot |
| `tvr_treatment` | PENDING · WITHIN_TOLERANCE (auto) · EXPENSE (cash short & excess) · RECOVER (the cashier's staff advance ledger: user → employee → `emp_loan_ledger_id`) · SUSPENSE (under investigation) · RETENDERED (it was a wrong tender, fixed by re-tender, no posting). **The other side of the posting is that tender's ledger**: Till cash only for CASH, the clearing ledger for card / UPI **[rev2]** |
| `tvr_rows` | **[49]** jsonb: the tender rows a non-cash close variance covers (from the slip check), so a later settlement or write-off credits Tender suspense, not clearing |
| `tvr_reason_id` | required once a real gap is decided |
| `tvr_recovery_ledger_id` | RECOVER only: the staff advance ledger resolved at decision time, kept on the row |
| `tvr_approval_id` | the CASH_VARIANCE / NONCASH_VARIANCE approval |
| `tvr_voucher_id`, `tvr_voucher_acc_year` | the TVar journal |
| `tvr_status` | OPEN · POSTED · REVERSED |
| `tvr_reverses_id` | the row this one replaces |
| `tvr_decided_by/on` | required once the treatment is not PENDING |

---

## 7 · Approvals

### 7.1 `till_approval` (tap_): one exception request, decided once

**Purpose:** records who let an **exception** through: a variance, a payout over a limit, a drawer
opened with no sale. It is **not** a voucher approval step (vouchers stay DRAFT → POSTED → CANCELLED).
It is generic, so the void / price-override / reprint gates write here too, and "every override in a
session" is one query.

**The two channels:**
- **COUNTER:** the approver types their PIN on the cashier's screen. The store server checks it and
  writes the decision in the same call, so it works with the cloud down.
- **REMOTE:** the request waits as PENDING. An approver with `taa_can_remote` decides from the inbox,
  still through the store server.

**Links:**
- **Foreign key:** `tap_reason_id` → reason.
- **Id columns (no FK):** session, counter, the rule, and `tap_src_doc_*` (the document it is about: TILL_SESSION, TILL_MOVEMENT, SALE_BILL …).
- **Pointing here:** session (float / close approval), movement, variance, day and event.

| Column | Use |
|---|---|
| `tap_event_code` | the event; **[rev2] `ck_tap_event`** holds the same list as the rule |
| `tap_session_id`, `tap_counter_id` | where it happened |
| `tap_src_doc_type`, `tap_src_doc_id`, `tap_src_refno` | the document it is about |
| `tap_amount` | the amount at stake |
| `tap_count_in_session` | how many of this event the session already had (for OVER_COUNT) |
| `tap_rule_id`, `tap_rule_snapshot` | the rule that fired, and a JSON copy as it was then |
| `tap_required_role`, `tap_two_person`, `tap_self_allowed` | copied from the rule at request time (`ck_tap_role` **[rev2]**). With `till.single_operator` the service sets `tap_self_allowed` true |
| `tap_channel` | COUNTER · REMOTE |
| `tap_requested_by/on`, `tap_reason_id`, `tap_request_note` | the request |
| `tap_status` | PENDING · APPROVED · REJECTED · EXPIRED · CANCELLED |
| `tap_decided_by/on`, `tap_decided_device_id`, `tap_decision_note` | the decision; required when APPROVED / REJECTED. Decider ≠ requester unless allowed (CHECK) |
| `tap_second_by/on` | the second approver of a two-person rule: a third person (CHECK); an APPROVED two-person request must have one |
| `tap_expires_on` | from `tar_expire_minutes` |

---

## 8 · Logs

### 8.1 `till_event` (tev_): the append-only till journal

**Purpose:** everything that happened at a till, in order. 3.0 kept one text blob instead. **Insert
only**: TillEventService is the only writer. Client facts (drawer kicked, no-sale, X report, idle lock)
arrive in batches.
**Links:** ids to session, day, counter, device, user, source document, reason and approval.

| Column | Use |
|---|---|
| `tev_event_code` | DAY_* · SESSION_* · DRAWER_OPEN_SALE · NO_SALE · DRAWER_LEFT_OPEN · X_REPORT · Z_REPORT · REPRINT · CASH_ALERT / BLOCK / UNBLOCK · MOVEMENT_* · VARIANCE_DECIDED · APPROVAL_* · PRICE_OVERRIDE · LINE_VOID · BILL_VOID · RETENDER · REFUND_CASH · OFFLINE_START / END · LOGIN · LOGOUT · PIN_FAIL · **[48]** RECEIPT / PAYMENT / EXPENSE_POSTED, MONEY_DOC_CANCELLED · **[rev2]** SYNC_PENDING_AT_CLOSE, LATE_ARRIVAL, SESSION_DAY_ENDED · **[49]** DUPLICATE_REF_BLOCKED, SLIP_CHECK, SETTLEMENT_IMPORTED / POSTED, NONCASH_WRITTEN_OFF |
| `tev_event_on` | **device** time (when it happened); `tev_created_on` = when it arrived |
| `tev_session_id`, `tev_day_id`, `tev_counter_id`, `tev_user_id` | where and by whom |
| `tev_src_doc_type`, `tev_src_doc_id`, `tev_src_refno` | the document it is about (a bill for BILL_VOID, a movement …) |
| `tev_reason_id`, `tev_approval_id` | the reason given, and the approval that let it through |
| `tev_amount`, `tev_payload` | the figure, and any extra detail as JSON |
| `tev_client_seq` | the device's own sequence: a batch sent twice is not stored twice (unique per device) |

### 8.2 `till_session_review` (tsr_): HO's verdict, cloud only

**Purpose:** sync is push-only (store → cloud, upsert by PK). If HO updated a store-owned session row,
the store's next push would overwrite it. So HO's view lives in its own rows that no store writes.
**Links:** `tsr_session_id` or `tsr_day_id` (one is required). `tsr_ref_voucher_id` = the journal HO raised.

| Column | Use |
|---|---|
| `tsr_outcome` | OK · QUERY · ESCALATED · RECOVERY_ORDERED · WRITTEN_OFF · SUSPENSE_CLEARED · BANK_MATCHED |
| `tsr_reviewed_by/on`, `tsr_note` | who, when, why |
| `tsr_ref_voucher_id`, `tsr_ref_voucher_acc_year` | e.g. the journal that cleared a SUSPENSE variance |

---

## 9 · Seeds (§10 of the file): rows added to existing tables

| Into | What |
|---|---|
| `acc_ledger_role` | SAFE_CASH · CASH_IN_TRANSIT · CASH_SHORT_EXCESS · TENDER_SUSPENSE. **No TILL_CASH role on purpose:** the till cash ledger is the CASH tender's ledger, and two sources would drift. Each company maps these on the Ledger Map screen |
| `acc_voucher_types` | TFlt (float, contra) · TDrp (drop, contra) · TPOut (paid out, payment) · TPIn (paid in, receipt) · TVar (variance, journal) · CRem (remittance, contra) **[48: + ExpV] [49: + TSet]** |
| `acc_voucher_seq` | a number series for each new type, cloned from the Bil series rows |
| `app_setting_def` | 17 `till.*` settings, + `till.single_operator` **[rev2]** = 18; **[48]** adds 3, **[49]** adds 6 (27 in all): require_session, day_cutoff, day_auto_open, float_mode, handover_mode, blind_close, max_recounts, count_place, close_witness, cash_tolerance, noncash_tolerance, close_with_holds, variance_default, idle_lock_minutes, session_max_hours, day_close_safe_count, no_sale_reason |
| `till_reason` | shipped reasons per category (company NULL) |
| `till_denomination` | INR ₹500 … ₹1 |
| `till_approval_rule` | 19 shipped rules (company / branch NULL) |
| partitions | 2026-2027 for the 9 transaction tables; later years come from the existing year sweep |

---

## 10 · How the tables work together (the short story)

1. **Day:** the first session of the date creates `till_business_day` (OPEN).
2. **Open:** `till_session` (OPEN) gets its float as a FLOAT_ISSUE `till_cash_movement` (TFlt voucher). The
   cashier counts it into `till_count` / `till_count_line`. A mismatch raises a `till_approval` and an
   OPEN-stage `till_variance`.
3. **Trading:** bills, returns and receipts carry `*_session_id`. Their tender rows in
   `acc_tender_detail` (`td_session_id`) are where expected cash comes from. `till_event` logs the
   drawer, no-sales and X reports.
4. **Cash out:** pickups and drops are `till_cash_movement` rows, each with its voucher. **[48]** An expense
   is an Expense voucher (ExpV) and a supplier is a bill-wise Payment, both with tender rows in the session. The
   ones the `till_approval_rule` names need a `till_approval` from someone the `till_approval_authority`
   allows.
5. **Close:** billing ends (COUNTING) once the device has no unsent bills **[rev2]**, and the cashier counts blind; every attempt goes to `till_count`. The
   server computes expected and writes a CLOSE-stage `till_variance` per tender. Beyond tolerance it
   becomes a `till_approval` (PENDING_APPROVAL). On approval the variance posts (TVar), the drawer goes
   to the safe (CLOSE_HANDOVER, TDrp), and the session is CLOSED with its totals frozen and its Z number.
6. **Day close:** every session closed, nothing pending, the safe counted (SAFE count), and a remittance
   if needed. `till_business_day` is CLOSED with frozen totals.
7. **HO:** the cloud writes `till_session_review`.

**The check that proves the posting:** the Till cash entries with `avh_session_id` = the session net to
`tss_float_left` (minus the carried-in float). In the worked story: 44,450 − 44,450 = 0. See
`47_session_flow.png`.
