# Notes 125: User Role uses the same names as the till approval levels (2026-10-16)

**STATUS:** asked. Server: a small enum change plus a data rename. Follows notes 96.

**Do §3 FIRST: it is a security hole.** Today any client ADMIN can create a SUPER ADMIN user through `/user-administration` (the server does not check), and so get VK Softwares' own powers: all settings, every company, and the implicit HO_FINANCE till grant. The Qt screen now hides the role and locks SUPER ADMIN users, but a direct API call still gets through. Items 1, 2 and 4 can follow later.

## Why

The user, looking at Approval Rules (276): the rules name **SUPERVISOR / STORE_MANAGER / CASH_OFFICE / AREA_MANAGER / HO_FINANCE**. But User Administration's "User Role" (`usr_type`) offers CASHIER / USER / ADMIN / SUPERVISOR / **MANAGER** / SUPER ADMIN. Only SUPERVISOR matches. Nobody can tell which user is a "store manager".

**The user's decision (2026-10-16): one list of names.** Notes 96 stays as it was:
- User Role is still a label. The till does not read it.
- A person's approval level still comes from their Approval Authority (285) grant, per company/branch, with an amount limit.
- SUPER ADMIN is still the implicit HO_FINANCE grant.

All that changes is that the two lists now use the same words. The 285 popup starts a new grant at the user's role.

## Asked

1. **Widen `UserType`** (`settings/userAdministration/types/user-administration.enum.ts`). User Role keeps the space spelling, like `SUPER ADMIN`:

   | add | matches till level |
   |---|---|
   | `STORE MANAGER` | STORE_MANAGER |
   | `AREA MANAGER` | AREA_MANAGER |
   | `HO FINANCE` | HO_FINANCE |
   | `CASH OFFICE` | CASH_OFFICE |

   `SUPERVISOR` is already there. CASHIER, USER, ADMIN, SUPER ADMIN, VIEWER and SYSTEM stay as they are.

2. **Rename `MANAGER` → `STORE MANAGER`** on the box (2 users today: `VKSOFT`, `zt-role-4804-manager`) and in the seed. Then drop `MANAGER` from the enum. Nothing in `src/` compares against `UserType.MANAGER`; we checked (`till-day.service.ts` `'MANAGER'` is a different thing, the day-open caller kind).

3. **SUPER ADMIN is VK Softwares' own account, never a client's role** (user, 2026-10-16: *"this user type is holding our acc to change setting, this role is not allowed to client"*). Please enforce it on the server, because today any ADMIN can save a user as SUPER ADMIN:
   - `user-administration` create/update: refuse `usrType = 'SUPER ADMIN'` unless the caller is a SUPER ADMIN (403 with a `code`, e.g. `USER_ROLE_RESERVED`).
   - The same for editing, deactivating, deleting or resetting the password of an existing SUPER ADMIN user.
   - Your call: hide SUPER ADMIN users from a non-super-admin's user list.
   - **The implicit HO_FINANCE till grant for SUPER ADMIN stays** (user re-confirmed 2026-10-16). No change in `assertApprover`.

4. **Check `isHqUserType`** (`selling-price-bulk.types.ts`, `['HQ','ADMIN','SUPERADMIN']`). Should `HO FINANCE` count as HQ for chain price saves? It's your call. If unsure, leave it out (deny by default, as the comment says).

Dropdown 68 (TILL APPROVER USERS) already returns `usr_type`. No change there.

## Qt side

- **Built 2026-10-16:** in the Approval Authority (285) popup, picking a user on a new grant sets **Level** from their User Role:
  - SUPERVISOR → SUPERVISOR
  - MANAGER or STORE MANAGER → STORE_MANAGER
  - AREA / HO FINANCE / CASH OFFICE → the same level
  - CASHIER / USER / ADMIN / SUPER ADMIN → stays at SUPERVISOR. SUPER ADMIN is not a client level; its implicit grant needs no 285 row.

  The level can still be changed by hand, e.g. a higher level in one branch.
- **Built 2026-10-16:** User Administration offers SUPER ADMIN only to a logged-in SUPER ADMIN (create). An existing SUPER ADMIN user opens **view-only** for anyone else (`lockReadOnly`, Save off): "A SUPER ADMIN user is changed only by a SUPER ADMIN." This is the screen only; §3 is the real guard. Please send the refusal with a `code`, so Qt can word it plainly.
- **Waiting for you:** User Administration's role list (`user_entry.cpp`) gets the four new names and loses MANAGER once items 1 and 2 are live. Until then a save with a new name would answer 400.

Tell Viji when it is deployed, and Claude will switch the list and check a save of each new role.
