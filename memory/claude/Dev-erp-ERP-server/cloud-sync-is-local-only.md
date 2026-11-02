---
name: cloud-sync-is-local-only
description: "Cloud-sync module + its migration are gitignored, but committed fragments still declare *_sync_date columns — every schema column needs a COMMITTED migration"
metadata:
  node_type: memory
  type: project
  originSessionId: 2ea9e424-41bd-4b73-aac5-3366550427d4
  modified: 2026-09-25T06:06:12.308Z
---

The offline→cloud sync feature (src/modules/cloud-sync, migration
20260923120000_offline_cloud_sync, prisma/public/syncDeletedRows.prisma) is
gitignored on purpose: the user wants it on local installs only, never on live.
But the committed prisma fragments declare the `*_sync_date` columns, so on
2026-09-25 live 500'd with "item_master.item_sync_date does not exist".
Fixed by committed migration 20260925110000_sync_date_columns (the 24 bare
nullable columns, IF NOT EXISTS, no triggers).

**Why:** schema.prisma is committed, the migration that backs it was not.

**How to apply:** when the local cloud-sync work adds a column to a committed
fragment, ship the bare column in a committed migration; keep triggers,
sync_deleted_rows and v_sync_tables in the ignored one. Verify with a shadow
replay of prisma/migrations minus the ignored folder against HEAD's schema.
See [[prisma-migration-gotchas]], [[vps-deploy-topology]].
