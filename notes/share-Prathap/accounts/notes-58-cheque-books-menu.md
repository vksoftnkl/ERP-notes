# notes (58): Cheque Books as their own menu under Accounts

2026-09-28. The user wants the cheque-book master as a separate menu under **&5 Accounts**. Today
it can only be reached from Issued Cheques (menu 52) through the "Cheque books" button. The Qt
screen is built and registered as **menu 263**. Two things are needed on your side.

## 1. The menu row

`fixed.menu_master` has no create route, so it needs a migration, the same way 259–262 came in:

| column | value |
|---|---|
| `menu_id` | **263** (the next free id; the client is registered on it) |
| `menu_parent` | 5 (Accounts) |
| `menu_name` | Cheque Books |
| `menu_visiblity` | true |
| `menu_position` | 13.10 (right after Issued Cheques, 13.00) |
| `menu_is_active` | true |
| `menu_verbs` | `{VIEW,CREATE,EDIT}`. A book is closed, never deleted, and nothing here posts or prints. |

If 263 is already taken on your side, use another id and tell me. The client registration is
one line.

## 2. Rights on the book routes

`cheque-books.service.ts:49` loads the caller's rights on `ISSUED_CHEQUES_MENU_ID` (52). With a
menu of its own, a user granted only **Cheque Books** can open the screen, but every save is
then refused, because the rights come from a menu they don't hold. Please judge
`/cheque-books/*` on **263**:
- `/list` and `/get` need VIEW;
- `/create` needs CREATE;
- `/update` and `/close` need EDIT.

**The Issued Cheques screen still opens the same dialog** from its button, for whoever has 263.
It's the same screen, reached from two places, and the rights are 263's in both. If you'd
rather keep 52 as a fallback ("52 OR 263"), say so. The client doesn't depend on which.

**Grant:** after the row exists, give VKPOS the three verbs on 263 so the live test can run.
Remember that `user_menus` save is a full replace (it once silently revoked four menus), so add
263 to the existing set, don't send it alone.
