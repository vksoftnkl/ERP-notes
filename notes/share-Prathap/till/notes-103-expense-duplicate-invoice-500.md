# Notes 103: /expenses/create answers a bare 500 on a repeated supplier invoice (2026-10-11)

**STATUS:** open. Server only. The Qt Expense Voucher (menu 277) needs no change once the refusal arrives: it already shows any `refusals[]` / field error it gets back.

**Order:** after notes 102.

## What happens

Save an expense with a GST bill, then save a second one for the **same supplier, same invoice number, same FY**. The second `/expenses/create` answers:

```
HTTP 500  { "message": "Internal server error" }
```

Found by the Qt live test (`tests/till/snap_expv.cpp`, scenario B, run twice with a fixed invoice number) and confirmed with a direct curl of the same body. The screen gets no field and no reason, so the operator cannot tell a double entry from a server fault.

## Why

`ExpenseService.create` writes `avh_doc_refno = gstBill.invoiceNo` and `avh_party_id = partyId` on `accounts.acc_voucher_header`. That hits the unique index from `vouchers.sql`:

```sql
CREATE UNIQUE INDEX ux_avh_doc_refno ON accounts.acc_voucher_header (
    avh_company_id, avh_party_id, avh_voucher_type_id, avh_acc_year, avh_doc_refno
) WHERE avh_doc_refno IS NOT NULL AND avh_is_deleted = false
    AND avh_voucher_status <> 'CANCELLED';
```

The index is right: it is the duplicate-supplier-invoice guard. But nothing in the expense path checks for the clash first, and nothing catches the Prisma `P2002` it raises, so it reaches `ExpenseExceptionFilter` as an unknown error and becomes a 500.

The general voucher already does this properly: `voucher-register.service.ts` (~line 1394) looks up `docRefnoClash` before writing, and `voucher-derive.ts` (~line 469) refuses `VCH.DUP_DOC_REFNO` with *"… is already on a live … for this party in this year"*, `field: 'header.docRefno'`.

## Asked

1. **Check before writing**, in `/expenses/validate` and `/expenses/create`: the same `exact` count as the voucher register (company, party, voucher type, acc year, doc refno, not deleted, not cancelled, **excluding this voucherId** so re-saving a draft does not clash with itself). On a hit, a refusal with:
   - `code`: `EXPENSE_DUP_INVOICE` (or reuse `DUP_DOC_REFNO`);
   - `field`: `gstBill.invoiceNo` (the filter's field pattern already passes `gstBill.*` through);
   - message like *"Invoice INV-123 from ABC Traders is already on expense voucher EXP/0042 this year"*. Naming the earlier voucher saves the operator a search.

   Doing it in `/validate` too means the screen shows it while the operator is still typing, not only on Save.
2. **Backstop**: wrap the header create/update in the existing `isUniqueConstraintError` helper (`common/utils/module-shared.utils.ts`) → `throwAccountsConflict` (409) with the same field, for the race where two terminals save the same invoice at once.
3. Optional: the voucher register's softer `OTHER` warning (same invoice number on a *different* voucher type for this party) would be useful here too, as a warning and not a refusal.

## Not affected

- Expenses without a GST bill: `avh_doc_refno` is NULL, so the index does not apply.
- No party: `avh_party_id` NULL never collides in a unique index.
- Cancelled or deleted vouchers are outside the index, so re-entering a cancelled invoice works today and should keep working.

## Test data

The Qt test now uses a unique invoice number per run, so it no longer trips this. Drafts it left on the box are `ZT-` prefixed. Leave or delete them as you like.
