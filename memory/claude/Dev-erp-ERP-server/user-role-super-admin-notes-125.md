---
name: user-role-super-admin-notes-125
description: "Notes 125 (2026-10-16) — UserType uses the till level names (STORE/AREA MANAGER, HO FINANCE, CASH OFFICE; MANAGER renamed + dropped); SUPER ADMIN is VK Softwares' own: 403 USER_ROLE_RESERVED; /user-administration and /users were @Public until then"
metadata:
  node_type: memory
  type: project
  originSessionId: d006fa32-cd91-4155-862e-f20e41c74d33
  modified: 2026-10-16T05:28:05.623Z
---

User's rule (2026-10-16): SUPER ADMIN "is holding our acc to change setting, this role is not
allowed to client". Enforced in `userAdministration/super-admin.ts` (`callerIsSuperAdmin` reads the
caller's user_master row, not the token claim): giving the role, or changing / resetting /
deactivating / deleting a SUPER ADMIN user, by anyone else → 403 `USER_ROLE_RESERVED`. The
implicit HO_FINANCE till grant of SUPER ADMIN stays (re-confirmed). HO FINANCE is NOT an HQ type for
chain price saves (`HQ_USER_TYPES`, deny by default).

Found while doing it: `UserAdministrationController` and `UsersController` were `@Public()` — no
token at all could create users, and `PATCH /users/:id` could set ANY user's password. Both now need a
token; `/users` (no client uses it) needs a SUPER ADMIN. PM2 :3000 was restarted (user OK'd) so the
hole closed there too. Other `@Public` controllers: auth, health, app-theme, device-list-master.
Not done (not asked): menu-58 rights on user-administration writes — any logged-in user can still
save users / menus, only SUPER ADMIN is reserved. Grid 62 still lists SUPER ADMIN users (the grid
runner has no trusted caller param).

**How to apply:** an e2e suite calling /user-administration must send a token (notes-95 suite was
fixed). Migration `20261016110000` renamed MANAGER → STORE MANAGER. Related:
[[till-variance-decision-notes-105]], [[till-notes-124-open-list]].
