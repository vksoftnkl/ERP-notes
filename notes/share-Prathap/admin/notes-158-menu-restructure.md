# notes-158 — menu restructure (names, order, separators)

Date: 2026-10-26. From: Viji / Claude. To: Prathap.

The user asked to rename every menu to a clear, current name, put each module in workflow order, add separators between the groups, and move items where they belong.

- **The new tree:** `admin/menu_restructure_2026-10-26.py`. Run it with no arguments to print the tree.
- **The SQL:** `admin/menu_restructure_2026-10-26.sql`.

**Please run the SQL on .106 once Viji says go.** The `claude` role is read-only, and there is no route that renames or moves a menu; `/menu-masters` has only GET and PATCH visibility.

What it does, in one transaction:
1. Backs up `fixed.menu_master` → `fixed.menu_master_bak_20261026`.
2. For 109 menus, sets `menu_name`, `menu_parent`, `menu_position` (10, 20, …), `menu_separator`, and visibility true.
   - **No new rows and no id changes**, so every `fm.reg(<id>)` and every grant stays valid.
   - Existing parent ids are reused. For example, 283 becomes "Till Setup" and now also holds Approval Rules 276 and Approval Authority 285.
3. Hides 12 menus that have no Qt screen behind them; they open "not available in this version": 13, 49, 106, 179, 181, 185, 187, 188, 258, 279, 160, 161. They are hidden, not deleted.
4. Grants every user who holds a screen view on each group above it (the `INSERT … ON CONFLICT` at the end), so a screen moved under a new parent never vanishes from someone's menu.

Every statement was checked against the live schema with EXPLAIN: 123 statements, 0 invalid.

**Undo:**

```sql
UPDATE fixed.menu_master m
   SET menu_name = b.menu_name, menu_parent = b.menu_parent, menu_position = b.menu_position,
       menu_separator = b.menu_separator, menu_visiblity = b.menu_visiblity
  FROM fixed.menu_master_bak_20261026 b
 WHERE b.menu_id = m.menu_id;
```

The ancestor grants it added are view-only on groups, and harmless to leave in place.

**Worth a route later:** `POST /menu-masters/update` (name / parent / position / separator) for Module Administration, so a menu change no longer needs SQL.
