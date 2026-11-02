# Notes 107: receipt surcharge, worked out on the amount for the bill (2026-10-12)

**STATUS:** open. Server. Small. The client already works with today's formula.

## What happened

Receipt for SEED-DEEPAN-1, receive 1,000 as CASH 100 + UPI 200 + CHEQUE 700. UPI has `tnd_surcharge_perc` 1.000 (CARD too). Post answered 400:

> The receipt does not balance • onAccount — Received 1000.00 + deductions 0.00 + credits 0.00 = 1000.00, but allocated 1000.00 + on account 0.00 + other income 2.00 = 1002.00. Out by -2.00.

The screen did not know about the surcharge. It does now.

## Decided by the user (2026-10-12)

**The customer pays the surcharge on top.** The bill gets the full amount the operator keys; the UPI / card line is raised by the surcharge.

## The rounding problem

`receipt-lines.ts` computes `surchargeAmt = tdAmount × perc / 100`, on the WHOLE tender amount. "On top" then needs tdAmount = 200 / 0.99 = **202.02** (surcharge 2.02) for 200 to reach the bill. The counter expects **202.00** (200 + 1%), as on the sale bill.

## Asked

For a percentage surcharge on receipts (and payments if they use it), take the surcharge as the part of the tender that is already inside it:

    surchargeAmt = round2(tdAmount × perc / (100 + perc))

202.00 at 1% → 2.00, and 200.00 goes to the bills. Please say when it is live; the client's `grossFor()` (receipt_entry.cpp) then becomes `forBills × (1 + perc/100)`, one line.

The flat `tnd_surcharge_amount` is unaffected.
