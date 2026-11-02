---
name: receipt-amend-unwind-relies-on-partial-indexes
description: /receipts/amend soft-deletes rather than deletes because every unique index in its way is partial on the is_deleted flag
metadata: 
  node_type: memory
  type: project
  originSessionId: 8806e8bc-d8c9-4a60-b458-3ba8bf781fae
  modified: 2026-09-17T14:03:15.218Z
---

`POST /receipts/amend` (R20, added 2026-09-17) restates a POSTED receipt in
place: unwind, then re-run §5.1 and §5.2 from the new payload, same
`avh_voucher_id` / `avh_voucher_no` / `avh_voucher_refno`, `avh_revision_no + 1`.

It **soft-deletes and never drops**, and that only works because every
uniqueness rule that would otherwise block the re-apply is partial:

- `ux_av_voucher_row` … `WHERE av_is_deleted = false` → new legs renumber from 1
- `ux_apd_instrument` … `WHERE apd_is_deleted = false AND apd_status <> 'CANCELLED'`
  → the SAME cheque number can be re-keyed, which is the commonest reason to
  amend at all
- `ux_abl_doc_refno` … `WHERE abl_is_deleted = false` → the new ADVANCE bill
  reuses the receipt's reference

Verified 2026-09-17 against rct00039 with a control: inserting a duplicate
`apd_instrument_no` while the register row is live is refused by the index;
after the unwind's soft delete the same number inserts fine.

**Why:** register rows are soft-deleted and deliberately **not** marked
CANCELLED (which cancel does) — a cancelled register row asserts the instrument
was cancelled, and an amended receipt's cheque was not.

**How to apply:** before adding a unique index touching
`acc_vouchers` / `acc_pdc_register` / `acc_bill_balance` /
`acc_bill_adjustment`, make it partial on the soft-delete flag or amend breaks.
`acc_bill_adjustment` has **no** unique index on (voucher, row_no), which is why
reversal rows may share row numbers with the originals; `ux_abj_reversal` is
per-year on `abj_reversal_of_id`, so the unwind must skip rows already reversed
or a second amend fails. Related: [[ck-avh-balanced-fires-per-statement]].
