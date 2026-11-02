---
name: menu-verbs-capability-column
description: "fixed.menu_master.menu_verbs (text[]) says what a screen can DO — render a permission cell only for a verb the menu holds; RETENDER is on menu 12 alone"
metadata: 
  node_type: memory
  type: project
  originSessionId: e9c9b22e-6446-4278-a2b8-28a014a35bbc
  modified: 2026-09-22T13:51:10.310Z
---

Added 2026-09-22, migration `20260922170000_menu_verbs_capabilities`. `fixed.menu_master.menu_verbs text[] NOT NULL DEFAULT '{VIEW,CREATE,EDIT,DELETE,PRINT,EXPORT}'`; the 20 posting documents also carry `POST,CANCEL,AMEND,OVERRIDE`; menu **12 (Sales Entry) alone** carries `RETENDER`; 249/250 are `{VIEW,PRINT}`.

**Why:** `user_menus` has 11 flags and `menu_master` said nothing about what a screen IS, so the grid drew 11 checkboxes against 197 active leaf menus — 2,167 cells, most meaningless. Following the verbs gives 1,255. An absent verb must get **no checkbox at all**, not a greyed one: an unchecked box says *denied*, which is a different statement from *this screen has nothing to post*.

**How to apply:** `menuVerbs` is returned by `GET /menu-masters/get` and `/usermenu`; the Prisma field is `Menu.menuVerbs` in `prisma/fixed/menu.prisma` (never edit schema.prisma — see [[prisma-schema-is-generated]]). Vocabulary is deliberately unconstrained by a CHECK so a new screen type needs no migration. A `true` `um_can_*` on a menu lacking that verb is a stale grant, not a capability. Roles (`role_master` / `role_menus` / `usr_role_id`) were sketched in the same handover and are NOT built — see [[user-menu-transaction-rights]].
