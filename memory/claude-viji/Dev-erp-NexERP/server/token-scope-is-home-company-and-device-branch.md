---
name: token-scope-is-home-company-and-device-branch
description: The access token's company_id is the user's HOME company (usr_company_id, may be null) and branch_id is the registered DEVICE's branch (null on web) — not a login choice; users routinely post in other companies; no user↔company/branch access table exists
metadata:
  type: project
---

auth.service.ts signs `company_id: user.usrCompanyId ?? null` and `branch_id: device?.devBranchId ?? null`
(the Qt login branch combo does not reach the token). Checked 2026-10-23 on the dev DB: 3 stock
vouchers, 3 bills, 3 accounts vouchers were made by users outside their home company, 3 vouchers / 3
bills from a device registered to another branch; 2 of 13 users have no home company. No table maps
users to the companies/branches they may work in (user_menus is not scoped either).

**Why:** "take company/branch from the token instead of the body" sounds like an easy hardening and
would refuse real usage; notes 143 #9 was settled as "actor only" for that reason.
**How to apply:** don't enforce body company/branch == token without first building a user-access
model; the login user (`requestContext.getUserId()`) IS safe to use as the actor. Related:
[[notes-143-opening-backend-review]], [[till-counter-signed-movements-notes-110]].
