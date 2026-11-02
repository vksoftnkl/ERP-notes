---
name: notes-158-menu-restructure-withdrawn
description: "Menu tree history — notes 158 restructure withdrawn 2026-10-26 (its migration folder + _prisma_migrations row are gone); the restructure was re-applied as 20261027100000_menu_restructure on 2026-10-27, seed updated to match"
metadata:
  node_type: memory
  type: project
  originSessionId: 9e5de5d7-7b44-4f6c-8cfe-9ba61cec4ba9
  modified: 2026-10-27T04:10:02.155Z
---

2026-10-26: the notes 158 menu restructure ran on .106 and was withdrawn the same day. By 2026-10-27 the folder `20261026120000_menu_restructure_notes_158` and its `_prisma_migrations` row no longer existed.

2026-10-27: the user pasted the restructure again and asked for it as a migration → `20261027100000_menu_restructure` (applied on .106 via `migrate deploy`). It sets name/parent/position (steps of 10)/separator/visibility=true on 109 menus, hides 12 (13, 49, 106, 160, 161, 179, 181, 185, 187, 188, 258, 279), and grants VIEW on every ancestor group to users who hold a screen. Left out of the pasted SQL: its `CREATE TABLE fixed.menu_master_bak_20261026` (would be schema drift + a Db_Table_Master registry miss) and the BEGIN/COMMIT. Backup instead: `logs/backups/menu_master_user_menus_before_menu_restructure_20261027.sql` (data-only pg_dump).

`prisma/seed/Menu_Master.sql` now carries the same values (110 rows). Checked with a throwaway DB: all 116 seeded rows match live. 249/250/266/269/270 are still missing from the seed.

**How to apply:** menu 12 is now "Sales Bill", 283 "Till Setup", 34 "Item Masters", 17 "Sales Masters". 189 Optical Prescription (1.10) and 229 Sale Entry (Bulk) (3.10) were not repositioned, so they sort above Sales Bill. Any later restructure needs a NEW migration plus a matching seed edit (see [[prisma-migration-gotchas]]).
