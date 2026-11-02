---
name: nexerp-prod-and-dev-topology
description: Where the NexERP backend lives on prod (169.58.213.171) and dev, how it deploys, and how grid/ui-table/dropdown config reaches prod (seed files regenerated from the dev DB)
metadata:
  type: project
---

- Prod VPS 169.58.213.171 (ssh root): NestJS API at /opt/erp-server/app (PM2 app `erp-api`, port 3000, nginx in front, host 169-58-213-171.sslip.io), local Postgres db `erp` (app role `erp_app`, tables owned by `postgres`), Redis. Prod tracks git branch `DEV-v1` of github.com/vksoftnkl/ERP-Server. Deploy = `/opt/erp-server/bin/deploy.sh` (git reset --hard origin/DEV-v1, npm ci, build, prisma migrate as postgres, grant-app-privileges.sql, pm2 reload; seeds run at boot from prisma/seed/*.sql, idempotent, ON CONFLICT DO NOTHING).
- Dev backend is on this machine: `/home/vk/Dev/erp/ERP server` (port 3011, Postgres db `ERP` as postgres, creds in its .env). The Qt client (this repo) points at it via appconfig.ini BaseUrl.
- Grids (fixed.grid_details/grid_columns), ui tables (fixed.ui_tables/ui_table_columns), dropdowns and widget form config are DATA, not code. New ones created on dev reach prod only via `npm run seed:export:ui-config` in the dev backend repo (rewrites 8 files in prisma/seed), then commit + deploy. As of 2026-10-06 the seed files were stale at grid 95 while dev had up to 132; regenerated locally (uncommitted) and dry-run OK on prod.
- Prod/dev drift exists in grids present on both (31,33,35,38,45-67,73-77,83,86,96,97,114,117,120,122,123 differ in SQL or column count) and ui_tables 2-6,19,20,22,23,39,41; dropdowns 36,38,45. Seeds never overwrite existing rows, so this needs a deliberate decision. Prod also has an orphan grid 124 "MAIN LIST - APP THEMES"; the client uses 125.

**Why:** the 404 "Grid with id 99 not found" on prod was simply a missing seed row, and the same gap covers ~24 grids and ~21 ui tables.
**How to apply:** when a screen 404s on prod for a grid/ui-table/dropdown id, compare `fixed.*` ids dev vs prod first, then regenerate and ship the seed files rather than hand-inserting. See [[prod-db-access-rules]].
