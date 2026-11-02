---
name: user-menu-full-replace
description: "/user-administration/create FULLY REPLACES a user's menu array — granting one menu silently revokes every menu the grid did not tick"
metadata:
  type: project
---

**`POST /user-administration/create` fully replaces the `menus[]` array.** Its
own description says so: *"The menus array fully replaces existing menu
assignments: omit the field entirely to leave menus unchanged on update."*

**So a save made to grant ONE menu rewrites ALL of them**, and any menu the
grid did not tick comes back with every flag false. Nothing warns anybody.

**It has already bitten once, 2026-09-19.** Granting menu 250 (ledger map)
touched **70** of VKPOS's rows at 11:08 and created 250's in the same instant.
66 came back all-true; **four came back all-false** — 51 Received Cheques,
66 Offer & Schemes, 248 Printing Assignments, 249 Stock Track Policy. The
Received Cheques screen then showed seven dead buttons with only History live,
and the only way to find out why was to read `public.user_menus` on the box.

**How permissions actually reach the client** (not obvious, and not the
usermenu route):
- `GET /menu-masters/usermenu` returns the TREE and carries **no permission
  flags at all** — only `menuVisibility`. A menu with `um_can_view = false`
  still appears in it.
- `GET /user-administration/get?usrId=<self>` → `menus[]` is what fills
  `AppSession::m_perms` via `storeUserPermissions()`.
- `AppSession::canEdit(id)` etc. default to **true when the flag is ABSENT**
  ("an unloaded permission is not a denied one") but return false when it is
  present-and-false. So a real all-false row genuinely locks the screen.

**AND THE REAL ONE: `loadPermissions()` NEVER RAN, for any screen** — fixed
`81722d1`. `NexDialog::setup()` is documented as "called via signalInitialize()
after the subwindow is visible". **`signalInitialize()` does not exist** — it
appears in two comments and nowhere else. Every screen calls `setup()` from its
own CONSTRUCTOR, and `FormManager::openForm()` stamps `dialog->ctx.menuId`
only AFTER the factory returns, so `setup()` always hit

```cpp
if (ctx.menuId == 0)
    return;          // ← before loadPermissions(), every time
```

The five flags therefore kept `FormContext`'s declared **false**, and menuId
was stamped over the top. Screens that cache the decision at
`applyPermissions()` time never noticed (they run at menuId 0 and take the
escape hatch); the cheques register re-evaluates its gate on every selection
change, so it read `canEdit == false` forever and **no grant on the server
could revive it**.

`FormManager` now calls `dialog->applyMenuGovernance()` right after the stamp.
**This is the first time per-menu grants are actually enforced** — deliberately
NOT the whole of `setup()`, because `loadWidgetConfig()` is dead for the same
reason and turning it on HIDES FIELDS.

**PERMISSIONS ARE ALSO CACHED AT LOGIN — a repaired grant does NOT reach a
running app.** `MenuWindow::loadMyPermissions()` runs exactly twice: at construction
and on a user switch. Nothing refreshes `AppSession::m_perms` mid-session, and
closing and reopening the FORM does nothing because the grants live in the
session, not the form. **Sign out and back in, or restart.** There is no
refresh route and no UI for one — an open question worth closing.

**When a screen looks inert, check the grant before the code.**
```sql
SELECT um_menu_id, um_can_view, um_can_create, um_can_edit, um_can_delete, um_can_print
FROM public.user_menus
WHERE um_user_id = '<usrId>' AND NOT um_is_deleted
  AND NOT (um_can_view AND um_can_create AND um_can_edit AND um_can_delete AND um_can_print);
```
VKPOS is `019e441b-6e48-7918-b246-b857ffb35db1`.

**The cheques screen now SAYS it** (`bf11636`): `updateActionBar()` writes
"read-only — your login may not act on cheques" into the action bar in amber.
History stays live on purpose — it reads and writes nothing. Worth copying to
any other screen that greys a whole action bar; see [[cheques-screen-built]].


## The other reason a whole action bar goes grey

On the cheques screen a **terminal cheque** greys it too, and it looks
identical to a missing permission. `allowedActions()` gives CANCELLED,
RETURNED and REPLACED nothing but History — and a run of receipt cancellations
left **74 CANCELLED cheques** on the box, so most rows an operator lands on are
in that state.

`ChequeRegister::whyOnlyLooking(status)` now names it in the action bar
(`f2fdfdc`), and an EMPTY status gets its own sentence rather than being
silently treated as terminal — which is how a grid that stopped sending a
column would otherwise look. A live status returns nothing: a DEPOSITED row
that cannot be deposited again needs no explaining, because Clear and Bounce
are lit up beside it.

**Diagnosing this class of thing:** build the screen offscreen with the REAL
grants (`/user-administration/get?usrId=` → `storeUserPermissions`), stamp
`ctx.menuId`, call `applyMenuGovernance()`, select a row, and print every
button's `isEnabled()`. That separates "permission", "state machine" and "row
not selected" in one run.

**DO NOT hand-set `ctx.canEdit` in that diagnostic.** I did, concluded the
screen was fine, and said so — twice. It proved the GATE worked, not the LOAD,
and the actual defect was the load. Drive it exactly as FormManager does or it
is not a reproduction.

**Safe grant recipe (used for menu 79, 2026-10-05):** GET `/user-administration/get?usrId=` → echo every `menus[]` entry (only SaveUserMenuDto keys) + the new menu → POST `/create`; snapshot `public.user_menus` before/after and diff. The GET returns FEWER rows than the table (VKPOS: 92 vs 106) — the missing ones are `um_is_deleted = true` (already revoked), so the GET is complete; don't "restore" them from the table.
