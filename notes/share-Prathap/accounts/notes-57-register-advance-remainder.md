# notes (57): Receipt / Payment Voucher — keep the unallocated remainder as an ADVANCE

2026-09-28. The Qt Voucher Register now opens a **bill-wise popup** as soon as a party amount is
typed on a Receipt or Payment Voucher. It opens pre-filled oldest-first, and whatever is not set
against a bill shows as **"Advance (on account)"**. Operators expect that remainder to post as an
advance, the way `/receipts` (menu 99) already does it (its R7: remainder = one ADVANCE bill).
`/vouchers` refuses it today.

## What refuses it

`voucher-derive.ts` (~1283), in the per-party-leg loop:

```ts
if (type.billwiseMode === 'DEMAND') {
  if (!allocated.equals(leg.amount)) {
    refuse(ctx, VCH.BILLWISE_SHORT, `${leg.ledger.name}: … must be allocated bill by bill — … is`, …);
  }
  continue;
}
```

RcpV and PmtV are both `DEMAND` (notes 53), so a receipt of ₹16,000 against ₹15,000 of open bills
can't be posted at all. The operator has to split it into two vouchers, and the register's own
hint told them to. The popup removes the reason for that workaround, provided the server takes
the remainder.

## The ask

For `DEMAND` on a RECEIPT / PAYMENT nature type:

1. **Allocated may be LESS than the leg**, never more. Over-allocation stays `BILLWISE_SHORT`
   (as it is for OPTIONAL today).
2. **The remainder raises one ADVANCE bill** on the party, for `leg.amount − allocated`:
   - `abl_bill_type = 'ADVANCE'`;
   - side = the leg's side. On a receipt, a CR on the customer: they have paid ahead. On a
     payment, a DR on the supplier.
   - dated the voucher, no due date;
   - one per party leg, and it must be linked to the voucher like the RAISE path links its bill.
   - It's the same row `/receipts` writes for its remainder, so the ADVANCE_ADJUST path on the
     sale bill and in later receipts (`bill.billType === 'ADVANCE'`) picks it up with no change.
3. **`derived`** says so, so the screen can show it before Post. Either add
   `advanceLines: [{ lineRowNo, partyName, amount }]`, or reuse `raisesBill` / `billAmount` per
   line, whichever is simpler for you.
4. **Setting, if you want a brake:** `accounts.voucher_allow_advance` (default true). When false,
   keep today's strict refusal. The client doesn't need it; decide on your side.

On a **Payment with TDS** (notes 53), the leg is the GROSS. So is the remainder: gross − allocated.
The advance is what we owe the supplier, and TDS on an advance follows the same deduction you
already make on the leg.

**Nothing changes for:**
- OPTIONAL (Journal): its remainder already stays unallocated, or raises a JOURNAL bill for a
  bill-by-bill ledger;
- the RAISE types (PurA, SalA, CrN, DrN);
- the cancel path: reversing the voucher must also retract the ADVANCE bill, the same way RAISE
  bills are retracted today (negative counter-row, never a delete, per the house rule).

## What the client does meanwhile

- The popup lets the operator leave a remainder. On a DEMAND type it shows a warning under the
  total: *"An advance on a Receipt/Payment Voucher is not accepted by the server yet (notes 57): set
  the whole amount against bills, or post the advance as a separate voucher."* Your existing
  `BILLWISE_SHORT` refusal still blocks Post.
- When this lands, we drop the warning. The payload doesn't change: the client never sends the
  remainder, since it is `leg − Σ allocations`.

**Tests to add:**
- RcpV ₹16,000 with ₹15,000 allocated posts, with one ADVANCE CR bill of ₹1,000;
- over-allocation is refused;
- cancel retracts the advance bill;
- a later receipt or sale bill can adjust that advance;
- PmtV with TDS: the advance = gross − allocated.
