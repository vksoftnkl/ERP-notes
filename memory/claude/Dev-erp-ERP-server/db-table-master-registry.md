---
name: db-table-master-registry
description: public.db_table_master classifies every table (MASTER/TRANSACTION/LEDGER/BALANCE/LOG/SETTING/METADATA/SYSTEM) and names its sync-date column (dtm_sync_col); a new table needs a row in Db_Table_Master.sql or the drift spec fails
metadata:
  node_type: memory
  type: project
  originSessionId: 2195add3-2ce6-4061-95cc-67400252d6a7
  modified: 2026-10-23T11:05:39.345Z
---

Since 2026-10-23 (migration 20261023140000_db_table_master, user-approved design) every ordinary table and
partitioned PARENT has a row in `public.db_table_master` — kind, write mode (EDITABLE / APPEND_ONLY /
DERIVED), module, header (`dtm_parent_id`, like menu_parent), maintaining menu, column prefix, and since
2026-10-24 (migration 20261024100000) `dtm_sync_col` varchar(63): the NAME of the table's sync-date column
('item_sync_date', 'saa_synced_on', 'price_lvl_SyncDate'; NULL = none: acc_cheque_book, _prisma_migrations,
_erp_seed_history). Text, not a timestamp — the user wants the registry to list every table's sync field.
192 rows at creation. Rows live in `prisma/seed/Db_Table_Master.sql`: ids in blocks of 100 per module (1xx core … 13xx
transport), ON CONFLICT DO UPDATE (the file owns the rows), and rows whose table is absent are skipped.

**Why:** a migration that adds or drops a table without touching the seed makes
`public.v_db_table_master_drift` non-empty (UNREGISTERED / MISSING / PARTITION / SYNC_COLUMN — the seeded
sync_col is not the catalogue's: a timestamp column ending sync_date|sync_on|synced_on|syncdate, *_sync_date first) and
`test/db-table-master.e2e-spec.ts` fails.

**How to apply:** when writing a migration that CREATEs a table, add its row to Db_Table_Master.sql in the
same change (free id in its module's block) WITH its sync_col; one that DROPs a table deletes the row; one
that adds/renames a *_sync_date column edits sync_col. Rows are `(id, sch, tbl, parent, name, kind, module,
prefix, sync_col, menu, write_mode, keep_days, active, descr)`. Partitions are never
registered. `public.v_db_table_master` adds partition key/count, row estimate and size from pg_catalog —
don't copy those into the table. Column prefixes are NOT unique (stl_, atc_, sdr_, tax_ used twice).
See [[prisma-migration-gotchas]], [[cloud-sync-is-local-only]] (sync_deleted_rows is local-only, hence the skip).
