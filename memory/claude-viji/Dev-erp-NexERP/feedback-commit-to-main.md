---
name: feedback-commit-to-main
description: The user commits straight to main — when they say commit, commit on main, no feature branch
metadata:
  type: feedback
---

On 2026-10-26 I committed the transfer screens on a new branch (the harness default "branch first when on main"); the user answered "commit to main".

**Why:** this repo's history is daily commits straight on `main` (e.g. "24-10-26", "23-10-26"); a side branch is just an extra step for them.

**How to apply:** when the user asks to commit here, commit on `main` directly. No push unless asked.
