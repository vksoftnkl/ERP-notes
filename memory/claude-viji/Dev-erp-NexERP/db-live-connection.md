---
name: db-live-connection
description: The NexERP Postgres to inspect is 192.168.0.106 = localhost since 10-29 (psql -h localhost -U claude -d ERP, password in ~/.pgpass the user types) — never the cloudjiffy host; its SCHEMA is authoritative but its DATA is test-only
metadata:
  type: reference
---

The live NexERP database to inspect is on the LAN box **192.168.0.106**
(PostgreSQL 18.6, database `ERP`):

    psql -h 192.168.0.106 -U claude -d ERP -At -c "..."

**10-29 (after the move):** Claude now runs ON .106, so use `-h localhost`. `claude` is read-only (pg_read_all_data). `~/.pgpass` on .106 must be created by the user; if it is missing, say so, and never ask for the password. Migrations go through Prisma with the server .env (postgres). Before the move: the password was in `~/.pgpass` (`192.168.0.106:5432:ERP:claude`), so no
credential needs to be typed or pasted.

Do **not** use the `node279312-vknex-postgres.cloudjiffy.net` connection string
that sits in `/home/vk/Shared/Prathap/notes.txt` — the user rejected connecting
to it. Ignore any other cloudjiffy login (the panel URL, the ssh gate) in that
file too.

**Why:** The user said on 2026-08-29, after cancelling a cloudjiffy psql call,
"skip the login to cloudjiffy, use 192.168.0.106 always". Cloudjiffy is a remote
deployment; 192.168.0.106 is the dev server the schema and API are actually
built against (see [[api-endpoint-docs]], same host, port 3011).

**IT IS A TEST / DEV DATABASE — THE DATA IS NOT REAL** (user, 2026-09-12:
"the db you see is just for testing ... no need to care about the existing
data"). This changes what a query there is evidence OF:

- **Authoritative:** the catalog. Tables, columns, types, defaults, GENERATED
  expressions, CHECKs, FKs, indexes, partitions, triggers, and the registered
  config rows the client hard-codes against (dropdowns, grids, ui_tables, menus,
  voucher types). Schema conclusions drawn here hold.
- **NOT evidence:** row counts, balances, which rows are populated, and data
  quality. Do not argue "X is the real home because it has data", do not report
  a data anomaly as a production incident, and do not size a migration from it.
  Test rows are typed, abandoned and left behind by design.

When something looks wrong in the DATA, say what it would mean *if the same
shape appeared in production*, and keep the finding structural — a constraint
that cannot catch a bad row is worth reporting; the bad row itself is not.

**How to apply:** Whenever a question needs the *real* catalog — column names,
FKs, constraints — query 192.168.0.106 directly instead of inferring from the
.sql files in [[db-schema-share-location]]. It is a LAN host, so it is only
reachable while the user is on that network.

**Localhost scratch (added 2026-09-20):** `psql -h localhost -U postgres`
works from `~/.pgpass` (user gave the login). `erp_dry` is a schema-only
clone of 192.168.0.106/ERP (dump in the session scratchpad; rebuild with
`pg_dump -h 192.168.0.106 -U claude -d ERP --schema-only --no-owner
--no-privileges`). Use it to dry-run any share .sql before Prathap runs it;
it has no data, no ledgers, no series rows — seeds that JOIN masters come
back empty there and must be checked read-only against live.
