---
name: stock-engine-not-deployed
description: The stock posting engine in 16_stock.sql/19_stock_posting.sql is NOT on the live DB — posting, lots, reversal and policy resolution all live in the NestJS layer
metadata:
  type: project
---

Verified against 192.168.0.106 on 2026-09-09 via `pg_proc`. The `stock` schema contains
exactly FOUR functions: `fn_create_stock_partitions`, `fn_sml_forbid_delete`,
`fn_sml_freeze_guard`, `fn_sml_immutable` (the ledger-immutability guards).

**Absent:** `fn_svh_post`, `fn_svh_cancel`, `fn_slt_resolve`, `fn_stp_effective`,
`fn_svh_recompute`, `fn_svi_refresh_header`, `fn_sml_apply`. There are also **no triggers
on `stock_voucher_item`**.

**Why:** posting, lot resolution, ledger writing, reversal, balances, the moving average,
header totals, tracking-policy resolution and the "already has an opening" rule are all
reimplemented in NestJS. The `.sql` files on the share (see [[db-schema-share-location]])
describe a design that is NOT running.

**How to apply:** never reason about opening/physical/transfer behaviour from
`19_stock_posting.sql` or `16_stock.sql` §17 — they are a specification, not the
implementation, and the two have already diverged (see [[opening-stock-txn-list]] for
three confirmed divergences). Check behaviour against the API, not the schema. Advice to
"add a guard to fn_svh_post" is advice about code that does not exist.

**2026-10-01, clean slate:** 36_clean_slate.sql is BROKEN. It truncates the dropped accounts.acc_voucher_doc_api_log, so it rolls back and clears nothing, and its §9 re-adds stock_ledger guards against 37's "no SQL functions" rule. **44_clean_slate.sql replaces it.** It was dry-run on erp_clean_dry, a FULL data copy restored locally from a pg_dump -Fc of the box (~30 s dump + 10 s restore): exit 0, 37 txn tables → 0, 135 others unchanged, cheque books reset. The user's decision is to keep the item master; the wipe is a test-box alternative to the notes 77 repair.
