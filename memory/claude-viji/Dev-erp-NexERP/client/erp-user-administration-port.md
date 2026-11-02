---
name: erp-user-administration-port
description: "User Administration (menu 58) React port 2026-09-26 — where it lives, the wire facts the plan got wrong, and the list-shell traps it hit"
metadata:
  node_type: memory
  type: project
  originSessionId: 34eaabef-7963-4c26-9602-0c2eeb4a718f
  modified: 2026-09-26T14:22:17.695Z
---

Rebuilt 2026-09-26 under `features/masters/settings/user-administration/`
(domain/ is pure + vitest, api/ wraps `store/api/userAdminApi.ts`, the dialog
is bespoke — the shell's generic modal cannot hold "the complete set or
nothing"). Live acceptance is `user-admin.integration.test.ts`, gated on
`USER_ADMIN_IT_API=https://127.0.0.1:3011/api/v1` — no token needed, the
whole `/user-administration/*` controller is `@Public()`.

**Wire facts that differ from the port plan (verified against live Swagger + DB):**
- `usrType` IS an enum on the DTO (`@IsEnum`): SUPER ADMIN, ADMIN, MANAGER,
  USER, CASHIER, VIEWER, SYSTEM. The plan's "no enum" (§10 TYPE-ENUM) is stale.
- `/menu-masters/get?visibleOnly=true` returns 87 nodes, not the 125 visible
  rows: the server builds the tree from roots and never reaches a visible menu
  under a HIDDEN parent (38 of them). 139 live grants sit on menus the tree
  does not show, so the grants map must hold every loaded row and send it back
  untouched — `grantNothing`/"None" only removes tree rows.
- Real stale grants exist (menus 249/250/258 hold CREATE/EDIT/DELETE/EXPORT
  though their verbs are VIEW+PRINT); kept in state, not painted, sent back.
- `umCanView` is never read by `/menu-masters/usermenu` — a row's presence +
  `umVisibility` is what shows the menu. The View column is legacy.

**List-shell traps (CrudMasterPage):**
- Pass `listResponseStyleArrayKey=""` or the shell defaults the key to
  `"styles"`, finds none in a configured-grid run, and paints S.No only.
- `usr_login_name` ends in "name", so the accessor heuristic files it under
  `masterName` (display name) and every row showed the display name as its
  login (VKPOS read "sfkdjfk"). Fixed page-locally with `columnRenderOverrides`
  keyed by the SQL alias; the shell heuristic is shared by 42 masters.
- New opt-in props `deleteConfirmMessage` / `deleteConfirmNote` (string or
  fn(row)) on the delete dialog; the title stays the shell's.

**Skin:** the master-shell rule `.erp-ms-modal-overlay .erp-ms-modal` pins
width to 460px with a two-class selector — size a bespoke panel INLINE, and
give header / tab strip / footer `flex: 0 0 auto` or the tab strip shrinks
under the body.

Related: [[erp-menu-permission-gating]], [[erp-crud-master-extensions]],
[[erp-client-ui-browser-verification]], [[erp-crud-master-grid-column-heuristics]].

**2026-10-09 (Qt 09274cd):** the tree now loads `visibleOnly=false` (245 menus; hidden ones listed greyed,
`MenuNode.hidden`) — so the "87 nodes / 38 orphans" note above is history. Identity gained Till PIN
(`usrPin`: absent keeps, "" clears, digits replace; GET answers only `usrPinSet`), Employee (dd 38, always
given `iemp_branch_id`, nil uuid without a branch) and the SUPERVISOR role; label is now "User Role".


**2026-10-17 (Qt 425fa80, server notes 125):** User Role list = CASHIER, USER, SUPERVISOR, STORE MANAGER, CASH OFFICE,
AREA MANAGER, HO FINANCE, ADMIN (+ SUPER ADMIN only for a SUPER ADMIN session, `selectUserInfo().userType`). Server
enum: those + VIEWER, SYSTEM; MANAGER is GONE (migration 20261016110000 renames it to STORE MANAGER) and `toPayload`
answers `usrType: null` for an out-of-enum stored value. A SUPER ADMIN user opens read-only for anyone else
(`superAdminLock`); server enforces it too (403 `USER_ROLE_RESERVED`). A stored role missing from the list is kept and
shown, never blanked (`userRoleOptions`).
