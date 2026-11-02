# notes (64): a cheque paid ON ACCOUNT cannot be stopped or returned

2026-09-29. Found while checking notes (63), which is fixed and verified. This one is on
`/issued-cheques/stop` (and so `/returned`, which shares the unwind), for a cheque written on a
menu-100 **Payment** whose money went on account rather than against a bill. Reproduced twice.

## What happens

Party: E2E-VCH-MUIEUX7D Chq Supplier Z. It reconciles cleanly before and after every step below
(`fn_party_bill_reconcile`).

1. A `/payments/post` of 10.00 by cheque (book ZT-PAY-01), with `allocations: []` and
   `onAccount: 10`:
   - pmt00497, leaf 94320003;
   - one ADVANCE (DR) bill of 10.00;
   - **0** `acc_bill_adjustment` rows carry `abj_cheque_id` = that leaf;
   - the party reconciles at −760.00 / −760.00.
2. `/issued-cheques/stop` on 94320003 is refused:
   > Party balance does not match its bills — E2E-VCH-MUIEUX7D Chq Supplier Z's open bills
   > come to −760.00 but the ledger says −770.00 (difference −10.00).

   The leaf stays HELD. I cancelled pmt00497 afterwards, and the party is back to −770.00.

The same stop on a cheque **allocated to a bill** works (pmt00496 → chqbnc00078, books balance).
So it is only the on-account part.

The first time (pmt00494) I cancelled the payment the same way, so **nothing is left stuck on
.106**.

## Why

`IssuedChequesService.unwind()` (`issued-cheques.service.ts`, the `standing` query near line
258) works out what the cheque discharged from its **bill adjustments**
(`abj_cheque_id` = the leaf). The ChqBnc voucher then puts the ledger back: DR bank / CR party.
But an on-account remainder writes no adjustment. It writes an **ADVANCE bill** (DR, src
PAYMENT_ADVANCE). The unwind never touches that bill, so after it:

- the ledger is back at −770.00;
- the ADVANCE still says we hold 10.00 of the party's;

and `assertVoucherBooksReconcile` rightly refuses the difference.

## Fix

In the unwind, the part of the cheque that went **on account** has to come off the ADVANCE bill
the payment created. That is the receipt's C4 cascade, mirrored:

- **ADVANCE bill still fully open:** delete or retract it by the cheque's on-account share.
- **ADVANCE already spent** (a later purchase bill or payment used it):
  - either refuse, naming the document that spent it (the receipt's rule);
  - or reopen what it settled.

  Your call. Refusing is the simpler and safer one.
- **Mixed payment** (part to bills, part on account): the bill part is the standing adjustments,
  as today. The on-account part is the cheque's amount less that, capped at the payment's ADVANCE
  still open.

**`/returned` shares `unwind()`, so it has the same gap** — please cover both.

## Tests to add

- A cheque payment fully on account → stop → 200: the ADVANCE is removed and the party
  reconciles.
- The same, but the ADVANCE was first spent by another payment → refused, naming it (or the
  bills reopened, if you choose that).
- Mixed: 10 to a bill + 5 on account on one cheque → stop → the bill reopens by 10, the ADVANCE
  goes, and the party reconciles.
- The same three with `/returned`.
