# Fix: `/bills/amend` returns 500 on a bill that has a temporary-credit (TEMP_CR) tender

## Context — what the error means

`POST /api/v1/bills/amend` failed with Postgres error 23505, raised by the partial unique index `ux_atc_tender`:

```sql
CREATE UNIQUE INDEX ux_atc_tender ON accounts.acc_temp_credit
  (atc_tender_id, atc_tender_acc_year, atc_acc_year)
  WHERE atc_is_deleted = false AND atc_tender_id IS NOT NULL;   -- migration 20260922000000
```

The rule is one live temp-credit row per TEMP_CR tender row. An amend runs these steps in one transaction ([bill-lifecycle.service.ts:351](src/modules/sales/bill/bill-lifecycle.service.ts#L351)):

1. **unwind** ([:1840-1865](src/modules/sales/bill/bill-lifecycle.service.ts#L1840-L1865)) sets the bill's temp-credit row to `atc_status = 'CANCELLED'`. It does **not** set `atc_is_deleted = true`, so the row stays in the index.
2. **applySaveInTx → `syncDocumentTenders`** ([tender-detail.service.ts:248](src/modules/accountsModule/tenderDetail/tender-detail.service.ts#L248)) updates the tender line in place when the client sends its `tdId` back. The TEMP_CR tender therefore keeps the **same `td_id`**.
3. **postCore** ([:1373](src/modules/sales/bill/bill-lifecycle.service.ts#L1373)) always *creates* a new `acc_temp_credit` row with `atc_tender_id = t.tdId`. The cancelled row from step 1 still holds that key, so the insert collides and the whole amend rolls back. Nothing is half-written.

So every amend of a posted bill that keeps a TEMP_CR tender fails. Other paths are not affected:
- **Cancel** never re-posts.
- **Retender** voids the tender and writes a new `td_id`.

The same bug was already fixed once for the doc register. `DocRegisterService.retire()` ([doc-register.service.ts:199-225](src/common/posting/doc-register.service.ts#L199-L225)) documents it: the unique indexes are "partial on `gdr_is_deleted` only, so a CANCELED row still holds the number and the re-post answered 23505". On an amend, that row is now soft-deleted instead of cancelled.

## Fix (follow the `retire()` precedent)

**File:** [src/modules/sales/bill/bill-lifecycle.service.ts](src/modules/sales/bill/bill-lifecycle.service.ts), `unwind()` step 7 (~L1856).

Change the `accTempCredit.updateMany`:
- Add `atcStatus: { not: 'CANCELLED' }` to its `where`. It then touches exactly the rows the `credits` read above it names. Credits an earlier retender already cancelled (on voided tenders) are left alone, and their remarks are no longer overwritten.
- In `mode === 'amend'`, also set `atcIsDeleted: true`. The row stays in the table as `CANCELLED` with balance 0, and its `txn_status_log` trail (CREATED … CANCELLED "Bill amended: …") is unchanged. The re-post then writes a fresh row with its own CREATED step and insert audit row. Add a short comment pointing at `retire()` for the reason.

A plain cancel keeps today's behaviour (`CANCELLED`, row still visible). No schema or migration change is needed, and `postCore` stays as it is.

Readers checked: all of them filter `atc_is_deleted = false`, so the retired row correctly drops out of each:
- `temp-credit.service` list
- `bill-read` tempCredits summary
- `bill-balance-recompute`
- receipt open-items
- the open-on-mobile checks

An amend can't strand money on the retired row: `assertCancellable` in `assertUnwindable` already refuses an amend once anything has been allocated against the bill.

## Test

Add a self-contained case at the **end** of [test/sales/bill-tender-http.e2e-spec.ts](test/sales/bill-tender-http.e2e-spec.ts) on a fresh bill, because the earlier cases depend on running in order:
- Create and post a walk-in bill with cash + TEMP_CR 500 (same body shape as the case at L98).
- `/bills/amend` with `baseRevision` and `editRemark`, sending the TEMP_CR tender back **with its `tdId`** and a changed amount/days. Model the request on the amend body in `test/sales/bill-qt-report-http.e2e-spec.ts:205`. Expect 201.
- Assert:
  - exactly one live temp credit for the bill: OPEN, new amount, new `atc_id`, same `atc_tender_id`
  - the old row is `atc_is_deleted = true` and `CANCELLED`
  - the old trail is `[CREATED, CANCELLED]` and the new trail is `[CREATED]`
  - `bill.tempCredits` has length 1

## Verification

1. `npx tsc --noEmit`. VSCode's Prisma types can be stale, so trust tsc over the editor.
2. `npx jest --config test/jest-e2e.json test/sales/bill-tender-http.e2e-spec.ts --runInBand`. The sales e2e specs need the rights fixture and `--runInBand`.
3. Repeat the original failing amend from the client against the `nest --watch` API on :3011 (it reloads from source). The PM2 API on :3000 serves `dist` and needs a build + restart before it picks up the fix.
