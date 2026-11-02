---
name: erp-header-user-menu
description: "Header account menu (2026-10-26) — profile from /user-administration/get, password via partial save with NO current-password check, `?` key sheet, pill name hidden below 1800px header room"
metadata:
  node_type: memory
  type: project
  originSessionId: 2cd01f6a-0c35-40c4-86aa-e4f6eee4f87b
  modified: 2026-10-26T06:33:19.657Z
---

Header redesign of 2026-10-26: `components/layout/user-menu.tsx` (pill + panel), `user-account-dialogs.tsx` (My profile, Change password), `keyboard-shortcuts-dialog.tsx` (`?` anywhere outside a field), `header-select.tsx` (icon + chevron select tiles). Styles live in `erp-header.module.css`, not a separate module: the pill-name container query must sit in the module that declares `container-name: erp-header`.

- Email, display name and last login are not in the session; they come from the user's own `GET /user-administration/get?usrId=` (GET mutation, once per load + on profile open). `usrLastLoginOn` is stamped at every login, so "Last login" is THIS session's sign-in.
- Change password = `POST /user-administration/create {usrId, usrLoginName, usrPassword}`. The server keeps omitted keys, so only the password moves. **No server route verifies the current password**, so the form doesn't ask for it. A real `/auth/change-password` is the open follow-up.
- App version is `NEXT_PUBLIC_APP_VERSION`, set in `next.config.mjs` from package.json (0.1.0).
- The header line is full at common widths: the pill's name shows only with ≥1800px of header room (2560 screens). Year/Size tiles have a min width; company/branch absorb the squeeze (full name in `title`).

**Why:** the mock assumed a roomier bar than the one-line header rule allows ([[erp-global-ui-scale]]).
**How to apply:** adding header controls costs company/branch width first; check 1366/1600/1920 shots.
