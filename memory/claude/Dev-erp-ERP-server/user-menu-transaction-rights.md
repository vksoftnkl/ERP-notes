---
name: user-menu-transaction-rights
description: The five um_can_post/cancel/amend/override/retender flags are now grantable through /user-administration/create — and omitting one REVOKES it
metadata: 
  node_type: memory
  type: project
  originSessionId: e9c9b22e-6446-4278-a2b8-28a014a35bbc
  modified: 2026-09-22T13:51:22.632Z
---

Wired 2026-09-22. `SaveUserMenuDto`, `UserMenuPayload(Dto)` and `replaceUserMenus` now carry `umCanPost`, `umCanCancel`, `umCanAmend`, `umCanOverride`, `umCanRetender`. Before this the columns existed (migration 20260921220000) but no request could set them, so all 1,111 rows were false and every sales `/post` answered 403 — the sale bill screen could not be tested at all.

**Why it bites:** a `menus[]` save is a **full replace**. Sending only the six CRUD flags does not "leave the rest alone", it revokes Post / Cancel / Amend / Override / Re-tender. The client must send all eleven.

**How to apply:** grant with `npm run rights:grant -- <login>` (`scripts/grant-posting-rights.js`) — additive, one row per menu, so it never wipes a 219-menu grid the way an API save would. It refuses a right the menu's `menu_verbs` does not allow. tester1 holds the sales chain (11, 12, 13, 182) as of 2026-09-22.

**The trap it fixes:** `/menu-masters/usermenu` builds the tree from the user's OWN rows and returns roots (parent NULL) with children nested — so a login granted Sales Entry (12) but not its parent "&1 Sales" (1) gets an **empty tree**: the right is real and the screen unreachable. The script adds a VIEW-only row for every ancestor.

Proved by `test/user-menu-posting-rights.e2e-spec.ts` (10 tests; creates and hard-deletes a throwaway user — never point that part at a real operator). `/menu-masters/usermenu` also answers `canPost…canRetender` under `permissions`. Read them together with [[menu-verbs-capability-column]].

**NOT built:** the roles layer the same handover sketched (`role_master`, `role_menus`, `usr_role_id`, role as a TEMPLATE applied on assign with `user_menus` staying the only thing the API reads). It was ranked third and "not urgent"; it also needs product decisions — which roles ship, and whether assignment is company-scoped.
