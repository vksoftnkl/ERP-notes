---
name: accounting-scope-register
description: "share/accounts/accounting_scope.md (rev 3, 2026-09-15): the full accounting form gap list vs Tally + GoFrugal, and the VOUCHER REGISTER decision — one type-driven screen for Journal/Contra/DN/CN/accounting-Purchase/accounting-Sales, which dissolves the GST-expense screen; S1-S17 decisions"
metadata:
  type: project
---

**File:** `/home/vk/Shared/Prathap/accounts/accounting_scope.md` — scope only,
no DDL/endpoints/layouts. Measures us against TallyPrime and GoFrugal;
**3.0 is a reference only, never the target** (the user corrected this
explicitly on 2026-09-15: "do not take 3.0 completely just use as a ref").

**The finding that reshaped it (user's observation):** in GoFrugal, Sales,
Purchase, Receipt, Payment, Debit Note, Credit Note, Contra and Journal are
**ONE form** with the type as the only variation, and **the register's "Sales"
and "Purchase" are NOT goods documents** — they are accounting vouchers with no
item/batch/godown/stock: party on one side, revenue/expense ledgers on the
other, GST as its own legs, invoice no+date, HSN/SAC + rate per line. That is
why they need no GST-expense screen: rent, freight, fees, AMC, fixed assets
with ITC, RCM all go in as *Purchase* vouchers.

**The model:** header + n legs (ledger, Dr/Cr, amount) + **per-leg** bill-wise
allocation; the voucher TYPE row carries all behaviour (numbering, which side
the main ledger sits on, legal ledger groups per side, whether party legs demand
bill-wise, GST register fed, day-book/cash-book classification). Validation is
only ΣDr = ΣCr + every party leg's allocations summing to it. Goods documents
post into the same tables, so the register shows both.

**We already have the structure:** `acc_voucher_header` + `acc_vouchers`
(`av_row_no`/`av_dr_cr`/`av_ledger_id`/`av_amount` + new `av_role`) +
`acc_bill_adjustment` keyed on **`abj_voucher_line_id`** (allocation hangs off
the LINE) + `acc_voucher_types` coarse flags. **Our Receipt/Payment already
write multi-leg vouchers** (OTHER LEDGERS band, one `av_role` leg per line);
P15 forbids only a FREE-FORM voucher on that screen and a SECOND party — not
multi-ledger. Do not repeat the "no multi-ledger" shorthand.

**Decision taken:** ONE register screen (Journal, Contra, DN, CN, accounting
Purchase, accounting Sales, free-leg mode) + the type master; the bill-wise
Receipt/Payment stay as the operator screens (tenders, cheque register,
credits, approval, print) but must serialize to the **same**
`{header, legs[], allocations[]}` payload so the posting engine and balance
validator are written once. Six MUST forms collapse to one screen; 15 MUST
forms + 12 MUST reports remain (was 19).

**Day close is LEDGER-based, not tender-based (rev 3, §2.5 — S19 decided).**
The ledger is complete by construction (every leg hits one; a tender row can be
absent) and a tender describes an instrument, not an account. Consequences:
the opening float is just the ledger's opening balance; **S18 decided —
tenders are optional instrument metadata on money legs, NOT an invariant**, so
the register needs no `ck_td_src_doc_type` change and no two-way validator;
tender rows keep only instrument/settlement facts + a tender-wise *report*.
Figures must come from a date-bounded sum of `acc_vouchers`
(`av_ledger_id`/`av_signed_amount`/`av_voucher_date`), **never** from
`led_total_balance`/`led_total_dr`/`led_total_cr` (running totals, no date).
Money ledger = group nature Cash-in-Hand / Bank Accounts / Bank OD A/c.
Tables (§6.1): `acc_day_close` (branch+date, status, closed_by, **the lock** —
the teeth are the lock, not the counting) + `acc_day_close_ledger` (opening,
movement, system closing, counted, variance) + denomination rows under cash
ledgers only. Bank ledgers are reconciled, not counted.
**S20 OPEN and the user's to call: one cash ledger per branch or per
counter/drawer** (this stands in for the till session, which the user is
deferring on purpose — "i will add session id later"); same for card/UPI
clearing ledgers per network/terminal vs shared. If a register voucher ever
needs tenders the additive shape is `td_voucher_acc_year` +
`td_voucher_line_id` (today `td_voucher_id` has no FK, no year, names the
voucher not the leg) + `'VOUCHER'` in the CHECK.

**New decisions S13-S17:** party cardinality (`avh_party_id` singular vs
multi-party register), rule columns on the voucher type, where the GST band on
a voucher lives (leg columns vs a voucher-detail table — blocks GSTR-2B/ITC),
register permissions/guards, and whether the leg payload is canonical for every
screen. S1-S12 are the earlier ones (cost centres, TDS/TCS, statements in Qt vs
React, Tally export, purchase ownership, GRN, day close, approval shape...).
Known prerequisite, not a decision: `avh_total_debit/_credit` must become a
trigger off the legs — see [[voucher-balanced-check-vacuous]].

**Build order:** wave 1 = register + type master + ledger statement/day book +
TB/BS/P&L; wave 2 = purchase entry/returns + outstanding & ageing; wave 3 =
bank recon + year close/period lock + day close; wave 4 = e-invoice/e-way
screens + GSTR-1/3B/2B. See also [[receipt-design]], [[payment-design]],
[[received-cheques-design]], [[opening-balance-concept]],
[[accounts-screens-own-ui-rule]].

**A DEFECT THAT LANDS ON DAY CLOSE (found 2026-09-16, notes (22)).** Three
accounts columns are `timestamptz` while every other date in the schema is a
plain `date`: `acc_voucher_header.avh_voucher_date`,
`acc_vouchers.av_voucher_date`, `acc_vouchers.av_doc_date`. Vouchers are
stamped `05:30:00+05:30` (midnight UTC), so any `col <= :to::date` compares
05:30 against midnight and **drops every row dated the last day of the range** —
i.e. today's documents never appear. Proved on grid 108: 1 row returned where
8 exist; `::date` on both sides returns 8.

**Why it matters here and not just on a list:** §2.5 settles day close as a
date-bounded sum over `acc_vouchers (av_ledger_id, av_signed_amount,
av_voucher_date)`. The same off-by-a-day will hit day close, the ledger
statement, the day book and the trial balance — silently, because the figures
stay internally consistent and are merely a day short. Fix is to make the three
columns `date` (a voucher date is a DAY — it is what the operator typed and
what `fy_lock_date` is compared against); cheap now at 2 and 8 rows.
