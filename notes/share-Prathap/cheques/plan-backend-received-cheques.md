# Received cheques — backend plan (NestJS)

**For Prathap. Revision 1, 2026-09-14.** Written against the schema at
192.168.0.106/ERP and on top of `../receipt/plan-backend-receipt.md` (rev 3),
whose §2 this plan assumes has been run. The spec is
`received_cheques_flow.md` in this folder; the pictures are
`received_cheques_ui_mockup.png` and `received_cheques_tables.png`. Decisions
C1–C10 are all taken — nothing here is open, only things you may disagree
with.

That box is a TEST database: its schema is authoritative, its rows are not
(0 register rows today).

**The whole module in one line:** move a received cheque through
HELD → DEPOSITED → CLEARED | BOUNCED → re-presented | REPLACED, or HELD →
RETURNED | CANCELLED, writing one register update, one status-log row and —
when money moves — one voucher per step, never editing a row.

---

## 1. What exists, what does not

| | state |
|---|---|
| `acc_pdc_register` | deployed, partitioned, fully constrained (`ck_apd_*`), 0 rows |
| voucher types for clearing / bounce | **none** — `VoucherNature` has CONTRA and JOURNAL, so rows only |
| `BOUNCE_CHARGES_RECOVERED` role | **none** |
| `txn_status_log` | deployed (07) — the history source |
| routes | none under `/cheques` |
| client | none; menu 51 hidden |
| the receipt module | this plan's producer — it creates every row here |

---

## 2. Schema prerequisites — run before the module

### 2.1 Two voucher types and one role

`received_cheques_flow.md` §2 — the two INSERTs into `acc_voucher_types`
(`ChqClr` CONTRA, bank voucher; `ChqBnc` JOURNAL) and the role, plus one
`acc_ledger_map` row per company.

### 2.2 Settings (`public.app_setting_def`, module `ACCOUNTS`)

| key | type | default |
|---|---|---|
| `accounts.bounce_charge_to_party` | NUMBER | 0 — the amount pre-filled on the bounce panel |
| `accounts.bounce_reasons` | TEXT (JSON list) | `["Funds insufficient","Payment stopped by drawer","Signature differs","Account closed","Post-dated presented early","Stale","Other"]` |

`accounts.pdc_posting_mode` already exists (receipt plan §2.8) and is read
off the ROW here (`apd_posting_mode`), not the setting.

### 2.3 The register grid

A `grid_details` row for `TxnMainView` (id to be issued): the SELECT in
`received_cheques_flow.md` §4 with the due bucket computed, joined to
`acc_ledger_master` (party), `acc_tender_detail` (the instrument row → Cheques
In Hand ledger), and the three vouchers' refnos. Key columns hidden:
`apd_id, apd_acc_year, apd_company_id, apd_branch_id`. Filters as
`grid_param`: `istatus` (list), `ifrom` / `ito` on instrument date,
`ibank_ledger_id`, `iparty_id`, `search`.

### 2.4 The deposit-slip dataset

Printing schema (17): dataset `cheque_deposit_slip` keyed by
`(company_id, bank_ledger_id, deposit_date, slip_no)` returning the bank's
own account (`acc_ledger_bank_accounts`), then one line per cheque. One
generic layout.

### 2.5 Menu

51 "Received Cheques" visible.

---

## 3. Module layout

```
src/modules/accounts/cheques/
  cheques.module.ts
  cheques.controller.ts
  cheques.service.ts                // §4.1 list, get, history
  cheque-deposit.service.ts         // §4.2 — bulk, no voucher
  cheque-clear.service.ts           // §4.3 — one contra voucher (or the receipt itself under ON_CLEARING)
  cheque-bounce.service.ts          // §4.4 — THE transaction: voucher, reversals, cascade, charge bill
  cheque-reissue.service.ts         // §4.5 / §4.6 — re-present, replace (the shared re-issue voucher)
  cheque-return.service.ts          // §4.7
  dto/ …
```

Reuse from the receipt module, do not rewrite: the voucher-number allocator;
`allocation-engine.ts` (re-issue and ON_CLEARING clearing allocate with it);
the posting order that makes `ck_avh_balanced` real (header DRAFT → legs →
POSTED last); the role resolver.

---

## 4. The endpoints

### 4.1 `GET /api/v1/cheques/list` (the grid), `GET /api/v1/cheques/get`, `GET /api/v1/cheques/history`

`get`: `?apdId&apdAccYear` → the row, its party, the instrument row, the
receipt / PDC voucher, the clearing / bounce / re-issue vouchers, the
adjustment rows with `abj_cheque_id = this` (settled bills, with pending now),
the charge bill if any, the replacement chain. `history`: the
`txn_status_log` rows for the cheque, newest first.

### 4.2 `POST /api/v1/cheques/deposit`

```jsonc
{ "cheques": [ { "apdId": "…", "apdAccYear": "2026-2027" }, … ],
  "bankLedgerId": "…", "depositDate": "2026-09-15", "slipNo": "D-121", "remarks": null }
```

Every row HELD (or BOUNCED → this is a re-present, routed to §4.5); deposit
date ≥ each instrument date and in an open year; bank ledger is a BANK ledger
of the company. Writes `received_cheques_flow.md` §3.1 per row + one
`txn_status_log` row each. One transaction. Returns the rows and the slip
key for printing.

### 4.3 `POST /api/v1/cheques/clear`

```jsonc
{ "apdId": "…", "apdAccYear": "2026-2027", "clearDate": "2026-09-22", "bankDate": "2026-09-22", "remarks": null,
  "allocations": [ … ] }      // ON_CLEARING rows only — the receipt wrote none
```

Row DEPOSITED; clear date ≥ deposit date. **ON_RECEIPT:** the `ChqClr`
voucher of flow §3.2 — header DRAFT with `avh_src ('ACCOUNTS','PDC', apdId)`
(a second attempt dies on `ux_avh_src` — surface it as "already cleared on
<date>"), two legs, `av_recon_date` = `bankDate` on the bank leg, POSTED
last; register + log. **ON_CLEARING:** an `Rct` voucher DR bank / CR party,
allocations through the receipt's engine (auto FIFO if the array is empty),
`abj` rows with `abj_cheque_id`, `apd_voucher_id` = this voucher. One
transaction either way.

### 4.4 `POST /api/v1/cheques/bounce`

```jsonc
{ "apdId": "…", "apdAccYear": "2026-2027", "bounceDate": "2026-09-14",
  "reason": "Funds insufficient", "reasonText": null,
  "bankCharge": 250.00, "partyCharge": 300.00 }
```

Row DEPOSITED; bounce date ≥ deposit date. In this order, one transaction:

1. Lock the register row, the receipt's bills (every bill in `abj` rows with
   this `abj_cheque_id`) and the cheque's ADVANCE bill, `FOR UPDATE`.
2. Resolve roles: `BANK_CHARGES` (if `bankCharge > 0`),
   `BOUNCE_CHARGES_RECOVERED` (if `partyCharge > 0`), Cheques In Hand from
   the cheque's tender row; refuse by role name if unmapped.
3. Number and insert the `ChqBnc` header (DRAFT), `avh_against_voucher_id` =
   `apd_voucher_id`, `avh_src_*` NULL.
4. Legs per flow §3.3 (five under ON_RECEIPT; under ON_CLEARING only the
   charge legs: DR party `partyCharge` / CR recovered, DR bank charges / CR
   bank). Trigger derives totals.
5. Negative `abj` rows for every row with `abj_cheque_id = this` (skip rows
   that are themselves reversals) — the trigger reopens the bills.
6. **C4 cascade:** find the ADVANCE bill(s) whose `abl_voucher_id` is the
   cheque's voucher and whose remainder came from this cheque (`abj_tender_id`
   on the ADVANCE's origin = the cheque's tender row); for each
   `ADVANCE_ADJUST` pair consuming it, write the negative pair; then
   soft-delete the ADVANCE.
7. The JOURNAL DR bill for `partyCharge` (flow §3.3 step 3) — skipped when 0.
8. Register: BOUNCED + the bounce columns, both voucher ids = this voucher;
   `txn_status_log`.
9. Header → POSTED (last).

Returns the voucher, the bills reopened (with pending now), the cascade
performed, the charge bill.

### 4.5 `POST /api/v1/cheques/re-present`

`{ apdId, apdAccYear, bankLedgerId, depositDate, slipNo, allocations?: [...] }`
— row BOUNCED. ON_RECEIPT: the re-issue voucher (`Rct`, DR Cheques In Hand /
CR party, `avh_against_voucher_id` = the ChqBnc voucher), fresh `abj` rows via
the engine (auto FIFO if empty; the bounce-charge bill is in the party's open
items and may be allocated to), then the deposit writes with
`apd_present_count = 2`. ON_CLEARING: the deposit writes only. Log.

### 4.6 `POST /api/v1/cheques/replace`

```jsonc
{ "apdId": "…", "apdAccYear": "…",
  "newCheque": { "instrumentNo", "instrumentDate", "amount", "bankName", "bankBranch", "ifsc", "micr", "drawerName" },
  "allocations?": [ … ] }
```

Row BOUNCED or HELD. From HELD: first the return reversal of §4.7 (no
charges). Then: new register row (HELD, today, `apd_tender_id` NULL,
`apd_posting_mode` copied from the old row), old row → REPLACED with
`apd_replaced_by_id / _acc_year`; ON_RECEIPT: the re-issue voucher for the
NEW amount, dated the new instrument date when post-dated (own voucher, the
receipt's rule), `apd_voucher_id` on the new row, allocations via the engine
(post-dated rows `abj_is_post_dated = true`). Log on both rows.

### 4.7 `POST /api/v1/cheques/return`

`{ apdId, apdAccYear, action: "RETURNED" | "CANCELLED", reason, remarks }` —
row HELD only. ON_RECEIPT: reversal voucher (`Rct`, DR party / CR Cheques In
Hand, `avh_against_voucher_id` = the receipt / PDC voucher), negative `abj`
rows, the C4 cascade, ADVANCE soft-deleted. ON_CLEARING: register only.
Register + log.

### 4.8 `GET /api/v1/cheques/deposit-slip`

`?bankLedgerId&depositDate&slipNo` → the dataset rows for printing (or the
print pipeline's own endpoint once 17 lands).

---

## 5. Rules the service enforces beyond the CHECKs

`received_cheques_flow.md` §6, all eight. Two to keep in front of you:
**Cheques In Hand comes from the cheque's tender row** (`apd_tender_id →
td_tender_ledger_id`), never from today's master; and **ON_CLEARING is
honoured per row** (`apd_posting_mode`), not per setting.

---

## 6. Conventions

1. camelCase keys; `apdId + apdAccYear` on every row reference.
2. Every mutating endpoint: one transaction, row locks first, header POSTED last.
3. Refusals name the state ("221870 is DEPOSITED — bounce or clear it") and the role when a mapping is missing.
4. No cache on the list; the due buckets change at midnight.

---

## 7. Test checklist

- Deposit of two HELD cheques writes both rows + two log rows and no voucher; a third that is CLEARED is refused by name and nothing is written.
- Deposit dated before a cheque's instrument date is refused ("deposit on or after 20-09").
- Clear (ON_RECEIPT): one `ChqClr` voucher, two legs, totals derived, `av_recon_date` = bank date, `apd_clear_voucher_id` set, status CLEARED; a second clear → "already cleared on …" via `ux_avh_src`; Cheques In Hand balance drops by the amount, bank rises.
- Clear (ON_CLEARING): an `Rct` voucher DR bank / CR party, `abj` rows with the cheque id, bills settle now.
- Bounce on 221870: five legs 19,050 / 19,050; bil00022 and bil00025 back to 12,500 / 6,000; a JOURNAL bill of 300 open for Anand Stores; `apd_bounce_voucher_id = apd_charge_voucher_id`; status BOUNCED with reason.
- Bounce of a cheque whose ADVANCE was applied to an invoice: the application is reversed (invoice reopens), the ADVANCE is soft-deleted, the cheque's own rows reversed — all or nothing.
- Bounce with `partyCharge` and no `BOUNCE_CHARGES_RECOVERED` mapping → refused by role name, nothing written.
- Re-present: `Rct` re-issue voucher, fresh allocations (the bounce-charge bill allocatable), status DEPOSITED, `apd_present_count` 2; the bounce voucher untouched.
- Replace from BOUNCED: new HELD row linked from the old REPLACED row; the new amount 12,300 re-issued and allocated; a post-dated replacement gets its own voucher dated the cheque.
- Return from HELD: reversal voucher, bills reopen, status RETURNED with reason; from DEPOSITED → refused.
- Every transition appears in `history` with who and when.
- The list's due bucket flips FUTURE → DUE_TODAY → OVERDUE → STALE with the date, and the summary strip's IN HAND + WITH THE BANK equals the Cheques In Hand ledger.

---

## 8. One phase

Everything above is built together (list, get, history, deposit, slip,
clear, bounce, re-present, replace, return). Not in this plan, by decision:
bank reconciliation (out), Issued Cheques (menu 52 — own plan), ON_CLEARING
for the *receipt* side (already in the receipt plan as "refused"; here it is
honoured per row so a later switch needs nothing from this module).

## 9. Decisions

C1–C10 in `received_cheques_flow.md` §8 — all taken.

## 10. Do not build

- **No edit of a register row's cheque facts** — return and re-key.
- **No delete** — CANCELLED is the deletion.
- **No second cheque table** — `acc_voucher_cheques` stays empty.
- **No clearing straight to the party after a bounce** — re-issue first (C5), so Clear has one shape.
- **No refusing a bounce because an advance was used** — cascade (C4).
- **No stored due bucket** — computed in the SELECT.
- **No ledger-name lookups** — roles, and the cheque's own tender row.

