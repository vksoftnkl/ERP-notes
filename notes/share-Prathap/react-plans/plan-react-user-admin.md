# User Administration (user entry + list) — React port plan

Scope: **the user master.** It covers:
- the list (grid 62), with Add / Edit / Delete;
- the entry form, in two tabs: **Identity** (login, password, type, company / branch, login channels, edit rights, active, notes) and **Menu Permissions** (a menu tree × twelve permission columns);
- the save that writes the user **and** its complete permission set in one call.

It does **not** cover how the signed-in user's permissions are *enforced* in the app shell.
That is a separate concern (§8 says what this screen must do to it).

Qt source (673 lines, last changed 2026-09-22):

    src/modules/admin/master/user/
      user_entry.{h,cpp}        UserAdminEntry : NexCrudForm — the fields, the
                                password rule, the "permissions changed" signal
      user_menu_widget.{h,cpp}  UserMenuWidget — the permission tree, menuVerbs
                                gating, All / None, fieldValue() → menus[]
      user_list.{h,cpp}         UserMasterList : NexCrudMasterPage — grid 62,
                                delete by usrId
    src/core/api/api_endpoints.h    UserAdminMaster, Menu::UserMenus
    src/core/session/session_events.*  notifyPermissionsChanged(userId)

Companion documents this plan does **not** repeat:

- **`plan-react-dropdown.md`**: dropdowns 22 (companies) and 32 (branches), and the
  searchable picker.
- **`plan-react-app-settings.md`**: nothing here reads a setting, but it is the house
  shape for a master with a list + form.

> **The whole screen in one line:** a save sends the user **and the complete list of
> every menu they may open**. Any menu missing from that list is **revoked**, so the
> form must always hold the whole set, never a delta.

---

## 1. What this screen is, and what it is not

- **It is** the one place a user's rights are granted. Every per-menu flag the app enforces
  lives in `public.user_menus`, and this screen writes them.
- **It is not** a role / group system. There are no roles; each user carries their
  own 218-odd menu rows. (A "copy rights from user…" button is the cheap substitute,
  §6.6, and is new.)
- **It is not** where a user changes their own password. There is no self-service
  route; an admin sets it here.

## 2. Module layout

```
src/features/userAdmin/
  UserListPage.tsx            grid 62 through the shared registered-grid list; Add / Edit / Delete
  UserEntryDialog.tsx         two tabs; Save / Cancel; the unsaved-changes guard
  tabs/IdentityTab.tsx        the fields of §4
  tabs/PermissionsTab.tsx     the tree of §6 (virtualised: ~230 rows × 12 cells)
  domain/                     no React, no fetch
    menuTree.ts               parse /menu-masters/get → nodes; verbs per menu; flatten
    grants.ts                 Grants = Map<menuId, Grant>; apply loaded; toMenusPayload()
    grantDiff.ts              what a save grants / revokes vs the loaded set (§7.3)
    columns.ts                the 12 columns, their verb, their tooltip
  api/userAdmin.ts            get / create / delete, typed (§3)
  api/menuMasters.ts          GET /menu-masters/get
```

`domain/*` holds every rule the tests check (§11). `toMenusPayload()` is the **only**
function that builds `menus[]`.

## 3. The wire contract (live Swagger, 2026-09-26)

| Route | In | Out |
|---|---|---|
| `GET /api/v1/user-administration/get` | `usrId` | `data`: `UserAdminPayloadDto`, **including `menus[]`** (every active row of the user) |
| `POST /api/v1/user-administration/create` | `SaveUserAdministrationDto` | create when `usrId` is absent, update when present |
| `DELETE /api/v1/user-administration/delete` | `usrId` (query) | soft-deletes the user **and all their menu rows** |
| `GET /api/v1/menu-masters/get` | `visibleOnly=true` | `data`: the menu **tree** |
| `GET /api/v1/configured-grid-sql/run` | `grid_id=62` … | the list (§9) |

**`SaveUserAdministrationDto`** (`*` = required):
`usrId`, `usrCompanyId`, `usrBranchId`, `usrEmployeeId`, `usrLoginName*` (≤ 50),
`usrDisplayName`, `usrFullName`, `usrMobileNo`, `usrEmail`, `usrAvatarUrl`,
`usrTimezone` (≤ 60), `usrLanguage` (≤ 10), `usrPassword`, `usrMustChangePassword`,
`usrType`, `usrEditDate`, `usrEditEntry`, `usrEditRate`, `usrDesktopLogin`,
`usrWebLogin`, `usrMobileLogin`, `usrIsActive`, `usrNotes`, `menus`.

The API runs **`forbidNonWhitelisted: true`**. Any other key 400s the whole save.
Build the body from the form state, **never by echoing the GET payload**: the payload
carries `usrIsLocked`, `usrFailedLoginCount`, `usrCreatedOn`… which the save refuses.

**`menus[]` item (`SaveUserMenuDto`):**
- `umMenuId*`;
- `umCanView`, `umCanCreate`, `umCanEdit`, `umCanDelete`, `umCanPrint`, `umCanExport`;
- **`umCanPost`, `umCanCancel`, `umCanAmend`, `umCanOverride`, `umCanRetender`**;
- `umVisibility`, `umIsFavourite`, `umIsPinned`, `umSortOrder`.

Swagger's own words on `menus`: *"Full replacement set of menu permissions. Existing
menus not in this list are soft-deleted."* **Omit `menus` entirely** to leave them
unchanged on an update; do not send `[]` for that, because `[]` revokes everything.

**A menu-tree node:**

```jsonc
{ "menuId": 10, "menuParentId": 1, "menuName": "Customers", "menuAlias": "CTRL+SHIFT+C",
  "menuVisibility": true, "menuPosition": "1", "menuSeparator": true, "menuIsActive": true,
  "menuVerbs": ["VIEW","CREATE","EDIT","DELETE","PRINT","EXPORT"],
  "permissions": null, "children": [ … ] }
```

The live tree has 7 roots. `permissions` is `null` on this route; it is not how a user's grants
arrive (they come from `/user-administration/get` → `menus[]`).

## 4. The Identity tab

| Field | Key | Rule |
|---|---|---|
| Login name | `usrLoginName` | required, ≤ 50, trimmed |
| Password | `usrPassword` | **required on create**; on edit, blank = keep (the key is **omitted**, §5) |
| Full name | `usrFullName` | |
| Mobile | `usrMobileNo` | |
| User type | `usrType` | required; a static list `USER · ADMIN · MANAGER · SUPER ADMIN` (the four values on the box). The DTO has no enum and the table no CHECK (§10) |
| Company | `usrCompanyId` | dropdown 22 |
| Branch | `usrBranchId` | dropdown 32. **Scope it to the chosen company.** Qt does not (§12) |
| Desktop / Web / Mobile login allowed | `usrDesktopLogin` / `usrWebLogin` / `usrMobileLogin` | defaults: true / true / false |
| Allow edit date / entry / rate | `usrEditDate` / `usrEditEntry` / `usrEditRate` | default false |
| Active | `usrIsActive` | default true |
| Admin notes | `usrNotes` | textarea |

**Fields the DTO has and Qt never showed:** `usrDisplayName`, `usrEmail`,
`usrEmployeeId`, `usrMustChangePassword`, `usrTimezone`, `usrLanguage`, `usrAvatarUrl`.
Decide per field (§10 Q1). The plan assumes the first three plus "must change
password" are shown, and the rest are left unsent.

**Read-only facts to show on edit** (from the payload, never sent back): `usrIsLocked`,
`usrFailedLoginCount`, `usrLockedOn`, `usrLastLoginOn`, `usrPasswordChangedOn`.
A locked user needs an **Unlock** action, and there is no route for it (§10).

## 5. The password

- **Create:** required; the client refuses before the round trip ("Enter a password").
- **Edit:** the field starts empty. Blank means **the key is omitted**, never sent as `""`,
  which Qt strips in `beforeSubmit`. Typing sets a new one.
- Show it masked, with a reveal toggle. No strength rule exists on the server (§10); if
  one is wanted, it is a server rule first.
- `usrMustChangePassword` (if shown): the next login forces a change. Whether the login flow
  honours it is unverified (§10).

## 6. The Permissions tab

### 6.1 Rows

- The tree from `/menu-masters/get?visibleOnly=true`, fully expanded by default.
- Each node is a row, parents included. A parent is itself a menu, with an id, and is sent
  like any other.

### 6.2 Columns

| # | Column | Verb | Tooltip (from Qt, keep) |
|---|---|---|---|
| 0 | Menu (visibility) | — | |
| 1–6 | View · Create · Edit · Delete · Print · Export | VIEW · CREATE · EDIT · DELETE · PRINT · EXPORT | |
| 7 | Post | POST | "Post a document — commit stock, ledger and register." |
| 8 | Cancel | CANCEL | "Cancel a POSTED document. It reverses; it never deletes." |
| 9 | Amend | AMEND | "Reopen a POSTED document and post a revision. A document already declared to GST cannot be amended by anyone — this right does not unlock that." |
| 10 | Override | OVERRIDE | "Waive an overridable WARNING at post time … Refusals can never be overridden." |
| 11 | Re-tender | RETENDER | "Change how a posted bill was PAID, without editing what was sold. Its own right, deliberately NOT the same as Amend." |

### 6.3 A cell exists only if the menu has the verb

- `menuVerbs` decides. A verb the menu lacks gets **no checkbox**: not a greyed one, not an
  unchecked one. An unchecked box says "denied", which is a different statement from
  "this screen has nothing to post".
- On the box: about 205 masters have the six CRUD verbs, 19 posting documents add
  POST / CANCEL / AMEND / OVERRIDE, 2 read-only screens have VIEW / PRINT, and RETENDER is on
  exactly one menu (12, Sales Entry).
- A node with **no `menuVerbs`** (an older row) falls back to the six CRUD verbs.
- Column 0 (the menu itself) is never gated.

### 6.4 Behaviour

- **Menu (col 0) ticked** means the menu is in the user's set. Ticking a parent's col 0
  ticks its children's col 0, and only col 0; unticking does the same. Qt does this,
  keep it.
- **All / None:** every *existing* cell in the whole tree. It must skip cells the verb
  gating removed, or All would grant Re-tender everywhere.
- **Count:** "ticked menus / total".
- Add row-level and column-level helpers (new, cheap):
  - a header click toggles that verb for every row that has it;
  - a row's context menu offers "all verbs on this menu" / "none";
  - a filter box narrows the tree by name.

### 6.5 Applying a loaded user

`/user-administration/get` → `menus[]` → `grants.ts` builds `Map<menuId, Grant>`:
- a row present → col 0 ticked plus each flag;
- a menu with no row → unticked.

**A stale grant on a verb the menu does not have** (for example `umCanRetender: true` on
Tax Master) is **kept in state and sent back unchanged**, but not painted. Qt keeps it the
same way. Do not silently clear it: that is data this screen did not decide.

The tree and the user load in parallel, and the grants apply when both have arrived.
Qt queues them in `m_pending` because its tree arrives async. In React this is simply a
derived value.

### 6.6 Copy rights from another user (new)

A picker of users, then load that user's `menus[]` into the **form** (not saved), then the
admin reviews and saves. This is the practical substitute for roles, and the cure
for granting 218 × 11 boxes by hand.

## 7. Save

### 7.1 One call

The body is the Identity fields (§4, whitelisted) plus `menus` = `toMenusPayload(grants)`.
It holds every menu whose col 0 is ticked, with **all eleven flags** plus `umVisibility: true`.

- **All five posting flags are sent.** Qt's old guard (`kServerHasPostingRights`) was
  for a server that 400'd them. The DTO carries them now. Do not port the flag.
- **Favourite / pinned / sort order:** send back **what the loaded row had** (default
  false / false / 0 for a new row). Qt never sends them. So if a user ever pins a menu, an
  admin's save could reset it, because the replace rewrites the row. Nobody on the box has
  one today (all zero), so nothing is lost yet; React must not introduce the loss.

### 7.2 The full-replace trap (why the form must hold the whole set)

On 2026-09-19 a save made to grant **one** menu rewrote 70 rows, and **four came back
all-false**: Received Cheques, Offers & Schemes, Printing Assignments, Stock Track Policy.
The Received Cheques screen then showed seven dead buttons.

The rule: the form state is always the **complete** set, loaded from `get` before
any edit is possible. **Until both the tree and the user's `menus[]` have loaded, the
Permissions tab is read-only and Save is disabled.** A save from a half-loaded form
revokes everything it has not yet seen.

### 7.3 Say what a save revokes (new, and cheap)

Before sending, `grantDiff(loaded, current)` lists what changes. If it **revokes** anything,
show a confirmation:
*"This save removes access to 4 menus: Received Cheques, …, and removes Post on 2.
Continue?"*
The revocation in §7.2 happened because nothing said it was about to happen.

### 7.4 Errors

A 400 with `errors[]` puts each message on its field (`usrLoginName` …). Anything else goes
to a banner. A duplicate login name answers from the server; show its sentence.

## 8. After a save: permissions go stale

Permissions are read **once at login** and cached for the session. Qt now emits
`SessionEvents::notifyPermissionsChanged(userId)` after a save.

**React:**
- if the saved user is the signed-in user, invalidate the permission query and refetch.
  A grant to yourself should take effect without signing out.
- Any **other** open session of that user stays stale until it signs in again. There is no
  push or refresh route (§10). Say so in the success toast: *"Saved. Arjun gets the new
  rights at his next sign-in."*

## 9. The list (grid 62)

- **Columns:** `usr_id` (hidden) · Login Name · Display Name (hidden) · Full Name ·
  Mobile · Type · Active · Locked · Last Login.
- Through the shared registered-grid list component, with the house grid-runner rules
  (`plan-react-dropdown.md`).
- **Add** opens an empty form. **Edit** (Enter / double-click) opens it by `usrId`.
- **Delete:** confirm *"Delete user "X"? Their menu rights go with them."* →
  `DELETE /user-administration/delete?usrId=…` → refresh.
  - Refuse deleting **yourself** client-side. The server behaviour is unverified (§10).
- Audit screen id 70 (`audit.audit_screen`), for the list's audit trail button.

## 10. Backend gaps this screen inherits

| id | What | Effect |
|---|---|---|
| **REPLACE** | `menus[]` is a full replace; an item left out is soft-deleted | §7.2–7.3 guard it client-side |
| **FAV-RESET** | a save may reset `umIsFavourite / umIsPinned / umSortOrder` if the client does not send them | send them back as loaded (§7.1). Verify once the favourites UI exists |
| **NO-REFRESH** | no route or event pushes changed permissions to a running session | toast says "at next sign-in" (§8) |
| **TYPE-ENUM** | `usrType` has no enum on the DTO and no CHECK on the table | the client's static list is the only guard. Ask for a CHECK or a lookup |
| **UNLOCK** | `usrIsLocked` / `usrFailedLoginCount` exist, but no unlock route | show the lock; ask for `POST /user-administration/unlock` |
| **MUST-CHANGE** | `usrMustChangePassword` is on the DTO; whether login enforces it is unverified | show it only once verified (Q1) |
| **PWD-POLICY** | no password rule server-side (plain text in the DTO, hashed by the service) | none client-side; a policy is a server rule first |
| **SELF-DELETE** | unverified whether the server refuses deleting the signed-in user | refuse client-side (§9) |

**Q1 (for the user):** which of `usrDisplayName`, `usrEmail`, `usrEmployeeId`,
`usrMustChangePassword`, `usrTimezone`, `usrLanguage`, `usrAvatarUrl` to show.

## 11. Tests

**Vitest, `domain/`:**
- `menuTree`:
  - it parses the live node shape (§3);
  - a node without `menuVerbs` gets the six CRUD verbs;
  - RETENDER exists on menu 12 only (a fixture from the box).
- `grants`:
  - applying `menus[]` ticks exactly those rows;
  - a stale grant on a missing verb survives a round trip unchanged;
  - a parent's col 0 cascades to its children's col 0 only.
- `toMenusPayload`:
  - only col-0-ticked menus appear;
  - each carries **all eleven** flags plus visibility plus favourite / pinned / sort **as loaded**;
  - a verb the menu lacks is sent false, unless it was a stale true.
- `grantDiff`: grants, revokes and flag-level changes are counted and named.
- The body builder:
  - no key outside `SaveUserAdministrationDto`;
  - `usrPassword` omitted when blank on edit, present on create;
  - `menus` omitted when the tab never loaded.

**Component (MSW):**
- Save is disabled until the tree **and** the user have loaded;
- a revoking save shows the §7.3 confirmation;
- saving yourself invalidates the permission query;
- a 400 lands on its field.

**Live acceptance (a ZT user, never VKPOS):**
1. Create ZT-USER with three menus; read `public.user_menus` back.
2. Grant one more; the other three are unchanged (this is the 2026-09-19 regression).
3. Revoke one; the confirmation named it.
4. Post / Cancel granted on a posting menu reach `user_menus`.
5. Delete ZT-USER.

## 12. Phases

| Phase | Builds | Done when |
|---|---|---|
| 1 | `domain/*` + fixtures from the live tree and one user | the §11 Vitest suites pass |
| 2 | List (grid 62) + Identity tab + create / edit / delete, no permissions (`menus` omitted) | a ZT user round-trips |
| 3 | Permissions tab: tree, verb gating, All / None, the load gate, full-set save, the revoke confirmation | live acceptance 1–4 |
| 4 | Copy-rights-from, header / row toggles, filter; the §10 asks as they land | live 5; unlock once the route exists |

## 13. Do not port

- **`kServerHasPostingRights`** and its stale comment ("the server does not accept the
  posting five yet"). The DTO carries them; send them.
- **`m_pending` / `m_blocking`.** They are the Qt answer to an async tree and to
  `itemChanged` firing on programmatic writes. React derives the painted state from
  `tree + grants`, with no signals to block.
- **Building `menus[]` from whatever the tree happens to show.** Qt's `fieldValue()`
  walks the widget. React builds it from the grants map, which is complete by construction (§7.2).
- **Dropping `umIsFavourite / umIsPinned / umSortOrder` from the save** (§7.1).
- **The branch dropdown unscoped by company** (§4).
- **Echoing the GET payload into the save.** `forbidNonWhitelisted` 400s it (§3).
- **Sending `menus: []` to mean "unchanged".** It revokes everything; omit the key.
- **A greyed or unchecked box for a verb a menu does not have** (§6.3).
- **A silent save.** Say what it revokes (§7.3) and when the rights take effect (§8).
