---
name: erp-ledger-map-port
description: "Posting Ledger Map (menu 250) React port of Qt accounts/ledger_map — route, grids, the bare itrl_role picker param, and the two deliberate deviations from Qt"
metadata:
  node_type: memory
  type: project
  originSessionId: a6b71520-8575-49c2-a567-e57c0a261ce0
  modified: 2026-10-07T07:57:47.445Z
---

Built 2026-10-07: `features/accounts/ledger-map/` at route `/settings/ledger-map`,
bound in `DEFAULT_PRIMARY_MENU` under Settings → Configuration as "Ledger mapping"
(menu 250 exists in `fixed.menu_master` and is granted, so binding was safe).
API slice `store/api/ledgerMapApi.ts`; ui_table 37 `ledgerMapRoles`; picker grid
105 `ledgerForRolePopup` ("POPUP - LEDGERS FOR ROLE").

Non-obvious points:
- `/ledger-map/roles` is a **GET-mutation**, not a query: the screen edits a copy
  of the rows, and `refetchOnFocus` / structural sharing would either overwrite
  edits mid-session or make a reload a no-op. See [[erp-rtk-pending-query-dedupe-abort]].
- Grid 105 binds a **bare** `itrl_role` token (`r.alr_role = itrl_role`); it must
  always be sent in `grid_param` or the run fails.
- Save = one `POST /create` per dirty row, serial, `almId` always sent for mapped
  roles (omitting it is a 409), `isActive` echoed, cleared remark sent as `null`.
- **Deviations from Qt (deliberate):** dirty = differs from the loaded copy (not a
  touched flag); and after a save/unmap re-read, rows that failed or were
  remark-only KEEP the operator's edits (Qt's reload discarded them).
- Reuses opening-balance's `resolveOpeningColumns` and `grid-focus` walk.
- Live DB had all 42 roles mapped at build time, so the loud states (amber row,
  red pill, banner) were verified by rewriting `/roles` in-flight over CDP.

Related: [[erp-opening-balance-port]], [[erp-menu-href-mapping]],
[[erp-client-ui-browser-verification]].
