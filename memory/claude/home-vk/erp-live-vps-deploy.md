---
name: erp-live-vps-deploy
description: Live VPS 169.58.213.171 deploy — its DB is seeded independently of dev 192.168.0.106, and deploy.sh runs migrations but never the seed runner
metadata:
  type: project
---

The live ERP server is `ssh root@169.58.213.171 /opt/erp-server/bin/deploy.sh`
(app tree /opt/erp-server/app, branch DEV-v1, pm2 app `erp-api`). Two things
about it are not visible from the code:

1. **Its database is seeded independently of the dev database at
   192.168.0.106.** Master-data rows differ — different names, different ids,
   far less of it (13 items vs ~10,000). A data migration written by reading
   dev and hardcoding what it saw there will fail on live. Write the mapping
   against a *property* (taxability + rate, code, type) with the names as a
   reviewed override, not against a list of names.
2. **deploy.sh applies migrations but never runs the seed runner** — only
   `npm run prisma:migrate:deploy`, not `npm run db:deploy`. So anything that
   lives in `prisma/seed/*.sql` (grids, dropdowns, menus, widget config) never
   reaches live through a deploy. Config changes for live need a migration;
   fix the seed file too, for fresh databases.

**Symptom of cause 2, seen on live 2026-09-19:** configured grids only run to
id 97 there (dev has 103+), so the GST Rate master — which hardcodes grid 103 —
sits on "Loading GST rate data..." forever; grid 30 fails with `relation
"sales.loyalty_sch_points" does not exist`; and `list` on bank-lists,
device-list-masters, item-cust-rates, state-code-masters and
user-login-sessions all answer "No configured grid found". A live screen that
hangs on "Loading ..." is this, not a frontend bug — check the grid id first.

**Why:** on 2026-09-12 the tax_rate_master deploy failed with P3009 on cause
1, and cause 2 is why the fix had to be a migration rather than a seed edit.

**How to apply:** a failed migration is recovered with
`. /root/.erp-deploy-secrets` then, as the `erp` user with
`DATABASE_URL="$MIGRATE_DATABASE_URL"`,
`npx prisma migrate resolve --rolled-back <name>` — then re-run deploy.
Dry-run migration SQL against live first by replacing its final `COMMIT;`
with `ROLLBACK;` and feeding it to `sudo -u postgres psql -d erp`.
Check drift with `prisma migrate diff --from-schema-datamodel prisma/schema.prisma
--to-url "$MIGRATE_DATABASE_URL" --script`: that direction prints what the
*schema* is missing, so `ADD CONSTRAINT` means the DB has it and the schema
does not. The `SET DEFAULT` statements it always prints are generated-column
expressions Prisma cannot express — pre-existing noise, not drift.
See [[erp-server-http-testing-without-credentials]].
