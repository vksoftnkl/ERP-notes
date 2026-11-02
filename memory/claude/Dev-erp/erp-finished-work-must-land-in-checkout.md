---
name: erp-finished-work-must-land-in-checkout
description: "The user runs/builds from their own checkouts (NexERP main, ERP client dev) and never looks in .claude/worktrees — finished job work must be brought there, or it is \"not applied\""
metadata:
  node_type: memory
  type: feedback
  originSessionId: cfb5e46c-d2f9-46e0-8a3a-5376ed8c408a
  modified: 2026-11-02T04:15:34.303Z
---

Finished work has to end up in the user's own checkout:
- Qt: `/home/vk/Dev/erp/NexERP` on `main`. The user runs it from Qt Creator, and the binary is in `build/Desktop_Qt_6_12_0_Debug`.
- Web: `/home/vk/Dev/erp/ERP client` on `dev`.

A branch left in a worktree counts as "not done" to them.

**Why:**
- On 2026-10-31 they asked "why it always store in work tree, need to write in ERP client/features here too". That day's session fast-forwarded the web `dev`, but hit the usage limit before doing the same for Qt.
- On 2026-11-02 they reported: "ran jobs for qt 2 days back under mainwindow but it wont applied in code". The 10-31 user-menu and Party Outstanding work were still only on branches.

**How to apply:**
- At the end of every job, once tests pass, fast-forward the user's checkout branch to the finished work (`git merge --ff-only`, clean tree only). If several branches touch the same files, integrate them on a branch first.
- In a worktree-isolated session, git commands on the main checkout are refused. Either leave the worktree when the user agrees, or hand them the one command `! git -C <checkout> merge --ff-only <branch>`.
- Tell them to rebuild in Qt Creator; the running binary keeps the old code.
- Do this BEFORE long optional follow-ups, so a usage-limit cutoff can't strand it.
- Related: [[qt-user-menu-react-look-v49]], [[qt-party-outstanding-react-port]], [[erp-workspace-layout-prathap-pc]].
