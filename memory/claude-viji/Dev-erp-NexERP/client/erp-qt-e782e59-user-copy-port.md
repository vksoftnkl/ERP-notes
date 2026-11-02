---
name: erp-qt-e782e59-user-copy-port
description: User Administration (menu 58) Copy — Qt e782e59 + uncommitted follow-ups, ported to React 2026-10-30 (uncommitted); rules both clients share and the deliberate React differences
metadata:
  type: project
---

**Feature (user, 2026-10-30):** a Copy button on the user list opens a NEW user entry that carries only the source user's rights. The user then asked for "copy allow edit date entry rate too, block super admin". React was asked for "the copy provision" only, with no other list changes.

**Shared rules (Qt = React):**
- **Copied:** exactly `menus`, `usrEditDate`, `usrEditEntry`, `usrEditRate`, from `GET /user-administration/get?usrId=`. Everything else is blank or the New default.
- **Save:** `POST /user-administration/create`, with no `usrId` and the source untouched.
- **SUPER ADMIN:** there is no role id or flag, only the text `usr_type` / `usrType`. Upper-case it, remove spaces and underscores, and compare with "SUPERADMIN".
- **Three guards:** the button gate (CREATE on 58 + a non-SUPER-ADMIN row), the click re-check, and the entry re-check after the GET (which closes the entry). The message is "Cannot copy" / "A SUPER ADMIN's rights cannot be copied. Pick another user." It applies to every caller.
- **Header:** "New user — rights copied from <login>".

**Code:**
- **Qt:** `btnCopy`, `UserMasterList::copySelected` / `isActionAllowed`, `UserAdminEntry::prefillPermissionsFrom` (`kCopied`), `isSuperAdminRole`, `superAdminCopyRefusal`.
- **React:** `features/masters/settings/user-administration/domain/copyUser.ts` (`isSuperAdminRole`, `copyPrefillFromPayload`, `placeCopiedGrants`), `entryModel.ts` (copy/loading|loaded|failed), the dialog prop `copyFrom`, the button in `page.tsx`, plus a `copy` icon in erp-action-icons.

**Deliberate React differences:**
- Save waits for the source read.
- The note sits in the title bar after " · ".
- The dialog's existing "Discard changes?" prompt applies.
- Verbs a menu doesn't have are dropped on the copy.

**Open questions for the user (asked 10-30):**
- (1) The new-user default role is CASHIER in Qt but User in React.
- (2) React's existing "Copy rights from…" picker on the Menu Permissions tab is not in Qt and can still copy a SUPER ADMIN's rights. Block it too?

**Status 10-30:** no browser check yet (web-tester; needs a login). No live save done yet.
