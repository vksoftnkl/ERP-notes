# notes (45) — `/bills/amend` answers 500 on a bill that earned loyalty points

2026-09-25. Please check the server log for the 500s at **10:44–10:51 today**
(bil00687, amended from the Qt screen) — the client cannot see the error text.

## What we know

| bill | loyalty EARN | amend (unchanged) |
|---|---|---|
| bil00687 (Freshora, 220 pts) | yes | **500** (later cancelled by the operator — cancel worked) |
| bil00688 / 689 / 690 / 692 / 693 (ZT, 0 pts; cash, card+surcharge, temp credit, credit) | no | OK, revision 2 |
| bil00686 (ZT, 12 pts) | yes | was 500 because the CLIENT sent an inconsistent payload (fixed today); now a proper 422 (credit limit) |

bil00687's payload was consistent with the stored bill (checked field by field),
so its 500 is server-side. Its only features the passing bills lack are
**loyalty points earned** and a B2B customer (GSTIN, REGULAR).

## Suspect (from the 24-Sep source snapshot — the live code has moved on)

`amend()` → `unwind()` → `loyalty.reverseForCancel()` writes a reversal row,
then `postCore()` earns again via `loyalty.earn()`, which inserts
`txnType 'EARN', rowNo: 1` (loyalty-ledger.service.ts ~line 731).

The original EARN row 1 is still live (`lld_is_deleted = false`), so

    ux_lld_src_row (lld_src_doc_type, lld_src_doc_id, lld_acc_year, lld_txn_type, lld_row_no)
                   WHERE lld_src_doc_id IS NOT NULL AND lld_is_deleted = false

refuses the second (SALE_BILL, bill, year, EARN, 1) → 23505 → 500.
Cancel never re-earns, which is why cancelling bil00687 went through (its
reversal was written as EARN row 2).

Fix options: number the re-earn after the highest existing row for the
document, or soft-delete the original + reversal pair on an amend (amend
restates in place, like the voucher legs).

**Live repro available:** bil00695 (Freshora, 8 pts, POSTED rev 1) — amend it
unchanged.

Also: a unique violation inside amend should map to a 409/422 with the
constraint name, not a bare 500.
