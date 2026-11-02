---
name: erp-partition-helper-correction
description: "ERP server: TWO acc_year partition helpers exist; ensure_acc_year_partitions is the one the app calls and the one new partitioned tables must be added to"
metadata: 
  node_type: memory
  type: project
  originSessionId: 7662a923-c4b5-499c-a215-d4a6f91491a9
  modified: 2026-09-21T10:24:42.300Z
---

`/home/vk/Dev/erp/ERP server` has **two** partition helpers, and the difference matters:

- **`public.ensure_acc_year_partitions(character)`** — an EXPLICIT per-table list, and the one the application calls. `receipt.guards.ts` tells an operator to run it by name, and six module READMEs document it. It also reaches the four `stock.*` partitioned tables via `stock.fn_create_stock_partitions`.
- **`public.fn_create_year_partitions(character)`** — catalogue-driven: scans `pg_partitioned_table` for LIST-partitioned, single-key tables in `public` / `sales` / `accounts`. It exists in the database but **is not in the migration history**, and nothing in `src/` calls it. It MISSES all four `stock.*` tables.

**How to apply:** when a pasted design note adds a table that is `PARTITION BY LIST (*_acc_year)`, add it to `ensure_acc_year_partitions` — `CREATE OR REPLACE` the whole body, copying the latest version forward and appending one `EXECUTE format(...)` per new table, parent before child where an FK links them. Backfill existing years by reading them off an established parent's partitions (`sale_bill` or `txn_status_log`) rather than hard-coding; installs differ. Verify with `SELECT public.ensure_acc_year_partitions('<a future year>')` inside a rolled-back transaction.

**A correction worth remembering as method, not just fact:** I asserted in four migration headers that `fn_create_year_partitions` "does not exist", because `grep` over `prisma/migrations/*/migration.sql` found no definition. It exists in the *database*. Absence from the migration files is not absence from the schema — this codebase has objects that were never in migration history. Ask the database (`pg_proc`, `to_regclass`), not the files. The adaptation was still correct for the reasons above, but the stated reason was wrong and had to be corrected in a later migration header.

Related: [[erp-pasted-design-notes-need-verification]]
