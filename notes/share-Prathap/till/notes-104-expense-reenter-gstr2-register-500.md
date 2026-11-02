# Notes 104: posting a corrected expense after a cancel answers a bare 500 (2026-10-12)

**STATUS:** DONE, verified live 2026-10-12. The index is now split into `uq_acc_voucher_doc_register_doc_in` (with party, skips CANCELED) and `_doc_out`. exp00057 was cancelled and re-entered with the same invoice no; the copy posted as exp00058. The register has the CANCELED row and the POSTED row, IGST 180 each.

**Order:** after notes 103 (same family: an uncaught unique-index error becomes a 500).

## Background

There is no amend for an expense voucher. The README says "cancel and re-enter". The Qt screen now has a **Cancel & re-enter** button on a posted voucher. It calls `/expenses/cancel`, then opens a copy of the voucher as a new one. The operator corrects it and posts. The copy keeps the supplier's invoice no and date, because it is the same supplier bill.

`ux_avh_doc_refno` on `acc_voucher_header` already allows this: it skips `CANCELLED` rows. So `/expenses/validate` and `/expenses/create` both accept the copy.

## What happens

`/expenses/post` of the copy answers:

```
HTTP 500  { "message": "Internal server error" }
```

Live repro, 2026-10-12 (`tests/till/snap_expv.cpp`, `SNAP_POST=1 SNAP_REENTER=1`):

| voucher | status | doc_refno |
|---|---|---|
| exp00047 (`01a12e5e-9798-7730-b1cc-856de14eca03`) | CANCELLED | ZT-EXPV-1791779304 |
| copy (`01a12e5e-e280-7212-8e58-c20af460efb6`) | DRAFT, post → 500 | ZT-EXPV-1791779304 |

## Why

The cancel leaves the GSTR-2 row of exp00047 in `accounts.acc_voucher_doc_register` with `gdr_doc_status = 'CANCELED'`, `gdr_is_deleted = false`. That is correct for the register. But this unique index does not skip cancelled rows:

```sql
CREATE UNIQUE INDEX uq_acc_voucher_doc_register_doc ON accounts.acc_voucher_doc_register
  (gdr_company_id, gdr_branch_id, gdr_acc_year, gdr_doc_type, gdr_doc_no, gdr_doc_date)
  WHERE gdr_is_deleted = false;
```

The copy's register row has the same company, branch, year, `INVOICE`, invoice no and date. It collides, and the `P2002` is not caught.

**A second problem with the same index:** it has no party column. Two different suppliers who both number an invoice "101" on the same date cannot both be booked in one branch. For purchase-side rows (`docFlow INWARD`), `gdr_doc_no` is the supplier's number, so it is only unique per supplier.

## Asked

1. Fix the index so that a cancelled row does not block the re-entry, and the supplier is part of the key. For example:
   ```sql
   ... (gdr_company_id, gdr_branch_id, gdr_acc_year, gdr_doc_type, gdr_party_id, gdr_doc_no, gdr_doc_date)
   WHERE gdr_is_deleted = false AND gdr_doc_status <> 'CANCELED';
   ```
   Check the outward side (sales) still wants its own-number uniqueness; it may need its own index by `gdr_doc_flow`.
2. Catch the unique error in the expense post (`isUniqueConstraintError` → 409 with a field), as asked in notes 103. Then a clash says what it is instead of a 500.

## Client side

Nothing more needed. Until this is fixed, a GST expense that is cancelled and re-entered with the same invoice no cannot be posted. The copy stays saved as a DRAFT (it can be posted once the server is fixed). Non-GST expenses are not affected: they write no register row.
