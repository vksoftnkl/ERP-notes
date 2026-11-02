---
name: live-vps-sync-migration-p3009
description: "Live ERP VPS lacks the sync functions; unguarded sync calls in Prisma migrations fail deploy with P3009, and how it was recovered"
metadata:
  node_type: memory
  type: project
  originSessionId: 2cfa5e4c-1dd9-4024-bbdc-c4d4e98c65e5
  modified: 2026-10-29T13:23:40.353Z
---

Live server: `ssh root@169.58.213.171`, deploy with `/opt/erp-server/bin/deploy.sh` (deploys branch DEV-v1 of the
ERP server repo, local checkout at `~/Dev/erp/ERP server`). Migrations run as `postgres` via `MIGRATE_DATABASE_URL`
from `/root/.erp-deploy-secrets`; the app runs as `erp_app`. The DB is small (~87 MB), so a full `pg_dump` plus a
rehearsal on a restored clone is cheap.

`prisma/migrations/20260923120000_offline_cloud_sync` is git-ignored, so `public.sync_install_triggers()`,
`fn_sync_mark_dirty()` and `fn_sync_log_delete()` exist only on local dev DBs. Any migration that calls them must wrap the
call in a `to_regprocedure(...) IS NOT NULL` DO-block (pattern: `20261028170000_print_share_link`). Otherwise the deploy
fails with P3009 (error 42883).

**Why:** On 2026-10-28/29, `20261028110000_purchase_b1_tables` and `20261028120000_stock_cost_revaluation_p13` called
`sync_install_triggers()` unguarded and blocked the live deploy.

**How to apply:** Recovery used on 2026-10-29:
1. Back up the DB.
2. Apply a copy of each migration with the call guarded, via psql in BEGIN/COMMIT.
3. Run `prisma migrate resolve --applied <name>` so the recorded checksum matches the repo file.
4. Re-run deploy.sh.

Rehearse on a restored clone DB first. Script: `/root/b1fix.sh`; backup in `/root/b1-fix-20261029/`.

Concurrent deploys: deploy.sh had no lock as of 2026-10-29, and the user and other sessions also run it, from the same IP.
Two overlapping runs clobber `node_modules` (`npm ci` ENOTEMPTY, TAR_ENTRY_ERROR, `prisma: not found`). Before
starting a deploy, check `ps -eo pid,etime,args | grep deploy.sh` on the server. Never start a second run or kill
someone else's run; wait for it with a `kill -0 <pid>` loop.
