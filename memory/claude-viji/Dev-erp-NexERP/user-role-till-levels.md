---
name: user-role-till-levels
description: User decision 2026-10-16 — User Role (usr_type) uses the same names as till approval levels; 285 popup pre-fills Level from the role; notes-125 asked
metadata:
  type: project
---
User decided 2026-10-16: **one list of names**. User Role (usr_type, User Admin `src/modules/admin/master/user/user_entry.cpp` combo) gets STORE MANAGER / AREA MANAGER / HO FINANCE / CASH OFFICE (space spelling like SUPER ADMIN); MANAGER renamed to STORE MANAGER. Still a label only (notes 96): approval level comes from the 285 grant; SUPER ADMIN = implicit HO_FINANCE.

**Why:** the rules in 276 name SUPERVISOR/STORE_MANAGER/CASH_OFFICE/AREA_MANAGER/HO_FINANCE but User Role offered MANAGER etc. Only SUPERVISOR matched, so it was confusing.

**How to apply:** built 10-16: `TillApproval::levelForUserRole()` (till_approval_logic) + `NexDropdownSingle::selectedRow()`; a new grant's Level follows the picked user's usr_type (dropdown 68 returns it). PENDING: the User Admin combo switches only after Prathap deploys notes-125 (Shared/Prathap/admin/notes-125-user-role-matches-till-levels.md); before that a new name gets a 400. Related: [[till-approval-setup-screen]], [[till-approval-gate-missing]].

**10-16 later (user):** SUPER ADMIN = VK Softwares' own account for settings, never given to clients. Qt: User Admin combo shows SUPER ADMIN only to a logged-in SUPER ADMIN (userRoles() in user_entry.cpp); levelForUserRole no longer maps SUPER ADMIN. Server guard asked in notes-125 §3. User KEPT the implicit HO_FINANCE till grant for SUPER ADMIN (asked 10-16, answer "Keep it") — don't re-ask.
