# Permissions — the grid is too wide, and most of it is untrue

**2026-09-22.** Raised by the user while the sale bill's Post / Cancel / Amend
verbs were being wired: *"um_can_retender used only for sales only, why should
we keep it for every menu… too many permission will confuse the application
user."*

Right on both counts. The numbers, off the live box:

| | |
|---|---|
| active leaf menus | **218** |
| permission flags per menu | **11** |
| checkboxes per user | **2,398** |
| active voucher types (⇒ postable screens) | **13**, call it ~20 menus |
| cells where Post / Cancel / Amend / Override can ever mean anything | ~20 × 4 = **80** |
| cells where they cannot | **~870** |
| menus where **Re-tender** means anything | **1** (Sales Entry, menu 12) |
| menus where it does not | **217** |

A grid that offers "Re-tender" against Tax Master is not granular, it is
**lying** — and an administrator has no way to tell the meaningful cells from
the noise. That is the confusion, and it is a data-model problem wearing a UI
costume.

---

## What NOT to do — move the flag to `user_master`

The user's first instinct was to put `um_can_retender` on the user instead.
It fixes retender and leaves the principle unsettled:

* The same argument applies to `override`, so you end up with two rights on the
  user and nine on the menu, and **no rule for where the next one goes**.
* It loses granularity that may matter: "this cashier may re-tender at the
  counter but not on the POS screen" becomes unsayable.
* Two places to look when answering "why can't this person post?" is how a
  permission bug survives a support call.

---

## THE RULE: a verb exists on a menu if the menu can DO it

`fixed.menu_master` today is `menu_id, menu_parent, menu_name, menu_alias,
menu_visiblity, menu_position, three icon paths, menu_is_active,
menu_separator`. **Nothing says what the screen IS.** There is no way for any
client to know that Sales Entry is a posting document and Tax Master is a
master — which is exactly the missing piece.

### 1. Give the menu its capabilities

```sql
ALTER TABLE fixed.menu_master
  ADD COLUMN menu_verbs text[] NOT NULL
      DEFAULT '{VIEW,CREATE,EDIT,DELETE,PRINT,EXPORT}';

-- the ~20 posting documents
UPDATE fixed.menu_master
   SET menu_verbs = menu_verbs || '{POST,CANCEL,AMEND,OVERRIDE}'
 WHERE menu_id IN (11,12,13,14,182,251, 23,24,25, 41,44,45,
                   51,52,55,99,100,101,102,103,104);

-- re-tender is ONE screen
UPDATE fixed.menu_master
   SET menu_verbs = menu_verbs || '{RETENDER}'
 WHERE menu_id = 12;

-- read-only screens should say so rather than offering Create
UPDATE fixed.menu_master
   SET menu_verbs = '{VIEW,PRINT}'
 WHERE menu_id IN (249,250);   -- track policy, ledger mapping
```

`text[]` rather than a flags bitmask or a join table: it reads in psql, it
extends without a migration, and `menu_verbs @> '{POST}'` is indexable if it
ever needs to be.

### 2. The grid renders only what applies

A cell for a verb the menu does not have gets **no checkbox at all** — not a
greyed one. An unchecked box says *denied*, which is a different statement from
*this screen has nothing to post*, and across 2,398 cells nobody can tell those
apart. The administrator then sees six columns on a master and eleven on a
bill, which is the truth.

Result: **~870 meaningless cells disappear, and Re-tender appears on one row.**

### 3. Roles, which is what actually answers "too many permissions"

Capabilities shrink the grid. They do not stop it being per-menu clicking
across twenty transaction screens for every new employee.

```
role_master        role_id, role_name, role_is_active, role_remarks
role_menus         role_id, menu_id, the same 11 flags
user_master        + usr_role_id  (nullable — NULL means fully custom)
```

An administrator picks **Cashier**, **Supervisor**, **Accountant** or
**Manager** and the matrix is seeded. The per-menu grid stays, for the
exceptions, instead of being the only way in.

**The rule that keeps roles honest:** a role is a TEMPLATE, applied on assign
and on demand — not a live indirection. `user_menus` stays the single thing the
API reads, so "what can this person do" has exactly one answer and a role
change never silently alters a user who was deliberately customised. Show
"differs from role" on the user, and offer re-apply.

---

## What the client has done meanwhile

`user_menu_widget.cpp` carries a **bridge**: a `postingMenus()` set and a
`menuHasVerb(menuId, column)` gate, with the twenty menus listed and
`kRetenderMenuId = 12`. Cells that do not apply get no checkbox, All/None skip
them, and the read-back does not paint a stale grant onto a menu that cannot
use it.

**It is explicitly marked as temporary.** When `menu_verbs` lands, the set is
deleted and `menuHasVerb()` reads the server's answer — nothing else in that
file changes.

---

## Order, and what blocks what

1. **`SaveUserMenuDto` + `UserMenuPayloadDto` + the service — the five fields.**
   See `USER_MENU_POSTING_RIGHTS.md`. **This one blocks the sale bill screen**:
   every `um_can_*` posting flag is false for all 1,111 rows, so `/bills/get`
   returns `rights` all-false and the screen correctly greys every verb.
   Nothing can be tested end to end until it lands.
2. `menu_verbs` on `menu_master`, and expose it wherever the menu tree is read.
3. Roles. Worth doing, not urgent, and much easier once 1 and 2 are in.

