---
name: payment-design
description: "Payment (menu 100) + Issued Cheques (menu 52) design rev 1, 2026-09-14: share/payment/ — the receipt mirrored (td CR / abj DR), cheque book table, TDS deducted by us, beneficiary e-payment hook, ADVANCE_PAID holding ledger, P1-P15 + P11b taken, notes (9)"
metadata:
  node_type: memory
  type: project
  originSessionId: 5cd4e0af-1d3c-497e-9bf6-ad46ff0af939
  modified: 2026-09-28T13:01:14.263Z
---

**Folder:** `/home/vk/Shared/Prathap/payment/` — README, `payment_ui_mockup`,
`issued_cheques_ui_mockup`, `payment_tables` (.py/.png), `payment_flow.md`,
`plan-backend-payment.md`, `plan-qt-payment.md`, `plan-qt-issued-cheques.md`.
`notes.txt` **(9)**. Builds on [[receipt-design]] rev 3 and
[[received-cheques-design]]; assumes both §2s run. User confirmed P1–P15 on
the three PNGs ("ok") before the docs — the PNG-first pattern.

**Model:** ONE screen, three payee kinds (Supplier bill-wise / Customer
refund of an unapplied CR item / Other ledger, no bills); cash-bank payee →
"use Contra 104"; menu 49 retired. `Pmt` type (PAYMENT). `td_dr_cr 'CR'`,
`abj_dr_cr 'DR'` on CR bills, DR party / CR instrument. Identity
PAID + DEDUCTIONS + CREDITS APPLIED = ALLOCATED + ON ACCOUNT + ADDITIONS;
bank charges outside it (bank-leg split). No multi-ledger voucher (user).
- Credit applied = bill PAIR + ONE holding-ledger leg (DR party / CR the
  debit note's ledger or `ADVANCE_PAID`); `ck_av_self` forbids party/party.
  **Found + fixed the receipt flow's §4.2 three-leg version (didn't balance).**
- On account → DR Advances to Suppliers (role `ADVANCE_PAID`, NEW) + ADVANCE
  (DR) bill src PAYMENT; the purchase bill's adjust panel spends it.
- TDS deducted by us: `led_tds_deductee_type` + `led_tds_nature_of_payment`
  → `accounts.tds_rates`; PAN missing → 20 %; on the on-account part too;
  server re-seeds. Discount received seeded from `sup_cash_disc_perc` within
  `sup_credit_days` (role `DISCOUNT_RECEIVED`, NEW). Roles also NEW:
  `INTEREST_PAID`, `BALANCES_WRITTEN_BACK`.
- Issued cheques post at issue (CR bank); post-dated = own voucher dated the
  cheque; NO ON_CLEARING. `acc_cheque_book` (NEW): Post takes `chb_next_leaf`
  FOR UPDATE; a leaf is consumed by issue or VOID, never re-used, never typed.
  `apd_printed_on / apd_print_count` NEW.
- E-payment hook only: `td_beneficiary_name/_account/_ifsc/_upi` snapshot
  from `acc_ledger_bank_accounts`; `td_settle_status` PENDING until UTR +
  bank date; export screen later.
- Menu 52 = `acc_pdc_register 'P'` read the other way: HELD=ISSUED,
  CLEARED=PRESENTED (bank date → `av_recon_date`, NO voucher — **P11b:
  loosen `ck_apd_cleared` to R rows**), BOUNCED=RETURNED (ChqBnc reused,
  bills reopen, ADVANCE cascade), CANCELLED=STOPPED (reversal) or VOID
  (spoilt blank, amount 0). Replace = next leaf + new voucher + fresh allocs.
- ~~`sup_ledger_id`~~ **WITHDRAWN 2026-09-15** — `sup_id` IS `led_id`; there
  is no bridge column and one `partyId` everywhere ([[api-house-conventions]]).
- Settings: ~~`payment_needs_approval`~~ (approval withdrawn 2026-09-15), `tds_rates`,
  `cheque_print_after_post`, `epayment_default_mode`.

**2026-09-28 (user): build menu 100 "do as like receipt".** The TEMPLATE is the
Receipt screen AS BUILT (src/modules/accounts/receipt + tests/receipt), not
this rev-1 design — the receipt changed after its design (notes 30 draft
allocations, R20 posted amend, credits/other-ledgers, etc.). **Do NOT follow
the Voucher Register** (no bill-wise popup, side lock, party-first grid).
**REV 2 WRITTEN 2026-09-28:** share/payment/plan-backend-payment-rev2.md +
plan-qt-payment-rev2.md (supersede rev 1 for the Payment screen). Facts found:
NO /payments module (source or live); no `Pmt` type (Rct=12 receipt, PmtV=31
register); acc_cheque_book (acb_*), tds_rates, td_beneficiary_{name,account_no,
ifsc}, apd_printed_* ALREADY live; missing roles ADVANCE_PAID, DISCOUNT_RECEIVED,
BALANCES_WRITTEN_BACK; menu 100 hidden, 49 visible. Reuse: receipt
allocation-engine.ts (+direction IN/OUT), ppd-slab.ts, vouchers/cheque-book.helper
takeNextLeaf, voucher-facts tds lookup, common/posting. Defaults D1 no payee kind
(R5 any party), D2 new `Pmt` type, D4 print out. Waiting on user confirm + Prathap.

**Out by decision (P15):** e-payment export, bank reconciliation, Journal
103, Contra 104, GST expense 163, Proc Bank Payment 215, batch runs.

**2026-09-29: BACKEND BUILT by Prathap** (payment/ module + migration
20260929090000, applied live): Pmt = type 33, grid 123, menu 100 visible / 49
hidden, 3 roles mapped, setting payment_salesman_mandatory. Deviation (sound):
ADVANCE_PAID unused, remainder stays on the party (books-reconcile). The three
/receipts defects his payment fixes exposed are written up as
share/accounts/notes-61-receipt-fixes-from-payment.md — all server-only; our
Receipt client is unaffected by the stale amend header (it reloads via /get).
Review found: amend soft-deletes old leaves (gap on menu 52, unique index no
longer guards the leaf); no test for PDC cancel / amend with cheque.
Full review written as share/accounts/notes-62-payment-backend-review.md
(A1 TDS on an advance always 400s; A2 no Pmt type check on any route; A3
draft loses cheque book; A4 PPD from customer ppd_slabs; D1 renames
debitsHeld/totalDebits). Client contract facts: `roundoff` lower-case, cheque
object = chequeBookId/favouring/acPayee (no bankLedgerId), /post cheques[] is
the short form, full rows in /get chequesIssued[].
**2026-09-29 MOCKUP REV 2** share/payment/payment_ui_mockup_rev2.py/.png — the
receipt .ui as built mirrored (bands + ui_table 32/33 columns), worked example
balances (bank charge = an ADDITION, inside tdAmount), cheque + beneficiary
popups, 14 notes. Open: payee dropdown (dd 43 filtered vs supplier-first).
Waiting on the user's "ok" on the PNG before coding.
