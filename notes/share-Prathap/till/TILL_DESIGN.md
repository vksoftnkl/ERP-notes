# Till Management — design (REV 1, 2026-10-07)

> **What this is.** The whole till / cashier-session architecture for NexERP, built for a
> **chain store**: counters, cashier sessions, blind close, cash pickups and drops, paid-outs,
> the store safe, bank remittance, the business day, tiered approvals, HO review — and the
> accounting each of those writes.
>
> **Files.** This document · `../47_till_management.sql` (schema, seeds, partitions — run
> after 46) · screen layouts are in §8 here (no PNG mockups yet; see §14).
>
> **3.0 was read for reference only** (user: *"no need to carry 3.0 legacy"*). Nothing is
> ported. §2 says what 3.0 did and what was wrong with it, so the same mistakes are not
> made again.
>
> **House rules applied throughout:** no DB function/trigger/view (NestJS owns logic);
> offline single-writer rows; amounts positive with direction in a column; company NULL =
> shared; new route family (`/api/v1/till/*`), no `/list` routes (grids), the
> `companyId · branchId · accYear · id` key on every verb; Decimal end to end.

---

## 0 · The decisions, on one page

| # | Decision | Why |
|---|---|---|
| D1 | **The accountability unit is the DRAWER, through a SESSION** = one cashier × one counter × one stretch of time. | A shortage must have one owner. A counter alone has many cashiers; a cashier alone moves between counters. |
| D2 | **A BUSINESS DAY per branch** sits above sessions. Day close is the store's Z and the point after which `SALES_DAY_CLOSED` refuses. | 3.0 had no day. Without one, "is the store closed for the 7th?" has no answer and late-night sales land on the wrong date. |
| D3 | **Blind close by default.** The cashier counts without seeing the expected figure. | A cashier who can see the expected cash can take exactly the excess. 3.0 showed it. |
| D4 | **The server computes "expected". The client never does.** | 3.0 computed expected and variance in the Qt client and the server stored whatever it was sent. |
| D5 | **Every count attempt is kept.** Recounts are new rows, never overwrites. | 3.0 deleted and re-inserted the count on every save, so a recount erased the evidence. |
| D6 | **Every non-sale cash movement is a document with its own accounting voucher**: float issue, top-up, pickup, drop, paid-out, paid-in, close hand-over, remittance. | 3.0's cash in/out (`pos_sess_pays`) **never reached the books** — checked: only `fx_pos_payments` / `fx_pos_session` ever touch it. |
| D7 | **Money moves in the session where the money physically moves.** A refund today of last week's bill comes out of *today's* drawer and is booked in today's session. | Otherwise a closed, counted session changes after it was counted. |
| D8 | **Approvals are a generic EXCEPTION gate** (rule → authority → decision), not a voucher approval step. Tiered by amount: supervisor → store manager → area manager → HO. | The chain needs levels and limits. The house rule "no approve step on vouchers" stays — this approves a variance, a payout, a drawer opened with no sale. |
| D9 | **The till cash ledger is the branch's CASH tender ledger** (`acc_tender_master.tnd_ledger_id`). The session is the sub-ledger, through `avh_session_id` (already a column). | One source of truth; a sale and a payout hit the same ledger. No ledger per counter (a 100-store chain would need 500). |
| D10 | **Non-cash differences are fixed by re-tender, not written off.** Only cash variance goes to Cash Short & Excess by default. | A "card short by 500" is almost always "keyed card, was UPI". `/bills/retender` already exists for exactly that. |
| D11 | **One writer per row.** The store server writes everything about its counters; the cloud (HO) writes only `till_session_review`. Remote approvals are decided *through the store's API*. | Sync is push-only, upsert by PK. An HO edit of a store row is overwritten on the next push. |
| D12 | Tables live in schema **`accounts`**, prefix `till_`. | `fn_create_year_partitions` sweeps public/sales/accounts only; a new schema would miss the year roll (the `print_log` trap). |

---

## 1 · What was read

| Source | What |
|---|---|
| 3.0 DB (`192.168.0.106/3.0`) | `pos_sess_logs` (34 rows), `pos_sess_pays` (23), `pos_sess_discr` (69), `pos_sess_denom` (81), `pos_sess_tender_detail_*` (0), `acc_sess_tenders`, `tender_master`, functions `app_utils.fx_pos_session` (flags 1–10) and `fx_pos_payments` (flags 1–3) |
| 3.0 Qt | `pos/Till/tillopeningsession.*`, `tillpaymententry.*`, `cashiermanagement.*`, `tillcashdenomination.*`, `tillsessionstatement.*`, `pos/EPOSGo/tillsessiondetail.*` (1,344 lines), `Init/loginmainwindow.cpp` (session resume at login), `Init/mainwindow.cpp`, `Sales/saleentry.cpp` |
| NexERP live (`ERP`) | every column named `*session*`, `*counter*`, `*device*`, `*till*`; `fixed.device_master`, `public.txn_hold`, `accounts.acc_tender_types/_master/_detail`, `audit.user_login_sessions`, `acc_voucher_seq`, `acc_ledger_role`, `app_setting_def`, `user_master` (`usr_pin_hash` exists), `employee_master` (`emp_loan_ledger_id` exists), `v_sync_tables`, `fn_create_year_partitions` |
| NestJS (`nexerp-api-ro`, read only) | `bill.service.ts`, `save-bill.dto.ts` (`sbSessionId`, `sbCounterId` nullable), `bill-counter-allocation.helper.ts`, `cloud-sync/README.md` |
| Qt client | `AppSession::sessionId()` — **a random uuid per app run** (`app_session.cpp:388`), sent on every sales header today |
| Share | `sales/SIX_TXN_CONTROLS.md` §7 (void + session, parked 2026-09-22), `sales/HANDOVER_endpoints.md` §12, `35_tender_void.sql`, `31_temp_credit.sql`, `sales/offline-sync-invariants.md` |
| GoFrugal | **from product knowledge, not a code reading** — §3 |

---

## 2 · 3.0 — what it did, and what was wrong

### 2.1 Tables

| 3.0 table | Held | Our equivalent |
|---|---|---|
| `pos_sess_logs` | one session: user, counter (PC name string), start/end, opening cash, closing cash, expected, cash diff, other diff, status bool, approved bool, reconciled bool, a text log | `till_session` (+ `till_event` for the log) |
| `pos_sess_pays` | cash in / cash out with an "expense type" string and an account id; cash-out stored **negative** | `till_cash_movement` (10 kinds, each posted) |
| `pos_sess_discr` | per tender at close: open, collected, count, expected, counted, diff, reason string, approved bool | `till_variance` (+ `till_count_line` for the count) |
| `pos_sess_denom` | per denomination at close: denom (varchar!), count, amount | `till_count` + `till_count_line` (any count event, not only close) |
| `pos_sess_tender_detail_*` / `acc_sess_tenders` | tender rows by session | already exists in NexERP: `acc_tender_detail.td_session_id` |

### 2.2 The flow

1. **Login** (`fusersettings` flag 6) returns the user's open `sess_id`; if one exists the user is asked "continue the session started at …?". If none and the PC is a session terminal, `TillOpeningSession` asks for opening cash only → `fx_pos_session` flag 1.
2. **Billing** stamps `global.sessionid` on the bill and the hold. The Payment button is hidden when there is no session — **the sale itself is not**.
3. **Cash in / out** (`TillPaymentEntry`) → `fx_pos_payments` flag 1. Delete = flag 3 (status false).
4. **Close** (`TillSessionDetail` / `CashierManagement`): the client loads tender totals (flag 3), adds cash in/out (flag 9), **computes expected and diff itself**, takes a denomination count, refuses if any hold bill is active (flag 10), then sends XML to flag 5, which UPDATEs the session, **DELETEs** denom and discr rows and re-INSERTs them, prints the session report.
5. **Approve** = an admin opens the same screen, ticks per-tender "approved" boxes (or **"Approve All"**) and saves through **the same flag 5**.
6. **Cash received** (flag 7) sets `sess_reconciled` and who collected it.

### 2.3 Defects — the list of things we must not repeat

1. **No business day.** Session `sr_no` = `max+1 WHERE sess_date::date = today` — **not scoped by branch or tenant**, and racy: two counters opening together get the same number.
2. **Expected and variance computed in the client**; the server stores what it is sent.
3. **"Approve All" overwrites counted with expected** for every non-cash tender and zeroes the diff — it destroys the evidence it is approving.
4. **Close DELETEs and re-INSERTs** the count and discrepancy rows. A recount erases the first count; an approver's save replaces the cashier's figures.
5. **Approval is one global boolean (`AdminUser`)** — no levels, no amount limits, and the person who closed can be the person who approves.
6. **Not blind.** The cashier sees expected cash while counting.
7. **Cash in/out never posts to the books.** Only the two session functions touch `pos_sess_pays`. Live data shows *"Cash In · Customer Receipt"* rows — customer money that never reached the customer's ledger.
8. **Deleting a payout** needs no reason, no approval, any user.
9. **Signed amounts** (cash-out negative) and **doubles** for money (`toDouble`, `'g', 13`).
10. **The audit log is a text blob** (`sess_log_rec`, concatenated lines).
11. **The session belongs to a PC-name string**; a closed session is matched by `ClientPCName`.
12. **Variance reasons hard-coded in the client** — including "Fraud or Theft", picked from a combo. Live: **52 of 69** discrepancy rows have no reason at all.
13. **States contradict each other**: 3 sessions are "approved" but not "cash received"; three booleans instead of one state.
14. **Held bills block the close** with no way to release them.
15. **No safe, no float issue, no pickup/drop distinction, no remittance, no force-close, no reopen, no X/Z report numbering, no denomination master, no cash-limit alert, no no-sale logging.** Abandoned sessions simply stay open until somebody logs in at that PC.
16. `sess_duration` is a **timestamp** column for a duration and is never written.
17. **Billing does not require a session** (`session_id` 0 is a legal bill).

Kept as ideas (re-designed, not ported): resume-at-login, refuse close with holds (now a setting), a per-tender close table, the collected-by name, denomination popup on the cash row.

---

## 3 · GoFrugal-class POS vs 3.0 vs this design

> GoFrugal from product knowledge (their retail POS / HQ): counter master, cashier login,
> day begin / day end, cash pickup, cash declaration by denomination, supervisor password
> overrides, X/Z reports, short & excess reports, HQ day-end status. Not a code reading.

| Capability | 3.0 | GoFrugal-class | NexERP design |
|---|---|---|---|
| Counter / terminal master | PC name string | ✔ | `till_counter` (drawer mode, device, float, cash limits, Z counter) |
| Day begin / day end | ✘ | ✔ | `till_business_day`, cut-off time, day-close checklist |
| Cashier session | ✔ basic | ✔ | `till_session`, 6 states |
| Opening float, counted | typed only | ✔ | issued from safe **or** carried; counted by denomination; mismatch → approval |
| Blind close | ✘ | option | **default on** |
| Denomination count | close only, hard-coded | ✔ | master + every count event (open, close, recount, pickup, drop, safe) |
| Recount history | ✘ (overwrites) | partial | every attempt kept, limited, approval past N |
| Cash pickup / safe drop | ✘ | ✔ | PICKUP (supervisor + witness) and DROP (sealed bag, verified later) |
| Cash limit alert | ✘ | ✔ | alert limit + hard block limit per counter |
| Paid-out / paid-in | ✔ not posted | ✔ posted | posted vouchers TPOut / TPIn, reasons, approval over limit |
| Supervisor override | admin flag | password | rule × authority × PIN/badge; tiered amounts; two-person; no self-approval |
| Remote approval | ✘ | partial | REMOTE channel via store API, inbox |
| No-sale drawer open | ✘ | ✔ | logged, reason, approval after N per session |
| X report / Z report | print only | ✔ | X logged; Z numbered per counter and per store, never reset |
| Variance treatment | reason string | short/excess | expense / recover from cashier / suspense / re-tendered — each posted |
| Shift hand-over on one counter | ✘ | ✔ | FULL_CLOSE or HANDOVER_COUNT (setting) |
| Force close / reopen | ✘ | ✔ | both, with approval and reason |
| Held bills at close | block | ✔ | RELEASE to branch pool (default) or BLOCK |
| Store safe & remittance | ✘ | ✔ | `till_safe`, safe count, REMIT with bag/seal/CIT, cash in transit |
| HO cash position | ✘ | ✔ | chain dashboard (§10) + HO review rows |
| Cashier performance | ✘ | ✔ | bills/hour, voids, no-sales, variance history per operator (§10) |
| Offline store | n/a | ✔ | single-writer design (§5.9) |

**Beyond GoFrugal-class** (chain-store gaps worth closing): the till ledger is a real sub-ledger
(session on every voucher), approvals are one table shared with the future void / price-override
gates, HO review cannot be overwritten by a store, and the day close refuses while anything
(variance, approval, unposted movement) is pending.

---

## 4 · What NexERP already has (do not rebuild)

- **`session_id` on every money table**, uuid, **no FK**: all six sales headers, `acc_tender_detail`, `acc_voucher_header`, `acc_vouchers`, `acc_bill_adjustment`, `acc_temp_credit`, `txn_hold`, `txn_status_log`, `loyalty_ledger`, `loyalty_coupon_txn`, `stock_voucher`.
- **`counter_id`** on `sale_bill`, `sale_dc`, `sale_dc_return`, `sale_return`, `acc_temp_credit`, `txn_hold` — **pointing at nothing today**; it will point at `till_counter`.
- **Today the client sends a random uuid per app run** as the session id. Existing rows therefore match no session (test data; harmless).
- `fixed.device_master` (17 rows) — device identity, blocked flag, branch.
- `public.txn_hold` (hold/park, already carries session, counter, device, lock).
- `acc_tender_detail` — tender rows with `td_session_id`, `td_device_id`, `td_user_id`, void columns (35), settlement columns (`td_settle_status`, `td_settled_on`, `td_mdr_amt` …).
- `acc_tender_master.tnd_open_cash_drawer`, `tnd_ledger_id`, `tnd_settlement_ledger_id`, branch scope.
- `user_master.usr_pin_hash` — the override PIN already has a home.
- `employee_master.emp_loan_ledger_id` — where a recovered shortage can go.
- `user_menus.um_can_override`, `um_can_retender`.
- `/bills/retender` + `TndC` (35) and the error `SALES_DAY_CLOSED` — designed to be refused after the drawer is counted. **This design is what finally makes that error mean something.**
- `public.print_log` — every print attempt, the anti-theft evidence for void.
- `app_setting_def/value` with GLOBAL < COMPANY < BRANCH < DEVICE < USER precedence.

---

## 5 · Architecture

### 5.1 The objects

```
Company
 └─ Branch (store) ── till_business_day (one per trading date)
      ├─ till_safe (1..n)  ◄──────── drops / pickups / close hand-over ─────┐
      │      └── remittance → Cash in transit → Bank                          │
      └─ till_counter (lane, drawer) ── device (fixed.device_master)          │
             └─ till_session (cashier × counter × time) ───────────────────────┘
                   ├─ sales / returns / receipts / advances / re-tenders   (existing docs, session_id)
                   ├─ till_cash_movement (float, top-up, pickup, drop, paid-out/in, hand-over)
                   ├─ till_count (+ lines)  — open, recount, close, surprise, hand-over
                   ├─ till_variance — per tender, decided and posted
                   ├─ till_approval — every exception let through, by whom
                   └─ till_event — append-only journal (drawer, no-sale, X, suspend …)
HO (cloud) ── till_session_review (append-only, HO-owned)
```

Roles (`till_operator.top_role`): CASHIER · SUPERVISOR (floor) · CASH_OFFICE · STORE_MANAGER ·
AREA_MANAGER · HO_FINANCE · AUDITOR (read-only). The role is a *label and default*; what someone
may approve is `till_approval_authority`.

### 5.2 Session lifecycle

```
             open (float counted)
                    │
     ┌──────────► OPEN ◄───────────┐ resume (same cashier, or supervisor PIN)
     │              │  suspend / idle lock
     │              ▼
     │          SUSPENDED
     │              │
     │   end billing (holds released or refused)
     │              ▼
     │          COUNTING ── count attempt 1..N (blind) ── recount past N → RECOUNT approval
     │              │
     │      variance within tolerance? ── yes ──────────────┐
     │              │ no                                     │
     │              ▼                                        ▼
     │     PENDING_APPROVAL ── approve (treatment) ──────► CLOSED ──► Z printed, hand-over posted
     │              │ reject → back to COUNTING              │
     │                                                        │ SESSION_REOPEN (approval)
     └────────────────────────────────────────────────────────┘
   VOIDED  ← only from OPEN with nothing written in it (SESSION_VOID approval), float returned
```

- **Force close** (cashier absent / device dead): a supervisor drives COUNTING → CLOSED, counting
  the drawer themselves; `tss_force_closed = true`, reason, FORCE_CLOSE approval. The variance is
  still the cashier's — flagged so HR sees it was not their count.
- **Reopen**: only a CLOSED session of an OPEN day; the hand-over and variance vouchers are
  **reversed** (new reversal rows, never deleted), the session returns to OPEN, and it must be
  counted again. `tss_reopen_count` goes up; the Z number is not reused.
- **Suspend**: break / meal / helping the floor, or idle lock (`till.idle_lock_minutes`).
  Nothing can be billed. Resuming is the same cashier's PIN or a supervisor's. A suspend over the
  rule's minutes raises SUSPEND_LONG (shipped NEVER — logged only).
- **One live session per counter** (unique index, safe offline because a counter belongs to one
  site). **One live session per operator** is a service rule + reconciliation query, NOT an index
  (§5.9).
- **TRAY counters** (`tcn_drawer_mode = TRAY`): the cash insert travels with the cashier. Moving
  counter = suspend on A, resume on B; the service re-points `tss_counter_id` only when B has no
  live session, and logs it. **NONE** (cashless lane): a session is still opened (accountability
  for card/UPI and voids), nothing is counted by denomination.

### 5.3 Day lifecycle

```
OPEN  (first session opens it if till.day_auto_open, else a manager's Day Open)
  │   — all sessions of the day hang off it
  ▼
CLOSING  (manager starts Day Close; no NEW session may open)
  │   checklist must be green:
  │     ✔ every session CLOSED or VOIDED
  │     ✔ no PENDING approval in the day
  │     ✔ no till_variance PENDING
  │     ✔ no POSTED movement without its voucher (ix_tcm_unposted)
  │     ✔ held bills: none, or released to the pool
  │     ✔ safe counted (till.day_close_safe_count) — SAFE_VARIANCE approval if off
  │     ✔ remittance raised if the safe is over tsf_insured_limit (warn)
  │   anything amber → DAY_CLOSE_EXCEPTION approval (store manager)
  ▼
CLOSED — totals frozen, store Z number issued, Z printed.
         From here SALES_DAY_CLOSED refuses: re-tender, cancel-with-cash, back-dated money docs.
DAY_REOPEN — area manager, REMOTE; frozen totals are recomputed at the next close.
```

**Cut-off:** `till.day_cutoff` (default 04:00). A sale at 00:40 on the 8th belongs to business date
the 7th. Every session carries `tss_business_date`; every till document carries the business date
as its doc date.

### 5.4 Expected cash — the one formula (server only)

`TillLedgerService.expected(sessionId, tenderTypeId, tenderId?)`, from facts already in the database:

| term | source | sign |
|---|---|---|
| **open** | `tss_float_counted` (cash only) — the *counted* float, so a float mismatch is settled at open and not carried into the close | + |
| **sales** | `acc_tender_detail` where `td_session_id = S`, type = T, `td_dr_cr = 'D'`, not voided, not deleted, source in (SALE_BILL, ORDER_ADVANCE, RE-TENDER new rows) | + |
| **refunds** | same, `td_dr_cr = 'C'` (sale-return cash settle, advance refund, re-tender voided side) | − |
| **receipts** | receipts / receipt vouchers posted **at a counter** with session S and tender T (incl. temp-credit collections, cheque deposit at desk) | + |
| **moved in** | `till_cash_movement` TOP_UP + PAID_IN, POSTED, session S (FLOAT_ISSUE is *not* added — the counted float above already is it) | + |
| **moved out** | PICKUP + DROP + PAID_OUT, POSTED, session S (CLOSE_HANDOVER is *not* subtracted — it happens after the count) | − |
| **expected** | open + sales − refunds + receipts + in − out | |

- The same split is written to `till_variance` (`tvr_open_amount … tvr_moved_out`) and a CHECK
  makes the row add up (`ck_tvr_expected` — one row's own arithmetic, allowed under the offline rule).
- **Change given** is already netted: `td_amount` is what stayed, `td_received_amt` / `td_change_amt`
  are kept for the receipt print.
- Non-cash tenders use the same formula with open = 0 and no movements.
- STATEMENT tenders (UPI, wallet, bank) are shown with expected and **no count**; they are
  reconciled later against the provider statement through the existing `td_settle_*` columns.
  NONE tenders (credit, temp credit, loyalty, RRN) are shown for information only.
- **Blind mode:** `GET /till/sessions/expected` refuses a caller without SUPERVISOR authority on
  this branch (`TILL_BLIND_CLOSE`). The count response to a blind cashier says only
  `ACCEPTED` / `RECOUNT_REQUIRED` / `SENT_FOR_APPROVAL` — never the figure.

### 5.5 Money moves where the money moves (D7)

| Situation | Booked in |
|---|---|
| Refund (sale return CASH settle) of a bill from a closed session | the **current** session of the refunding cashier |
| Re-tender of a bill from a closed session (UPI failed, customer came back with cash) | the current session; the original session is **not** touched |
| Cancel of a POSTED bill whose session is **CLOSED** and which took **cash** | **refused** — `TILL_SESSION_CLOSED_USE_RETURN`: do a sale return (cash leaves *today's* drawer). Credit-only / non-cash bills may still be cancelled. |
| Temp-credit / customer receipt at a counter | the current session (it is cash in this drawer) |
| Receipt entered in the back office (bank, cheque by post) | no session (`till.require_session` is false on back-office devices) |
| Back-dated money document into a CLOSED day | refused — `SALES_DAY_CLOSED` |

The sales document keeps its own `*_session_id` (where it was raised); the tender rows and
voucher of the money movement carry the session where the money moved.

### 5.6 Approvals

**Three tables:** the RULE (when is approval needed), the AUTHORITY (who may approve what, up to
how much), the APPROVAL (one request, decided once).

```
event occurs (e.g. paid-out 1,800)
   │
   ├─ ruleFor(company, branch, PAID_OUT, businessDate)   branch row > company row > shipped row, latest effective
   │     mode OVER_AMOUNT 500 → needed
   │
   ├─ requiredLevel = max(rule.tar_min_role, the lowest role whose authority covers 1,800 at this branch)
   │
   ├─ channel COUNTER → approver types PIN (or swipes badge) on the cashier's screen
   │     store server: verify usr_pin_hash · authority covers event/branch/amount/date ·
   │                   decided_by ≠ requested_by (unless tar_allow_self) · two-person if asked
   │     → till_approval APPROVED + the movement posts, in ONE transaction
   │
   └─ channel REMOTE → till_approval PENDING; the counter shows "waiting for approval" (tar_blocks_till)
         approver with taa_can_remote decides from the Approval Inbox / mobile,
         THROUGH THE STORE SERVER'S API (the decision row is store-written)
         expiry (tar_expire_minutes) → EXPIRED, the cashier re-requests or cancels
```

- **Escalation by amount, not by event.** A CASH_VARIANCE of 40 is a supervisor's; of 4,000 needs a
  store manager whose `taa_max_amount` covers it; of 40,000 needs HO_FINANCE remotely. Same rule
  row; the authority grid decides the level.
- **Two-person** (`tar_two_person`): two distinct approvers, neither the requester (CHECKs
  `ck_tap_second`, `ck_tap_two_person`). Shipped on SAFE_VARIANCE.
- **No self-approval** (`ck_tap_self`) unless the rule snapshot on the row says allowed.
- **PIN failures** log `PIN_FAIL` events; 5 in 10 minutes locks that approver's PIN
  (`usr_failed_login_count` pattern) — the service, not the DB.
- **Offline:** the COUNTER channel needs only the store server (user_master and the authority grid
  are local). The REMOTE channel needs the approver to reach the store server; if the store is
  cut off from the internet, only COUNTER works and the counter says so.
- **Reserved events** (`VOID_BILL`, `VOID_LINE`, `PRICE_OVERRIDE`, `DISCOUNT_OVER`,
  `RETURN_NO_RECEIPT`, `REFUND_CASH`, `REPRINT`, `RETENDER`) are already legal in the CHECK so the
  parked void module (SIX_TXN_CONTROLS §7) writes to the same table and the same gate component.
  No rule rows are shipped for them yet.
- **Not a voucher workflow.** DRAFT → POSTED → CANCELLED is unchanged. A paid-out needing approval
  is simply not posted until the approval exists; there is no "APPROVED" voucher state.

Shipped rules (company/branch NULL — override per company or branch):

| event | mode | threshold | channel | min level | notes |
|---|---|---|---|---|---|
| FLOAT_MISMATCH | OVER_AMOUNT | 0 | COUNTER | SUPERVISOR | any opening difference |
| CASH_VARIANCE | OVER_AMOUNT | 10 | EITHER | SUPERVISOR | above tolerance; level rises with amount |
| NONCASH_VARIANCE | OVER_AMOUNT | 0 | EITHER | STORE_MANAGER | re-tender first |
| PAID_OUT | OVER_AMOUNT | 500 | COUNTER | SUPERVISOR | |
| PAID_IN | NEVER | | | | logged |
| PICKUP / TOP_UP | ALWAYS | | COUNTER | SUPERVISOR | the supervisor doing it is the approver; cashier witnesses |
| NO_SALE | OVER_COUNT | 3 / session | COUNTER | SUPERVISOR | |
| CASH_LIMIT_OVERRIDE | ALWAYS | | COUNTER | SUPERVISOR | billing past the hard limit |
| FORCE_CLOSE · SESSION_REOPEN | ALWAYS | | EITHER | STORE_MANAGER | |
| SESSION_VOID · MOVEMENT_VOID | ALWAYS | | COUNTER | SUPERVISOR | |
| RECOUNT | OVER_COUNT | 1 | COUNTER | SUPERVISOR | past `till.max_recounts` |
| DAY_CLOSE_EXCEPTION | ALWAYS | | EITHER | STORE_MANAGER | |
| DAY_REOPEN | ALWAYS | | REMOTE | AREA_MANAGER | |
| SAFE_VARIANCE | OVER_AMOUNT | 0 | EITHER | STORE_MANAGER | two-person |
| REMITTANCE | NEVER | | | CASH_OFFICE | |

### 5.7 Accounting — every movement, every voucher

Till cash = the branch's CASH tender ledger. Safe = `till_safe.tsf_ledger_id` (role SAFE_CASH).
Every voucher carries `avh_session_id` (and the session's business date as date).

| Event | Voucher | Dr | Cr |
|---|---|---|---|
| Float issue / top-up | **TFlt** (contra) | Till cash | Safe |
| Pickup / drop / close hand-over / float return | **TDrp** (contra) | Safe | Till cash |
| Paid-out (petty expense, supplier COD, staff advance) | **TPOut** (payment) | expense / supplier / staff ledger | Till cash |
| Paid-in (scrap sale, staff advance returned) | **TPIn** (receipt) | Till cash | income / staff ledger |
| Float mismatch at open (counted ≠ issued/carried), once approved | **TVar** | Cash short & excess (short) — or the reverse for excess; treatment as any variance | Till cash |
| Variance — shortage, EXPENSE | **TVar** (journal) | Cash short & excess | Till cash |
| Variance — excess | TVar | Till cash | Cash short & excess |
| Variance — RECOVER | TVar | cashier's recovery ledger (`top_recovery_ledger_id`, else `emp_loan_ledger_id`) | Till cash |
| Variance — SUSPENSE | TVar | Tender suspense | Till cash (cleared later by an HO journal) |
| Variance — RETENDERED | none | — | the re-tender's TndC already moved it |
| Remittance to bank via CIT | **CRem** (contra) | Cash in transit | Safe |
| Bank credit confirmed | existing **Con** contra, at HO | Bank | Cash in transit |
| Safe ↔ safe / store → HO cash | CRem | other safe / HO cash | this safe |

**Invariant the design delivers:** after a session closes, the till cash legs *of that session*
(sum of legs with `avh_session_id = S` on the till cash ledger) equal
`tss_float_left − (tss_float_mode = 'CARRIED' ? tss_float_issued : 0)` — the carried-in float was
booked under the previous session, which is why that one closes at its own `tss_float_left`. This
needs the float mismatch posted at open (row above); otherwise the session is off by
`tss_float_variance`. HO runs it as a reconciliation query; a non-zero answer is a posting bug, not
a cash problem.

**Paid-outs with a GST invoice:** a paid-out records cash leaving the drawer and the expense ledger
only. If input credit is to be claimed, the bill is booked as a purchase/expense voucher against the
supplier in the back office and the paid-out pays *the supplier ledger* — the till never builds a
GST document.

**Non-cash at close:** card slips are counted (SLIPS) for the EDC batch; the money reaches the bank
through the existing settlement path (`td_settle_*`, `tnd_settlement_ledger_id`). The till does not
post card settlement.

### 5.8 Anti-theft controls this enables

| Fraud / error pattern | Control |
|---|---|
| Cashier skims exactly the excess | blind close; expected never shown to the cashier |
| Recount until it "matches" | every attempt kept; limit; approval past it; attempts visible to HO |
| Approver rubber-stamps | no self-approval; amount ceilings; two-person on the safe; approver history report |
| Drawer opened to take cash between sales | NO_SALE logged with reason; approval after N; DRAWER_LEFT_OPEN events from the drawer sensor where present |
| "Printer failed" then bill cancelled | session + `print_log` (void module will use the same approval table) |
| Payout to self | reason + ref mandatory; approval over limit; payouts per cashier report |
| Excess cash left in drawer for later | cash alert / block limits force pickups; surprise counts (`tct_kind = SURPRISE`) |
| Float shortage passed to next cashier | float counted at open, mismatch approved and owned at that moment |
| Session left open, others bill on it | idle lock, resume by PIN, session max hours flag, force close |
| Short cash "fixed" by editing a closed session | closed rows are not edited; reopen = approval + reversal + recount |
| Store edits what HO reviewed | HO review lives in HO-written rows only |

### 5.9 Offline and sync

Sync is **push-only, upsert by PK** (`cloud-sync/README.md`): the store's row overwrites the cloud's.

| Table | Written by | Why that is safe |
|---|---|---|
| masters (`till_counter`, `till_safe`, `till_operator`, `till_reason`, `till_denomination`, `till_approval_rule`, `till_approval_authority`) | **wherever they are authored — see Q1** | |
| `till_business_day`, `till_session`, `till_count(_line)`, `till_cash_movement`, `till_variance`, `till_approval`, `till_event` | the **store server** owning the branch's counters | one site per counter → one writer per row |
| `till_session_review` | the **cloud** only | no store ever writes it, so no push overwrites it |

- **No cross-site uniqueness.** "One live session per operator" would be a contended resource
  (an operator signed in at two offline sites) → service rule + reconciliation query
  (`SELECT tss_operator_id … HAVING count(*) > 1` over live sessions), never an index.
- **GENERATED columns** (`tss_*_variance`, `tbd_cash_variance`, `tvr_variance`, `tcl_amount`)
  must be omitted from a push column list (Prisma already does).
- **`till_event` is idempotent under re-send:** `ux_tev_client_seq` on (device, client seq).
- **A till device that loses the store server** keeps billing only if the product's offline-till
  mode allows it; the session id it already holds stays valid, and the movements it queues are
  sent with their device timestamps. Opening or closing a session needs the store server.

---

## 6 · Schema — `47_till_management.sql`

Validated 2026-10-07 on a throwaway PG 18 cluster loaded with the **live DDL** of every table it
touches: applies clean, re-runs clean (idempotent), and 13 behaviour probes pass (§15).

| Table | Kind | Purpose |
|---|---|---|
| `till_safe` | master | store safe / cash office, its SAFE_CASH ledger, insured limit |
| `till_counter` | master | lane: kind, drawer mode, bound device, default float, alert/block limits, Z counter |
| `till_operator` | master | cashier/approver profile on a user: role, badge hash, max float, recovery ledger |
| `till_reason` | master | controlled reasons by category (17 categories, 39 shipped rows) |
| `till_denomination` | master | INR notes & coins (11 shipped), bundle size, withdrawn-after date |
| `till_approval_rule` | master | event × mode × threshold × channel × level × two-person × self × blocks (19 shipped) |
| `till_approval_authority` | master | user × company × branch × level × event × max amount × remote |
| `acc_tender_types` + 2 cols | ALTER | `ttm_close_mode` DENOM/SLIPS/STATEMENT/NONE, `ttm_handover` |
| `till_business_day` | txn | the day: status, Z, frozen totals, exception count |
| `till_session` | txn | the shift: open/float, suspend, close/count, frozen totals, 3 generated variances |
| `till_count` / `till_count_line` | txn | every count event, every attempt; DENOM / SLIPS lines, generated amount |
| `till_cash_movement` | txn | 10 kinds; per-kind shape CHECKs; voucher link; void with reason |
| `till_variance` | txn | per session per tender, stage OPEN (float) or CLOSE: the expectation split, treatment, posting |
| `till_approval` | txn | request → decision; no self, two-person, rule snapshot |
| `till_event` | txn | append-only journal, 40 event codes, idempotent client seq |
| `till_session_review` | txn | HO-owned review outcomes |

Seeds: 4 ledger roles (`SAFE_CASH`, `CASH_IN_TRANSIT`, `CASH_SHORT_EXCESS`, `TENDER_SUSPENSE`;
**no TILL_CASH role** — D9), 6 voucher types (`TFlt`, `TDrp`, `TPOut`, `TPIn`, `TVar`, `CRem`) and
their number series cloned from the `Bil` rows, 17 settings, reasons, denominations, rules,
current-year partitions.

**Not changed:** the `*_session_id` / `*_counter_id` columns on existing tables stay FK-less (other
partitions and years, offline push ordering). `TillSessionService.assertLive()` validates the id at
post time instead.

**The map rows** (`acc_ledger_map`) for the four new roles are per company and are set on the
Ledger Map screen (menu 250); the service refuses a till posting with `TILL_LEDGER_UNMAPPED` naming
the role.

---

## 7 · Server (NestJS) — services, routes, integration

### 7.1 Services (each table has ONE writing service; a test asserts no other file writes it)

| Service | Owns |
|---|---|
| `TillDayService` | open / close-check / close / reopen; frozen totals; store Z |
| `TillSessionService` | open, current, suspend/resume, end billing, count, close, force close, reopen, void, `assertLive()` |
| `TillLedgerService` | `expected()` (§5.4), session totals, the till-ledger invariant check |
| `TillMovementService` | the 10 movement kinds, numbering, posting through the existing voucher posting service, void = reversal voucher |
| `TillVarianceService` | per-tender variance, treatment, TVar posting, reversal |
| `TillApprovalService` | `ruleFor()`, `requiredLevel()`, PIN/badge verify, decide, expire sweep |
| `TillEventService` | the only writer of `till_event`; batch ingest |
| `TillSafeService` | safe balance (derived), safe count, remittance |
| `TillReviewService` | cloud only — `till_session_review` |

### 7.2 Routes — new family, nothing existing is called

All take `companyId · branchId · accYear` and, on an existing record, its id. Lists are grids.

| Route | Does |
|---|---|
| `POST /till/days/open` · `GET /till/days/get` · `GET /till/days/close-check` · `POST /till/days/close` · `POST /till/days/reopen` | the day |
| `POST /till/sessions/open` | counter (from device), float mode, issued, count lines → session + TFlt + FLOAT_MISMATCH approval if needed |
| `GET /till/sessions/current?deviceId=&userId=` | the live session for this device/user (resume at login) |
| `GET /till/sessions/get` | header + per-tender summary (expected hidden in blind mode for cashiers) |
| `GET /till/sessions/expected` | supervisor only; refused for the cashier in blind mode |
| `POST /till/sessions/suspend` · `/resume` | break, idle lock, PIN resume |
| `POST /till/sessions/end-billing` | stops money; releases or refuses holds (`till.close_with_holds`) |
| `POST /till/sessions/count` | one count attempt (lines); answers ACCEPTED / RECOUNT_REQUIRED / SENT_FOR_APPROVAL |
| `POST /till/sessions/close` | variances + treatments, hand-over (TDrp), TVar, Z number, CLOSED |
| `POST /till/sessions/force-close` · `/reopen` · `/void` | with approval id |
| `POST /till/sessions/x-report` | logs X_REPORT, returns the read |
| `POST /till/movements/create` · `GET /till/movements/get` · `POST /till/movements/void` | the 10 kinds |
| `POST /till/approvals/request` · `POST /till/approvals/decide` · `GET /till/approvals/get` · `POST /till/approvals/cancel` | the gate (decide = PIN or remote) |
| `POST /till/events/batch` | client events (drawer, no-sale, idle lock, reprint) |
| `POST /till/safe/count` · `GET /till/safe/balance` | cash office |
| `POST /till/reviews/create` | cloud only |
| masters | registered CRUD (ui_tables) for counter, safe, operator, reason, denomination, rule, authority |

### 7.3 Error codes

| code | status | when |
|---|---|---|
| `TILL_SESSION_REQUIRED` | 409 | money-moving post with no session and `till.require_session` |
| `TILL_SESSION_NOT_OPEN` | 409 | session SUSPENDED / COUNTING / CLOSED |
| `TILL_SESSION_NOT_YOURS` | 403 | session's operator ≠ caller (supervisor actions excepted) |
| `TILL_SESSION_WRONG_DEVICE` | 409 | device not bound to the session's counter |
| `TILL_COUNTER_BUSY` | 409 | counter already has a live session |
| `TILL_OPERATOR_BUSY` | 409 | operator has a live session elsewhere (online check) |
| `TILL_DAY_NOT_OPEN` / `TILL_DAY_CLOSING` | 409 | no day / day closing: no new session |
| `SALES_DAY_CLOSED` | 409 | (existing) money doc into a closed day |
| `TILL_CASH_BLOCKED` | 409 | drawer over `tcn_cash_block_limit` — pickup or CASH_LIMIT_OVERRIDE |
| `TILL_APPROVAL_REQUIRED` | 428 | carries the event, amount, required level, channel, request id |
| `TILL_APPROVAL_DENIED` | 403 | PIN wrong / authority does not cover / self-approval |
| `TILL_BLIND_CLOSE` | 403 | cashier asked for expected in blind mode |
| `TILL_HOLDS_OPEN` | 409 | end billing with holds and BLOCK |
| `TILL_RECOUNT_LIMIT` | 428 | past `till.max_recounts` |
| `TILL_SESSION_CLOSED_USE_RETURN` | 409 | cancel of a cash bill whose session closed |
| `TILL_LEDGER_UNMAPPED` | 422 | a till role has no ledger in `acc_ledger_map` |
| `TILL_DAY_CLOSE_BLOCKED` | 409 | close-check not green and no exception approval |

### 7.4 Changes to existing modules (each a small, listed change)

| Module | Change |
|---|---|
| bill post / amend | `assertLive(sbSessionId, user, device)` when any tender moves money; stamp `sbCounterId` from the session; tender rows + voucher carry the session |
| bill cancel | refuse `TILL_SESSION_CLOSED_USE_RETURN` when the original session is CLOSED and a cash tender exists |
| sale return CASH settle | refund tender rows carry the **current** session |
| order advance (ARc), receipts at a counter, temp-credit collection | same `assertLive`, current session |
| `/bills/retender` | the new tender rows and TndC carry the **current** session; `SALES_DAY_CLOSED` now reads `till_business_day` |
| `txn_hold` | end-billing RELEASE clears `txh_session_id` / `txh_counter_id` (pool); resume stamps the resuming session |
| cash-drawer kick (`tnd_open_cash_drawer`) | the client logs DRAWER_OPEN_SALE with the bill id |
| cloud sync | nothing — new tables have `*_sync_date` and a PK |

---

## 8 · Screens (Qt) — each its own `.ui` on `NexDialog`

Layouts below are the content and order, not the look; the template supplies styling. Every popup
installs `EnterNavigation`. Keys follow the sale bill's conventions.

### S1 · Open Session (popup — at login on a till device, or when billing opens with no session)

```
┌ Open Till Session ──────────────────────────────────────────────┐
│ Counter  C01 · Counter 1 (this device)     Business date 07-10-26 │
│ Cashier  RAVI K                            Session   C01-261007-02 │
│ Float    (•) Issued from safe   ( ) Carried from C01-261007-01     │
│          Issued / carried: 2,000.00                                │
│ ┌ Count ─────────────────────────────────────────────────────────┐ │
│ │ ₹500 [  2] 1,000 │ ₹100 [  5]   500 │ ₹20 coin [  ]          │ │
│ │ ₹200 [  1]   200 │ ₹50  [  4]   200 │ ₹10 coin [ 10]  100    │ │
│ └──────────────────────────────────────────── Counted 2,000.00 ┘ │
│ (difference → "Supervisor PIN required", reason combo)            │
│                                   [F10 Open]  [Esc Not now]        │
└─────────────────────────────────────────────────────────────────┘
```
If a live session already exists for this device/user: "Resume session C01-261007-02 opened 09:02?"
(Resume / Supervisor force-close). "Not now" on a device with `till.require_session` leaves billing
disabled, with the reason shown.

### S2 · Till strip on the sale bill (and return, receipt)

A one-line strip under the verb bar: `C01-261007-02 · RAVI · Cash in drawer ●●●○` (a gauge against
the alert limit, never the figure in blind mode) · `F11 Till menu`. When over the alert limit the
gauge turns amber ("Pickup due"); over the block limit billing stops with `TILL_CASH_BLOCKED`.

**F11 Till menu:** Paid out · Paid in · Drop · No sale · X read · Suspend · End shift ·
Pickup (supervisor) · Top-up (supervisor).

### S3 · Cash Movement (one form, mode chosen from the menu)

```
┌ Paid Out ─────────────────────────────────────────┐
│ Reason   [Petty expense       ▼]   Ref no [_____]  │
│ Ledger   [Tea & refreshments  ▼]   Paid to [_____] │
│ Amount   [      250.00]                            │
│ Notes    [________________________________]        │
│ (over limit → approval strip: PIN / Send remote)   │
│                         [F10 Post & print] [Esc]   │
└───────────────────────────────────────────────────┘
```
Drop / Pickup / Top-up show the denomination grid instead of Ledger, plus Bag no / Seal no, and a
Witness PIN line. Prints the slip (two signature lines for pickup).

### S4 · End Shift (blind close wizard, the cashier's screen)

```
Step 1  End billing      held bills: 2 → [Release to pool]  (or refused if BLOCK)
Step 2  Count cash       denomination grid, running total — NO expected shown
Step 3  Count slips      Card: slips [ 14] amount [12,340.00] batch [0031]
                         Cheque: [1] [5,000.00]   Gift voucher: [0]
                         UPI / Wallet / Bank: "verified later" (no input)
Step 4  Submit           → "Count accepted. Hand the drawer to the cash office."
                         or "Please recount (1 recount left)."
                         or "Sent to supervisor for approval."
```
In CASH_OFFICE mode step 2–4 run on the cash-office device; the counter shows "Drawer sealed —
awaiting count".

### S5 · Session Cockpit (supervisor / cash office) — the heart of the module

```
┌ Session C01-261007-02 · RAVI K · Counter 1 · 09:02–17:41 · PENDING APPROVAL ───────────────┐
│ Tender        Open   Sales  Refunds Receipts  In    Out    Expected Counted Variance Reason  Treatment│
│ Cash        2,000 41,250    -470     1,200    0  -30,000   13,980  13,900    -80 [Count ▼][Expense ▼]│
│ Card            0 12,340       0         0    0       0    12,340  12,340      0                     │
│ UPI             0 18,900       0         0    0       0    18,900   (later)                          │
│ Cheque          0  5,000       0         0    0       0     5,000   5,000      0                     │
│ Credit          0  3,200       — information only —                                                 │
│ Total                                                                    -80                        │
├ tabs: Bills │ Returns │ Receipts │ Movements │ Counts (3 attempts) │ Approvals │ Timeline ───────────┤
│ Counts:  #1 13,850 (blind)  #2 13,900 (blind, final)   expected at count 13,980                  │
│ Timeline: 09:02 open · 11:15 pickup 15,000 (SUP ANIL) · 13:40 no-sale "change" · 15:20 pickup …   │
├────────────────────────────────────────────────────────────────────────────────────────────────┤
│ [F10 Approve & close] [Recount] [Re-tender a bill] [Force close] [Reopen] [Print Z] [X read]     │
└────────────────────────────────────────────────────────────────────────────────────────────────┘
```
- Approve & close needs the approver's PIN at the level the variance requires (§5.6).
- "Re-tender a bill" opens the existing Ctrl+F6 re-tender form filtered to this session — the
  intended fix for a non-cash gap, before approving anything.
- Double-click a bill/return/receipt opens the document read-only.

### S6 · Sessions list (TxnMainView, grid)

Filters: branch, business date range, counter, cashier, status, "with variance", "force closed".
Columns: session no, date, counter, cashier, opened, closed, duration, bills, net sales, cash expected,
counted, variance (chip red/green), approvals, status chip. Actions: Open cockpit, Print Z, History
(Ctrl+H → `till_event` timeline).

### S7 · Approval gate (reusable component) + Approval Inbox

- **Gate popup:** event, amount, requester, rule ("Paid out over 500 needs Supervisor"), approver
  login + PIN (or badge scan), decision note; "Send to remote approver" when the channel allows.
- **Inbox** (managers, grid over `ix_tap_pending`): branch, counter, event, amount, waiting since,
  requester; Approve / Reject with note. Polls; shows the expiry countdown.

### S8 · Day Open / Day Close (store manager)

A checklist screen: each row green/amber/red with a jump button — sessions not closed (open cockpit),
pending approvals (inbox), pending variances, unposted movements, held bills, safe count (S9),
remittance due. "Close day" when green, or with a DAY_CLOSE_EXCEPTION approval. Prints the store Z.

### S9 · Cash Office — Safe book & remittance

Safe running balance for the day (opening, + drops/pickups/hand-overs, − floats/top-ups, −
remittances), drop bags awaiting verification (count each bag → DROP_VERIFY; a bag difference becomes
the dropping session's variance via a reopened variance row, never an edit), safe count, Remit to bank
(bank, CIT agency, bag, seal, slip no).

### S10 · Masters (registered CRUD forms)

Counter · Safe · Till Operator · Approval Authority (grid per user: branch, event, level, max, remote)
· Approval Rules (per event, override at company/branch) · Reasons · Denominations.

### S11 · HO Cash Position (chain dashboard, cloud)

Per store, live as of last sync: day status, open sessions, cash in tills (expected), safe balance,
last remittance and age, variance today / MTD, pending approvals, last sync time. Drill to the
session cockpit read-only, and HO review actions (§9 table).

---

## 9 · Printing (data-driven templates, `print_log` on every attempt)

| Doc type | When | Content |
|---|---|---|
| `TILL_SESSION_Z` | session close (and reprint, logged) | Z no, counter, cashier, times, per-tender expectation split, counted, variance, counts summary, movements, approvals, signatures (cashier, approver) |
| `TILL_X_READ` | any time in session | the read **without** expected cash in blind mode for the cashier |
| `TILL_MOVEMENT` | each movement | slip with doc no, kind, amount in words, denominations, bag/seal, signatures |
| `TILL_DAY_Z` | day close | store Z: sessions, tenders, variances, safe, remittance, exceptions |
| `TILL_REMIT` | remittance | deposit memo for the CIT/bank, bag + seal |

---

## 10 · Reports and grids

| Report | Answers |
|---|---|
| Session register | every session, filters as S6 |
| Cashier variance history | per operator, period: sessions, short count/total, excess count/total, force-closed, recounts used |
| Exceptions | no-sales, paid-outs, pickups, force closes, reopens, recounts, PIN failures — per cashier, per hour |
| Approver activity | per approver: approvals, amounts, rejections, self-approval attempts refused |
| Safe book | running safe balance, drops, bag differences, remittances, insured-limit breaches |
| Remittance register | raised vs bank-credited (via HO contra), age of cash in transit |
| Day summary (store Z) | per day, per branch |
| Tender reconciliation | card/UPI expected per session vs settled (`td_settle_*`) |
| Cashier performance | bills/hour, items/bill, avg scan-to-tender time (from bill timestamps), voids (later) |
| Chain cash position | S11 |
| Till ledger invariant | sessions where till-cash legs ≠ `tss_float_left` (posting bugs) |

---

## 11 · Settings (17, seeded in 47 §10.3)

`till.require_session` (DEVICE) · `till.day_cutoff` · `till.day_auto_open` · `till.float_mode`
(ISSUED/CARRIED) · `till.handover_mode` (FULL_CLOSE/HANDOVER_COUNT) · `till.blind_close` ·
`till.max_recounts` · `till.count_place` (COUNTER/CASH_OFFICE) · `till.close_witness` ·
`till.cash_tolerance` · `till.noncash_tolerance` · `till.close_with_holds` (BLOCK/RELEASE) ·
`till.variance_default` (EXPENSE/RECOVER/SUSPENSE, COMPANY) · `till.idle_lock_minutes` (DEVICE) ·
`till.session_max_hours` · `till.day_close_safe_count` · `till.no_sale_reason`.
Approval thresholds are **rules**, not settings (they need levels and a channel).

---

## 12 · Client changes outside the new screens

- `AppSession::sessionId()` stops minting a random uuid. It returns the **live till session id**
  (set by S1 / resume) or null. On a device with `till.require_session`, the sales screens disable
  posting money with the reason shown when it is null.
- `AppSession` gains `counterId()` (from the session) and the sales screens send `sbCounterId` /
  `sdcCounterId` from it (the DC today sends the device id as the counter — fix).
- A small `TillGate` client component: wraps a call, catches `TILL_APPROVAL_REQUIRED`, runs S7,
  retries with the approval id. The void module reuses it.
- Drawer kick (when a CASH tender has `tnd_open_cash_drawer`) logs DRAWER_OPEN_SALE; a no-sale opens
  the drawer only through F11 → No sale.

---

## 13 · Build order

| Phase | Server | Client | Gate to next |
|---|---|---|---|
| **1 · Core session** | 47 applied; masters CRUD; day auto-open; open / current / suspend / resume / end-billing / count / close (no approvals yet: everything within tolerance passes, over it → PENDING); `assertLive` on bill/return/receipt/advance/retender; TFlt/TDrp/TVar posting | S1, S2 (strip only), S4, S6, `AppSession` change | a cashier can open, bill, close blind; ledgers reconcile (invariant query = 0) |
| **2 · Movements** | 10 movement kinds + voids; cash limits | S3, F11 menu, slips | payouts appear in the books |
| **3 · Approvals** | rules, authority, gate, PIN, remote, expiry | S7 gate + inbox, approval tab in S5 | every exception has an approver |
| **4 · Cockpit + day** | expected (supervisor), force close, reopen, day close-check/close/reopen, store Z | S5 complete, S8 | a store can close its day |
| **5 · Safe** | safe count, drop verification, remittance | S9 | cash office runs on it |
| **6 · HO** | review rows (cloud), chain queries | S11, reports §10 | HO sees every store |
| **then** | the parked **void bill / void line** (SIX_TXN_CONTROLS §7) — on this session and this approval gate | | |

---

## 14 · Decisions taken on the open questions (user, 2026-10-07)

1. **Masters are authored on the store server** — counters, safes, operators, reasons,
   denominations, approval rules and authority. They are per-branch anyway, and the COUNTER (PIN)
   approval must work with the cloud unreachable. Company-wide / shared rows (`company_id` or
   `branch_id` NULL) are seeded with the store; changing them chain-wide waits for a masters-down
   channel, which is out of scope here.
2. **Shortage recovery is booked to the loan ledger only** — `top_recovery_ledger_id`, else the
   employee's `emp_loan_ledger_id`. No payroll link. HR recovers from the ledger balance.
   `TILL_LEDGER_UNMAPPED` (naming the cashier) when neither is set and RECOVER is chosen.
3. **PNG mockups drawn** for S1 Open Session, S4 End Shift, S5 Session Cockpit and S8 Day Close —
   `till/till_open_session_ui_mockup.png`, `till_end_shift_ui_mockup.png`,
   `till_session_cockpit_ui_mockup.png`, `till_day_close_ui_mockup.png` (scripts beside them,
   `../sales/mock_lib.py`). They are the visual spec for those screens.

---

## 15 · Validation done (2026-10-07)

Throwaway PG 18.1 cluster, `erp_dry`, loaded with the **live DDL + data** of `acc_ledger_master`,
`acc_tender_types`, `acc_voucher_types` (+ enums), `acc_voucher_seq`, `acc_ledger_role`,
`fixed.device_master`, `app_setting_def`. Two seed bugs were caught and fixed this way
(`alr_group` NOT NULL with its CHECK; `asd_data_type` has no STRING — TEXT).

- 47 applies with `ON_ERROR_STOP`, and re-runs clean (seed counts unchanged on the second run):
  6 voucher types · 6 number series · 17 settings · 39 reasons · 11 denominations · 19 rules ·
  4 roles · 9 partitions · tender close modes set on all 11 types.
- Behaviour probes, in a rolled-back transaction — refused as designed: second live session on
  one counter · CARRIED float with no previous session · self-approval · two-person approval with
  one approver · a second FINAL close count · PAID_OUT with no ledger · negative movement · a
  variance whose expectation does not add up · an EXPENSE treatment with no reason · a CLOSED
  session whose hand-over + float left ≠ counted. Accepted as designed: open → pickup → blind count
  → variance −20 (generated) → close → a CARRIED next session on the same counter. Second pass after adding `tvr_stage`: an OPEN-stage
  float variance (−10) and a CLOSE-stage cash variance coexist; an OPEN-stage row carrying sales is
  refused; all earlier probes re-run unchanged.
- **Not done:** nothing was applied to 192.168.0.106; no NestJS or Qt code was written.
