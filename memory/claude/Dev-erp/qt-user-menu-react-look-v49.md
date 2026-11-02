---
name: qt-user-menu-react-look-v49
description: "2026-10-31 Qt main-window user menu rebuilt like the Next.js one — NexERP branch feature/user-menu-react-look; theme template v49 LIVE on dev (not yet shipped as a migration)"
metadata:
  node_type: memory
  type: project
  originSessionId: 672ac295-a7dd-457d-b5da-356502871f15
  modified: 2026-11-02T04:15:36.662Z
---

**2026-11-02 update:**
- Merged with Party Outstanding on NexERP branch `qt-2026-10-31-integration` (merge 1b7eb25, pushed).
- It builds, and the user-menu snapshot plus all 4 Party Outstanding suites pass.
- The user said "apply it", so NexERP `main` was fast-forwarded to 1b7eb25 on 2026-11-02. Later the same day it moved to b093677 with the parity screens ([[qt-parity-screens-in-main]]). It is NOT pushed. See [[erp-finished-work-must-land-in-checkout]].

On 2026-10-31 the user asked that the Qt main window's top-right user menu look like the Next.js header's. It was built in NexERP branch `feature/user-menu-react-look` (commit 959a74c, pushed; not merged to main). The worktree is `NexERP/.claude/worktrees/user-menu-react-look`.

**What it is:**
- A pill with initials, name and chevron.
- UserMenuPanel drops from it: identity, My profile, Change password, Keyboard shortcuts [?], Switch user (Qt only), Logout, and an app/version footer.
- `?` opens `TxnKeys::showAllKeys`.
- Change password posts only usrId, usrLoginName and usrPassword to /user-administration/create, as React does.

**Published:**
- **v49 LIVE on dev, 2026-10-31 06:59 UTC.** The user said "publish".
- `share/theme/seed-template-v49.qss` = v48 + `block-user-menu-v49.qss`, nothing removed.
- Live was v48 (md5 95af0fae) before the save; the read-back md5 5b42b9fc == the file.

**Not done:**
- **No migration yet:** v49 still has to be shipped as a migration for the live VPS (Qt memory `theme-shipping-workflow`, model `20261030190000_app_theme_template_v48`).
- **The new build needs v49:** under v48 its panel has no card background. Older builds keep v48's avatarIcon/userRole/userArrow rules under v49.
- **Never tested live: an actual password change.** The harness only checks the blank-Save error, so it sends nothing.

**Why:** the look comes from the theme template only ([[erp-workspace-layout-prathap-pc]]; Qt memory `theme-shipping-workflow`): publish v49 on dev with the user's OK, then ship it as a migration.

**How to apply:**
- Before publishing, check live md5 == v48.
- Publish v49 with the merge/rollout of this branch, not before or after.
- Preview: `MW_QSS=<seed> MW_USERMENU=mouse|key MW_ACCOUNT=1 tests/mainwindow/run.sh`.
