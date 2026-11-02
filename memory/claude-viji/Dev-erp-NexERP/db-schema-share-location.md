---
name: db-schema-share-location
description: "DB schema scripts live in /home/vk/Shared/Prathap/schema (backend dev's share), outside git — not in db/ in the NexERP repo"
metadata: 
  node_type: memory
  type: project
  originSessionId: b94e5647-9223-4757-8374-77a9848305cb
  modified: 2026-08-10T04:42:16.733Z
---

Write NexERP database schema scripts to **/home/vk/Shared/Prathap/schema/**, not to `db/schema/` in the repo — on 2026-08-10 the repo's `db/` folder was moved to that share and deleted from git. Three subfolders there: `schema/` = the numbered backend chain `00_init.sql … 18_audit_projections.sql` (run in order), `grids/` = the client grid/list definition scripts, `legacy/` = the older loose scripts (`sale_bill.sql`, `charge_*.sql`).

**On 2026-09-02 the whole stock module moved to `schema/stock/`** — `16_stock.sql`, `16q`, `19_stock_posting.sql`, `19q`, `19m`, plus its .md/.pdf/.png docs and `16_docgen/`. The 16_/19_ filename prefixes were deliberately kept, so the run order is still `… 15 → stock/16 → 17 → 18 → stock/19`; see `schema/stock/README.md`. The stock .py/.png mockups stay one level up beside the other UI mockups.

**Why:** the backend developer, Prathap, works from that share and needs the schema directly; they don't have the Qt6 client repo.

**How to apply:** create and edit `NN_*.sql` there, keeping the house rules from [[sales-backend-schema-design]] unchanged (numbered run order, transaction tables partitioned by acc_year, varchar + CHECK). The share is **outside version control** — there is no history to fall back on, so read a file before overwriting it and say so when a change would otherwise be unrecoverable. Validate DDL by running it on a throwaway cluster (`initdb` in /tmp on port 55432, stub the FK master tables, run `00_init.sql` first) rather than against the live database.

**2026-10-28:** the share's master now lives on Prathap's PC (.106) at the same path; on the user's PC `/home/vk/Shared/Prathap` is an sshfs mount of it. See [[move-to-prathap-pc]].
