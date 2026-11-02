---
name: docs-plan-phase-0
description: "Documentation plan (8 phases) — Phase 0 discovery committed on branch docs/phase-0-discovery; where the web, Qt and mobile repos live; Phase 1 blocked on the user's answers"
metadata:
  node_type: memory
  type: project
  originSessionId: 930f9afe-3803-4672-ae91-e475fb17b920
  modified: 2026-10-15T14:31:09.069Z
---

The user's documentation plan has 8 phases: discovery, onboarding, Swagger, DB, architecture,
modules, frontend, ops, then a docs site. Each phase gets its own branch and PR, and work
stops after each phase for a report.

Phase 0 was done on 2026-10-15. `docs/_discovery.md` and `docs/README.md` are committed on the
local branch `docs/phase-0-discovery` (fbbb14d6, cut from DEV-v1 4f1ea5db). It is not pushed and
has no PR yet.

**Why:** later phases build on that inventory. Section 8 of the file holds 13 open questions,
including the PR base branch, PG 18 vs the PG 16 compose file, whether to link or move the 102
module READMEs, and scrubbing committed IPs. Phase 1 waits on the answers.

**How to apply:**
- Read `docs/_discovery.md` on that branch before starting any later phase.
- Work in a git worktree, not the shared checkout, because other sessions edit this tree
  ([[concurrent-claude-sessions]]).
- Facts describe the commit. Other sessions' uncommitted cache and SSE work was marked
  "in flight".

Repos in scope (not obvious from the server repo):
- web: `/home/vk/Dev/erp/ERP client` (vksoftnkl/ERP-frontend, branch dev)
- Qt: `/home/vk/VKNexERP/NexERP` (vksoftnkl/NexERP, main); the deployed build is `/home/vk/NexERP`
- mobile: `/home/vk/ERP client mobile` (Expo, **not a git repo**, no eas.json)
- analytics sidecar: not found
- out of scope: `/home/vk/Dev/vkmobileapp-dev` (a separate Express backend)

Controller scans must grep `@Controller(` in every `.ts` file. Some controllers live in
`*.controllers.ts` or camelCase files (gst provider parts, itemStockBalance, itemBatchStock).
