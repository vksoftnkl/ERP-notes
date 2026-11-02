---
name: qt-party-outstanding-react-port
description: 2026-10-31 Qt Party Outstanding (menu 279) re-laid out like the Next.js screen + footer actions wired; branch feature/party-outstanding-react-look; template v50 drafted NOT published
metadata:
  node_type: memory
  type: project
  originSessionId: 672ac295-a7dd-457d-b5da-356502871f15
  modified: 2026-11-02T04:15:39.530Z
---

**2026-11-02 update:**
- Merged with the user-menu branch on `qt-2026-10-31-integration` (1b7eb25, pushed). Tests: rules all pass, live 20/20, keys 88/88, wiring 19/19.
- The worktree `party-outstanding-react-look` is now on that integration branch. `feature/party-outstanding-react-look` is unchanged at 54d86d3.
- v50 is still unpublished.
- The 10-31 `feature/qt-parity-screens` fixer was cut off by the usage limit with Groups A/B/C uncommitted, so it is NOT in the integration branch.

**Asked (2026-10-31):** "copy the party outstanding ui from next js application to implement that in qt desktop application". Qt already had the screen (menu 279, built in code), so this is a layout + look port. It was done by agents: qt-screen (port), Explore (wiring), qt-tester (regression suite), reviewer, then fixes.

**Where:**
- NexERP branch `feature/party-outstanding-react-look`, pushed, NOT merged.
- Commits: 8c808f8 (keys test), 0d80d2a (port), 54d86d3 (review fixes).
- Worktree: `NexERP/.claude/worktrees/party-outstanding-react-look`.
- Pictures (web vs Qt): `share/report/party-outstanding-react-ref/`.

**Not done:**
- **Template v50 is NOT published.**
  - `share/theme/block-party-outstanding-v50.qss` + `seed-template-v50.qss` = v49 + block (rstripped md5 316ca741…).
  - Until v50 is live, the new grids paint plain (no tints or pills).
- **Wiring:** Ctrl+R (99 / 100), PDF = Print, Print bill-wise preselects PARTY_OUTSTANDING_BILLWISE_A5, Enter or double-click on a bill opens it via drillTarget (only 55 / 51 can open today — the server sends no branchId/voucherTypeId).
- **Outside the screen folder:** small edits to printing_module / print_options_dialog (optional preferredTemplateCode), PaymentEntry::applyOpenArgs, and the ReceiptEntry no-ref wording.
- **Open user questions:**
  - opening on the session branch (matches the web);
  - Enter in More filters moves to the next field (house rule) vs applying (web).
- **Not tested live:** Ctrl+R into a real payable party; real paper / PDF output; calendar days with dues.

**Why:** the look comes from the theme template only ([[qt-user-menu-react-look-v49]] did the same), so this screen and v50 must ship together.

**How to apply:**
- Run the suites ONE AT A TIME, from the worktree root: run.sh, run-live.sh, run-keys.sh, `SRC=test_party_outstanding_wiring run-keys.sh`. In parallel they collide in the build dir; from another cwd the include paths break.
- Before publishing v50, check live md5 == v49 (5b42b9fc…).
