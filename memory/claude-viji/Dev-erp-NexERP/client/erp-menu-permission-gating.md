---
name: erp-menu-permission-gating
description: Client-side menu permission layer (view/create/edit/delete/print/export) built on /menu-masters/usermenu — where it lives and how to gate a new screen
metadata:
  type: project
---

Built 2026-08-25. `/menu-masters/usermenu` returns each granted menu with a
`permissions` block; the client now enforces it.

- `lib/permissions/menu-permissions.ts` — pure index/resolution logic (unit-tested
  in `menu-permissions.test.ts`; `vitest.config.ts` include now covers `lib/**`).
- `hooks/useMenuPermissions.ts` — `usePagePermissions({href?, menuId?, disabled?})`.
  Shares the header's RTK Query cache entry, so it costs no extra request.
- `components/auth/menu-access-guard.tsx` — route gate, mounted in `app/layout.tsx`
  inside `.erp-app-content`.

**Policy (deliberate):** fails OPEN while the menu is loading and for any route
not listed in `MENU_ROUTES` (`GOVERNED_ROUTES` is derived from it since 2026-10-26, was `DEFAULT_PRIMARY_MENU` —
designers, `/home`, `/print-designer` are ungoverned). Fails CLOSED only when a
loaded menu lacks a route the catalogue does name. So **a new screen is only
gated once its href is added to `MENU_ROUTES`** (keyed by menu_id) in
`components/layout/constants.ts` — see [[erp-menu-href-mapping]]. `EXEMPT_ROUTES` in the same lib forces the
three designers (`grid-designer`, `dropdown-designer`, `ui-table-designer`) back
open — they are listed in the catalogue for navigation but are reached from the
record being configured, so they must never need a menu grant (user, 2026-08-25).

Wired consumers: `CrudMasterPage` (42 master pages — Add/Edit/Delete/Export, plus
the Alt+C / Alt+A shortcuts; props `permissionMenuId`/`permissionHref`/
`disablePermissionGating`), quotation and sale-order entry views (folded into
their existing `editable`/`canAlter` gates). Other bespoke screens (item master,
audit logs, companies, account-ledger, item-qty-price, templates) still only get
the route gate — call `usePagePermissions()` when wiring their toolbars.

**TRAP (hit 2026-08-27):** adding an href to the catalogue (now `MENU_ROUTES`) does not
merely register navigation — it makes the route GOVERNED, and `fixed.menu_master`
supplies the actual tree. Bind an href for which no menu row exists and the gate
denies EVERYONE ("You do not have access to this screen"), including admins, on a
screen that worked a moment earlier. So the DB menu row, the href binding and the
User Administration grant have to land together; until then leave the route
unbound and ungoverned. Cost a full browser-verification cycle on the new
`/settings/printing/**` screens.

**WAY PAST THE TRAP (2026-08-28):** if the screen replaces an older one that is
already in the menu, REPOINT that entry's `href` instead of adding one. The
DB row and every grant follow the menu id, so the new route inherits them and
nothing has to be seeded — this is how `/settings/printing/templates` was bound,
under menu 62 "Printing Configuration", whose old href pointed at the dead
`/settings/print-templates`. Matching is by prefix, so child routes come along.

Verified headlessly per [[erp-client-ui-browser-verification]] by stubbing the
usermenu response. Gotcha: Playwright matches the **last** registered `page.route`
first, so register a catch-all `**/api/v1/**` BEFORE the usermenu stub.

**Document rights (2026-10-09, Qt NexDialog port).** `canCancel`/`canAmend` are parsed from usermenu
beside `canPost`. `lib/permissions/document-rights.ts` holds the ONE draft/post rule: Create = save a new
draft, Edit = re-save a saved draft, Post = post (New needs Create OR Post), Delete drops a draft, Cancel
reverses a POSTED doc; an absent posting flag reads as allowed. Opening Stock (44), Physical Stock (45)
and Stock Adjustment (264) use it for buttons AND keys. The server does NOT enforce posting rights on the
stock routes, so this is the only gate — dev grants left VKPOS without Post on 44 (intended, as in Qt).
CrudMasterPage's Add/delete slot still gate on Create/Delete, so list-level New can't follow the rule yet.

