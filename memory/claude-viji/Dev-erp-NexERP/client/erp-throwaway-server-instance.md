---
name: erp-throwaway-server-instance
description: Run your OWN copy of the ERP server to verify new server code without touching dist/ or the shared dev server — tsc into scratch, NODE_PATH, REDIS_DB=15
metadata:
  type: reference
---

Used 2026-10-28 when the dev server (:3011) was down and :3000 ran a stale dist
([[erp-stale-dist-404]]). Never `nest build` — it wipes the shared dist/.

1. `cd "ERP server" && npx tsc -p tsconfig.build.json --outDir <scratch>/srv-dist` (~35 s).
   `src/...` imports stay unrewritten, so:
2. Run FROM the server dir (fonts load from `cwd/assets/fonts`, `.env` from cwd):
   `PORT=3095 HOST=127.0.0.1 HTTPS_ENABLED=false REDIS_DB=15 CLOUD_SYNC_ENABLED=false
   NODE_PATH="<scratch>/srv-dist:<server>/node_modules" node <scratch>/srv-dist/src/main.js`
   (run_in_background). Shell env beats `.env`.
3. `REDIS_DB=15` keeps the instance's BullMQ workers (audit-log-archival,
   stock-reconciliation) off the shared queues — the build includes other
   sessions' unfinished code. Mint the session into DB 15 too (`redis-cli -n 15`),
   per [[erp-auth-endpoint-verification]].
4. Browser checks: proxy `*/api/v1/*` to :3095 via CDP Fetch
   ([[erp-client-ui-browser-verification]]); fake write endpoints you are not testing.
5. Cleanup: TaskStop the server, delete DB 15's keys, delete test rows — a table
   with sync triggers also writes tombstones to `public.sync_deleted_rows`
   (sdr_schema/sdr_table) on DELETE; remove those for never-synced test rows.
