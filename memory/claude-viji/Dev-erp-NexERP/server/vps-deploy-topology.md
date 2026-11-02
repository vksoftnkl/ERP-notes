---
name: vps-deploy-topology
description: Self-managed VPS 169.58.213.171 runs the API; migrations need the postgres superuser while the app runs as erp_app
metadata: 
  node_type: memory
  type: project
  originSessionId: f2f76393-9349-4cf6-bc5c-17b74ff869cd
  modified: 2026-08-21T10:05:23.418Z
---

As of 2026-08-21 the API is deployed on a self-managed Ubuntu 24.04 VPS at
`169.58.213.171` (root SSH, key `~/.ssh/id_ed25519`), separate from the
CloudJiffy topology in [[cloudjiffy-deploy-topology]].

Layout: code at `/opt/erp-server/app` (git clone, branch `DEV-v1`), run by the
unprivileged `erp` system user under PM2, which systemd resurrects via
`pm2-erp.service`. PM2 config is `/opt/erp-server/ecosystem.config.js` —
deliberately **outside** the git tree, because `deploy.sh` does
`git reset --hard` and would otherwise clobber it. Its `env` block is empty on
purpose so `/opt/erp-server/app/.env` is the single source of truth (dotenv does
not override vars already in `process.env`, so anything pinned in PM2 silently
wins). nginx terminates TLS and proxies to `127.0.0.1:3000`; Postgres, Redis and
Node are all loopback-bound with ufw allowing only 22/80/443. Secrets live in
`/root/.erp-deploy-secrets` (mode 600).

Redeploy with `/opt/erp-server/bin/deploy.sh [branch]` as root.

**Why:** two things about this app make a plain `npm install && pm2 start`
deploy fail, and neither is obvious from the repo.

1. **Migrations require a superuser, the app must not be one.** Seven migrations
   carry pgAdmin-style `ALTER TABLE ... OWNER to postgres`, which fails as any
   other role with `42501: must be able to SET ROLE "postgres"`. Dev never sees
   this because dev connects as `postgres`. So `deploy.sh` applies migrations
   with a separate superuser URL (`MIGRATE_DATABASE_URL`) while the app connects
   as `erp_app` — a SQL injection under a superuser role reaches
   `COPY ... PROGRAM` and therefore a shell, which is why they are split.
   Because migration-created objects are then owned by `postgres`,
   `/opt/erp-server/bin/grant-app-privileges.sql` re-grants `erp_app` after every
   migrate, including `ALTER DEFAULT PRIVILEGES` so future objects are covered.
   `erp_app` also needs **CREATE on `public`** — the seed runner creates
   `public._erp_seed_history` on first boot.

2. **PostgreSQL 18 is mandatory.** The schema defaults to `uuidv7()` and no
   migration defines that function — it relies on the native PG18 builtin.
   Ubuntu 24.04 ships PG16, where every insert would fail, so the VPS uses the
   PGDG apt repo. Extensions needed: `pgcrypto`, `btree_gist`.

Also note `public.fiscal_years` is **empty after seeding**, so the partition
bootstrap in the acc_year migration creates nothing and `sales.sale_bill` ends up
with zero partitions — see [[sale-bill-acc-year-partitions]]. `deploy.sh` works
around this by calling `ensure_acc_year_partitions()` for the current and next FY
on every run.

**How to apply:** for any change to this deployment, edit the scripts in
`/opt/erp-server/bin/` rather than running ad-hoc commands, and never point the
app's `DATABASE_URL` at the `postgres` role to "fix" a permission error — add the
missing grant to `grant-app-privileges.sql` instead.
