---
name: avh-against-voucher-id-is-overloaded
description: acc_voucher_header.avh_against_voucher_id names BOTH a PDC child and a reversal voucher, so "orphaned child" queries give false positives
metadata:
  type: project
---

`accounts.acc_voucher_header.avh_against_voucher_id` carries two different
relationships, and nothing on the row distinguishes them:

- a **post-dated cheque's voucher** points at the receipt it belongs to
  (`receipt-posting.service.ts`, step 5);
- the **reversal voucher** that `/receipts/cancel` writes points at the voucher
  it reverses (`receipt-cancel.service.ts`, `reverseVoucher`);
- cheque-bounce vouchers (`chqbnc*`) use it too.

A reversal is deliberately a real, numbered, **POSTED** voucher — the day book
for the day of the cancellation has to show it. So every cancelled receipt
leaves a POSTED child of a CANCELLED parent, forever and correctly.

**Why it matters:** on 2026-09-18 a client report concluded from
`WHERE p.status='CANCELLED' AND c.status='POSTED'` that cancel abandons the PDC
child, citing 9 orphans holding 5,400. All 9 were reversal vouchers. Cancel does
reverse the children, in the same transaction, before marking the parent.

Discriminate with the parent's own reversal pointer:

```sql
AND p.avh_reversal_voucher_id IS DISTINCT FROM c.avh_voucher_id
```

**Two live consequences, found 2026-09-19:**

- `loadFullReceipt` (`receipt.service.ts`) collects a receipt's PDC children with
  `avhAgainstVoucherId = header.avhVoucherId` and no other filter, so
  `/receipts/get` on a **CANCELLED** receipt returns its reversal voucher inside
  `pdcVouchers[]`, mirrored legs and all. Not fixed. The filter wanted is
  `avhVoucherId !== header.avhReversalVoucherId`.
- The register is NOT affected: the `MAIN LIST - RECEIPTS` grid
  (migration 20260915120000) already has `AND h.avh_against_voucher_id IS NULL`,
  which excludes PDC children and reversal vouchers alike. A client report that
  a reversal "appears in the receipt list" is looking at something else.

Related: [[txn-status-log-stock-vouchers]], [[voucher-totals-trigger-missing]].
