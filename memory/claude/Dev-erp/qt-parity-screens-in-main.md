---
name: qt-parity-screens-in-main
description: "2026-11-02 — the 10-31 Qt parity screens (Audit Logs 106, Day Book Detail 181, Customer/Item Template 160/161) + the cut-off fix batch A1–C2 finished and fast-forwarded into NexERP main b093677 (unpushed); theme blocks NOT published"
metadata:
  node_type: memory
  type: project
  originSessionId: cfb5e46c-d2f9-46e0-8a3a-5376ed8c408a
  modified: 2026-11-02T04:53:06.556Z
---

**What was done on 2026-11-02:**
- The 10-31 `feature/qt-parity-screens` fix batch had been cut off by the usage limit. A qt-screen agent finished it in three commits:
  - Group A: Audit Logs + Day Book reviewer fixes (`1f30c30`).
  - Group B: Customer/Item Template (`81439f0`).
  - Group C: Promotion Scheme Rs/Qty band + layout ids (`cf2fb84`).
- I then merged main into the branch (`b093677`) and fast-forwarded NexERP `main` to it.
- The branch is pushed. `main` is NOT pushed: 13 ahead of origin on 11-02.

**Verified on the merged code:**
- The build is clean.
- Audit Logs live passed 44/0, Party Outstanding rules and keys 88/0, the new promotion `run-rules.sh` passed, and the user-menu snapshot was fine.
- The agent also ran Day Summary live 61/0, Template live 67/0 with saves, and 48/0 with `TPL_SAVE=0`.

**Still open:**
- **Theme blocks NOT published.** They are copied to `/home/vk/Shared/Prathap/theme/`: `block-audit-logs-and-daybook.qss` and `block-templates.qss`. The originals are in the worktree's `build/share-out/`.
  - Until a template that includes them goes live, the screens use the QssLook token fallbacks (A1).
  - They must be combined with the unpublished v50 Party Outstanding block ([[qt-party-outstanding-react-port]]).
- **`tests/promotion/run.sh` still shows 40 failures.** They are old checks expecting bold captions and blue headings, from before styling moved to the theme on 10-14. The user must decide whether to update or drop them.
- **Template scope stays "always BRANCH".** B3 only changed the wording; the rule itself is the user's decision.
- **Promotion grids have no real ui_table rows.** They now use `NexTable::useLocalLayout()`. Registering them would be a db-schema job.
- **`NexTable::saveColumnWidths()` is now a silent no-op when tableId == 0.** It affects the Loyalty Scheme grids too.
- **Dev data:** the Template save run left two soft-deleted BRANCH rows in `app_setting_value`.

**Why:** the user said "if there are any pending in worktree means apply" ([[erp-finished-work-must-land-in-checkout]]).

**How to apply:** check `git log main` before redoing any of this, and check the live template md5 before publishing a theme.
