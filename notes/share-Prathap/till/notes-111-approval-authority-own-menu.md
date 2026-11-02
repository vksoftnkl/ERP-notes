# Notes 111: Till Approval Authority gets its own menu (285) (2026-10-13)

**STATUS:** DONE + verified live 2026-10-13: menu 285 + 276 renamed; grant routes judged on 285 (a user without it: 403 "menu 285: um_can_create is not granted"); the Qt test passes under menu 285.

## What changed (the user's call)

Till Approval Setup (276) was one screen with two tabs: Rules (WHEN) and Authority (WHO). The user wants Authority to work **"like other"** till masters: a list, with Add / Edit opening a popup, as Counters / Safes / Reasons / Denominations (notes 101). It is now two screens:

| Screen | Menu | Grid | Routes it uses |
|---|---|---|---|
| Till Approval Rules | **276** (existing, rename) | 151 (+ 152 read only) | `approval-rules/create`, `approval-rules/delete` |
| Till Approval Authority | **285** (new) | 152 | `approval-authorities/create` |

Grid 152 was updated in place on the box (same id). It now carries `grant_open`, `grant_status`, `event_label`, `max_label` and date labels, shows Remote / PIN as pills, and takes the token `iinactive` ('true' = ended or switched-off grants too). Source: `share/grids/till_approval_registration.py`.

A grant is still never deleted from the screen. "Active off" switches it off at once (`taaIsActive` false), as on the other till masters. "Active on" reopens it, and clears a Valid To that has already passed. A planned end is Valid To in the popup.

## Asked

### 1 · Menu rows

| menu_id | parent | name | position | verbs |
|---|---|---|---|---|
| 276 | 271 | rename to **Till Approval Rules** | 5.00 (as now) | as now |
| 285 | 271 | **Till Approval Authority** | 5.10 | VIEW, CREATE, EDIT, DELETE, EXPORT |

Copy 276's `user_menus` grants to 285 (the same flags), so nobody loses access. If 285 is taken by then, tell me the id you used. It is one line in `form_sales_menu.h`.

Please also add both to `prisma/seed/Menu_Master.sql`.

### 2 · Rights per screen

`till-masters.controller.ts` checks `TILL_MENU.APPROVAL_SETUP` (276) for both families. Please judge:
- `approval-rules/*` on **276**, as now;
- `approval-authorities/*` (create / get / delete) on **285**: add a `TILL_MENU.APPROVAL_AUTHORITY = 285`.

Without this, a user needs 276 rights to give a grant, and the split gives no separate control over who may hand out approval powers. That is the more sensitive of the two.

The enum comments that say "menu 276" for grants (`TILL_APPROVER_NO_GRANT` in till-enum.ts, and the decide-variance description) should say 285.

## Not asked

- No route changes. The screen fills its popup from the grid row, as the other till masters do.
- The "Who can approve?" preview is now a popup on the 285 list (F7). It is still the client's mirror of rule × grant matching.
