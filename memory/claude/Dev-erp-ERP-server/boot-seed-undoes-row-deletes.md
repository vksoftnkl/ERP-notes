---
name: boot-seed-undoes-row-deletes
description: A migration that DELETEs seeded rows is undone at the next API boot unless the prisma/seed file drops them too
metadata:
  node_type: memory
  type: project
  originSessionId: 7dec38b1-e3b9-4365-916e-a8ca5b53a062
  modified: 2026-10-31T05:45:15.944Z
---

PM2 runs the API with `DB_AUTO_SEED=true` on BOTH the dev box and the VPS, and every
`prisma/seed/*.sql` inserts with `ON CONFLICT DO NOTHING`, so a row a migration deleted is
simply re-inserted on the next restart. The migration still shows as applied.

Seen 2026-10-31: `20261030200000_menu_master_remove_unbuilt` removed 137 menus on dev
(10-30 19:04) and on the VPS (deploy 10-31 10:26:35). `Menu_Master.sql` still listed them,
so the PM2 boots at 10:11 (dev) and 10:27 (VPS) put all 137 back. Re-running the migration
SQL by psql (VPS: MIGRATE_DATABASE_URL from /root/.erp-deploy-secrets) removed them again:
dev 265 → 128, VPS 263 → 126. They return at the next restart until the seed is fixed.

**Why:** the migration record says done, so the comeback is easy to miss.

**How to apply:** any migration that deletes reference rows must also remove them from the
matching seed file in the same change. To check whether a delete held, compare `*_created_on`
with the PM2 `[Seed] Running …` log line. See [[vps-deploy-topology]],
[[notes-158-menu-restructure-withdrawn]].
