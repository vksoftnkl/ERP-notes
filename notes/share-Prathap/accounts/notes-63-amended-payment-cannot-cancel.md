# notes (63): an amended payment can no longer be cancelled or amended again

2026-09-29. This follows your notes (62) C1 fix, verified live the same day. The fix works:
amend now CANCELs the old leaf, with the reason "Amended into revision N", and the leaf stays on
the book. But it opened a new hole.

## What happens

1. pmt00398 was posted by cheque: 70.00 to E2E-VCH-MUIEUX7D Chq Supplier Z, book ZT52-31409,
   leaf 94314094.
2. `/payments/amend` changed it to cash. The answer was "now revision 1". The leaf went to
   CANCELLED, not deleted, with reason "Amended into revision 1". So far, correct.
3. `/payments/cancel` on pmt00398 is refused:
   > Cheque 94314094 is CANCELLED. Once our cheque has been acted on the payment behind it
   > cannot be cancelled here — unwind it on the Issued Cheques screen (menu 52) first.

   There is nothing to unwind on menu 52: CANCELLED is terminal. **pmt00398 is stuck POSTED
   on .106.** Please cancel it once this is fixed.

## Why

`assertIssuedChequesStillHeld` (`payment-unwind.guards.ts:26-51`) refuses any live
`acc_pdc_register` row of the payment whose status is not in `CANCELLABLE_PDC_STATUSES`
(`[HELD]`). The row the amend just CANCELLED is live (`apd_is_deleted = false`, as C1 wants)
and not HELD, so it counts as "acted on".

Before C1, the amend soft-deleted that row, and the guard never saw it.

The same guard runs before an amend (`payment-amend.service.ts:125`), so **a second amend of
the same payment is refused too**.

## Fix

Leave out the rows the amend itself cancelled. Those are history of an earlier revision, not a
cheque somebody acted on. Either of these works:

- `apdCancelReason` starting "Amended into revision"; or better,
- a flag or column the amend sets, so the test does not depend on a sentence.

A row cancelled by **Stop** on menu 52 should still refuse, as today. That is a real act on the
cheque, and its own reversal belongs there.

## Test to add

Post a payment by cheque, amend it to cash, then:

- `/payments/cancel` → 200, and the leaf stays CANCELLED with its amend reason;
- a second `/payments/amend` → 200.

Then post by cheque, Stop the leaf on menu 52, and `/payments/cancel` → refused, as today.
