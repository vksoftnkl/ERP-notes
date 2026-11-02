---
name: sandbox-testing
description: How to test sync changes end-to-end without touching ERP or the live DB (private PG instance + latency proxy)
metadata:
  node_type: memory
  type: project
  originSessionId: 81f8b5fb-a16d-4f4a-90f9-bd1c4f890b7a
  modified: 2026-10-27T12:15:30.075Z
---

Reading ERP (pg_dump) and creating databases on the shared local PostgreSQL are blocked by the permission classifier, and runs against the live DB must be avoided. What worked (2026-10-27):

- Private PostgreSQL 18 in the session scratchpad: `initdb -U postgres --auth=trust`, start with `pg_ctl -o "-p 25433 -c unix_socket_directories='' -c listen_addresses=127.0.0.1"` (the scratchpad path is too long for a Unix socket).
- Synthetic schema of ~40 tables (serial ids, `updatedAt`, unique codes, FKs, tables without `updatedAt`, one without PK, an FK cycle, a NOT VALID check, `db_table_master` + `sync_deleted_rows` + mark-dirty triggers for the production flags).
- A small Go TCP proxy adding 290 ms each way: the live DB's round trip through the tunnel measured ~580 ms. A `-trace` option that greps SQL from client packets counts round trips.
- Run the binary from the scratchpad with `env -i`, so the project's `.env` (live URLs, mail) is never read; check with per-table md5 checksums of both DBs, or `-verify`.

**Why:** performance work on this tool is about round trips, and correctness must be shown with real inserts, deletes, swaps and cycles.
**How to apply:** rebuild this sandbox for any change to the sync path; see [[linux-service-install]] for deploying afterwards.
