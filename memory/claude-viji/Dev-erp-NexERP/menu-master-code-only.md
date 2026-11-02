---
name: menu-master-code-only
description: Menu Master holds only menus the code opens (user, 2026-10-30) — 137 unbuilt menus incl. whole Transport + Attendance removed by migration 20261030200000; re-add file in the share; how to compute "built"
metadata:
  type: project
---

**Decision (user, 2026-10-30):** "remove unused all menu and transport, attendance the whole module i need just simple what we have in our code". A menu is added back one at a time, only when its screen is built.

**What the migration does:** `ERP server/prisma/migrations/20261030200000_menu_master_remove_unbuilt` (applied on dev 2026-10-30 19:04: 265 → 128 menus, 1343 → 703 user_menus; committed in server 16074e95 and pushed; live gets it with the next deploy.sh).
- **Removes 137:** Sales 46, Reports 16, Settings 15, Stock 13, Attendance 13 (the whole module), Inventory 11, Transport 9 (the whole module), Purchase 8, Accounts 6. This includes 152 / 153 (e-Invoice / e-Way, which only answer "not built" on the sale bill's Shift+F2 / Shift+F3) and 286 (the retired first Cash Office).
- **Removes the rights:** 640 `user_menus` rows on dev, deleted first because of ON DELETE RESTRICT.
- **Keeps 144 Ledger Monthly Summary:** it is hidden, but ledger-statement grants the statement on view of 258 OR 144, and two users hold only 144.
- **Guards:**
  - it matches on id + name, so a live row that differs is left alone with a NOTICE;
  - leaves go first, and a folder that still has a child stays;
  - a menu a voucher type points at stays;
  - it never RAISEs (P3009).
- **Re-adding a menu:** `/home/vk/Shared/Prathap/admin/menu-cleanup-20261030/readd-menus.sql` has one idempotent INSERT block per menu (parents first) and the 8 `db_table_master` links the delete sets to NULL.

**How to compute "built":**
- **Qt:** probe the app itself. Text search misses `voucherMenu(id, …)` helpers and `fm.reg<X>(X::kMenuId)`, and catches commented-out lines. Link the build's objects without main.cpp.o; build a `FormManager` on a `NexMdiArea`; call the five `register*Forms` (declare them only, because the `form_*_menu.h` headers define them out of line and menu_window.cpp.o already holds them); then `isRegistered(1..5000)`. Add 174, which `MenuWindow::openModule` opens specially.
- **React:** the ids with an `href` in `components/layout/constants.ts` `MENU_ROUTES`.
- **Keep set:** built, plus the ancestors of built, plus any id code checks for rights (grep the server for `um_menu_id IN`, `MENU_ID` and the like).
- **`menu_separator = true` is NOT a divider row.** It is a real menu, with a line drawn after it (Qt `addSeparator()` after the item); 41 rows carry it, including Customers, Suppliers and Expense Voucher. Never drop those rows as "separators".

Related: [[live-vps-sync-migration-p3009]], [[user-run-commands-with-bang]].
