---
name: stock-ledger-cancel-leaves-forward-row
description: "Querying stock.stock_ledger: `sml_is_reversal = false` is NOT `not cancelled` — a cancel leaves the forward row live and adds a mirror; use unreversedLedgerRow() (added 2026-09-09) or the NOT EXISTS on sml_reverses_id"
metadata: 
  node_type: memory
  type: project
  originSessionId: 19873ae8-14fd-416f-b28a-f72ee65727a0
  modified: 2026-09-09T12:52:44.392Z
---

`stock.stock_ledger` is append-only. **A cancel does not touch, delete or flag the original row** — `cancelStockVoucher` writes a MIRROR beside it (`sml_is_reversal = true`, `sml_reverses_id = <original sml_id>`, direction flipped). So after a cancel the forward row is still there, still `sml_is_deleted = false`, still `sml_is_reversal = false`.

Any existence check written as `sml_is_deleted = false AND sml_is_reversal = false` therefore reads a **cancelled** movement as a live one. Sums are safe (the pair nets to zero); EXISTS / NOT EXISTS are not.

Use the exported fragment instead — `unreversedLedgerRow()` in `src/modules/stocks/stock-voucher/stock-voucher-posting.helper.ts`, which expects the ledger aliased `sml`:

```sql
sml.sml_is_deleted = false AND sml.sml_is_reversal = false
AND NOT EXISTS (SELECT 1 FROM stock.stock_ledger rev
                 WHERE rev.sml_reverses_id = sml.sml_id
                   AND rev.sml_acc_year    = sml.sml_acc_year
                   AND rev.sml_is_reversal = true
                   AND rev.sml_is_deleted  = false)
```

It probes the partial unique index `ux_sml_reversal (sml_acc_year, sml_reverses_id) WHERE sml_is_reversal AND NOT sml_is_deleted`, so it is one index lookup per row — repeat that index's predicate or you lose it.

**Why:** this shipped as a real bug (fixed 2026-09-09). A cancelled OPENING never freed its holding, so the corrected opening could never be entered — the only workaround was inventing a fake batch number, permanently corrupting lot identity. Three sites had it: the `opened` CTE in `validate()` (which `assertPostable` reuses, so it was the post guard too), the item picker's `alreadyOpened` badge, and `pendingItems`.

**How to apply:** when adding any new query over `stock_ledger`, decide whether you are summing (fine as is) or testing existence (needs the fragment). A backtick inside a `Prisma.sql` template ends the literal — write SQL comments without them.

Regression test: `test/opening-cancelled-holding-free.e2e-spec.ts` (live DB; creates a DRAFT and deletes it again). Reverting the fragment in the `opened` CTE turns it red with the exact user-facing message.

Related: [[stock-engine-ddl-lives-outside-repo]], [[erp-server-http-testing-without-credentials]], [[txn-status-log-stock-vouchers]].
