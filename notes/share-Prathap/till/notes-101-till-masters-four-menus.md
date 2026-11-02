# Notes 101: Till Masters split into four screens, four menus (2026-10-09)

**STATUS:** done (migration 20261009190000). Checked live: menus 280–282 exist and 275 is renamed; the server judges each master by its own menu. The Qt test opens each list under its own menu: 51/51.

**Order:** independent of notes 99 and 100.

## What changed (the user's call)

Menu 275 "Till Masters" was one screen with four tabs. The user found it hard to understand and maintain, and one menu meant one right for all four masters. It is now four list screens, each with its own popup entry (NexCrudMasterPage + NexCrudForm, like every other master):

| Screen | Menu | Grid | Server routes it uses |
|---|---|---|---|
| Till Counters | **275** (existing) | 133 | `counters/create` |
| Till Safes | **280** (new) | 136 | `safes/create` |
| Till Reasons | **281** (new) | 134 | `reasons/create` |
| Denominations | **282** (new) | 135 | `denominations/create` |

Grids 133–136 were updated in place on the box (same ids). They now take `iinactive` ('true' = include inactive rows), and 134 also takes `icategory` ('' = all). Source: `share/grids/till_masters_registration.py`.

## Asked

### 1 · Menu rows

Please add these menu rows. I can't write `fixed.menu_master`, and there is no route for it.

| menu_id | parent | name | position | verbs |
|---|---|---|---|---|
| 275 | 271 | rename to **Till Counters** | 4.00 (as now) | as now: VIEW, CREATE, EDIT, DELETE, EXPORT |
| 280 | 271 | **Till Safes** | 4.10 | VIEW, CREATE, EDIT, DELETE, EXPORT |
| 281 | 271 | **Till Reasons** | 4.20 | VIEW, CREATE, EDIT, DELETE, EXPORT |
| 282 | 271 | **Denominations** | 4.30 | VIEW, CREATE, EDIT, DELETE, EXPORT |

If 280–282 are taken by then, tell me the ids you used. They are one line each in `form_sales_menu.h`.

### 2 · Rights per master

`till-masters.controller.ts` checks `TILL_MENU.MASTERS` (275) for all four masters. With four menus, please check each master against its own menu:
- counters: 275
- safes: 280
- reasons: 281
- denominations: 282

Without this, a user needs rights on 275 to save a safe, and the split gives no separate control.

`denominations/list` (the count grids) checks "272 VIEW, else 275 VIEW". It should read "272 VIEW, else 282 VIEW".

### 3 · Optional: names on `/get`

`counters/get` returns no `deviceName` / `safeName`, and `reasons/get` no `ledgerName`. The popups fill from the grid row instead, so nothing is blocked.

## Not asked

- Delete is not offered on any of the four lists. A master with history is deactivated (an Active on / off button) instead.
- The client sends every grid token, always.
