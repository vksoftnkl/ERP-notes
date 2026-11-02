---
name: erp-menu-href-mapping
description: "How a header menu item gets its route — DB menu_master supplies tree AND labels, MENU_ROUTES maps menu_id → href/icon (no labels in the frontend since 2026-10-26)"
metadata:
  node_type: memory
  type: reference
  originSessionId: 87ced172-ac6c-43ce-912f-bd88db3a4c8a
  modified: 2026-10-26T13:53:52.563Z
---

The header menu is fully DB-driven for what it SHOWS. `/menu-masters/usermenu`
returns the user's granted tree (labels, ids, order, permission flags) from
`fixed.menu_master` + `public.user_menus`; `buildPrimaryMenu`
(`components/layout/constants.ts`) keeps the DB labels and looks each row up in
`MENU_ROUTES: Record<menu_id, {href?, iconKey?}>` to attach its route/icon. A DB
row with no entry renders as a plain (dead) label. The header shows NOTHING
until usermenu answers — there is no static fallback menu any more.

User asked (2026-10-26) that menu names never be static in the frontend:
renaming a menu in the DB must change the header and keep its route. Before
that, `DEFAULT_PRIMARY_MENU` matched by normalized LABEL, so a rename silently
dropped the href and — because the route stayed governed — locked everyone out.
Label matching also mis-iconed "Transport > Reports" / "Attendance > Reports".

Adding a route to a DB menu = add `<menu_id>: { href }` to `MENU_ROUTES`. Menu
ids are seeded explicitly by server migrations, so they match dev and live.
Find ids with a recursive query on `fixed.menu_master` (menu_parent).

Consequences:
- The item only APPEARS for users granted that menu id in `public.user_menus`
  (grant it in Settings → User Administration).
- Adding the href makes the route **governed** ([[erp-menu-permission-gating]]):
  `GOVERNED_ROUTES` = MENU_ROUTES hrefs + `GOVERNED_WITHOUT_MENU_ROW` (5 GSP /
  ledger bank+shipping screens with pages but no menu row — refused to everyone,
  as they were before; Customer links to ledger-shipping-address-master anyway).
