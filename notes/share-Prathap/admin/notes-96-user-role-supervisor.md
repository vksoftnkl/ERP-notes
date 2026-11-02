# Notes 96 — User Role: add SUPERVISOR to UserType

**From:** Qt client (Viji / Claude) · **Date:** 2026-10-07 · **For:** Prathap

## Why

User Administration now labels `usrType` as **User Role** and offers six values:

```
CASHIER · USER · ADMIN · SUPERVISOR · MANAGER · SUPER ADMIN
```

`UserType` (`userAdministration/types/user-administration.enum.ts`) already has CASHIER, but has no
**SUPERVISOR**. A save with it is refused today by `@IsEnum(UserType)` with a 400. There is no DB
constraint on `usr_type`, so this is enum-only.

## Asked

1. Add `SUPERVISOR = 'SUPERVISOR'` to `UserType`.
2. Grid 62 (the user list): rename the column header `Type` to `Role`. The column is still `usr_type`.

The role is a **label** only. It grants nothing by itself:
- what a user may open is still `user_menus`;
- what they may approve at a till is still `till_approval_authority` (47 §1.7).

Nothing else on the server needs to change.

## Acceptance

`POST /user-administration/create` with `usrType: "SUPERVISOR"` answers 2xx, and GET returns `usrType: "SUPERVISOR"`.
