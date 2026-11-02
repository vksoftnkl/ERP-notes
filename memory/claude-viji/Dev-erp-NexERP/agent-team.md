---
name: agent-team
description: "The 11 NexERP subagents created 2026-10-29 in ~/.claude-viji/agents (one shared rules file); the feature / minor-change / bug flows the main session runs; the trial run still pending"
metadata:
  type: project
---

On 2026-10-29 the user approved ("go") and I created 11 subagents in `/home/vk/.claude-viji/agents/`. They load only in a `claude-viji` session started AFTER that date, from `/home/vk/Dev/erp/NexERP`. `/agents` lists them.

- **Shared rules:** every agent reads `/home/vk/.claude-viji/agent-house-rules.md` first. It covers paths, :3011, Qt 6.12, DB as claude, the memory indexes and the never/always lists. Change shared rules THERE, not in 11 files.
- **Opus agents:** architect (plan + mockup PNG in the share, no code), db-schema (migration + fragment + seed files, dry-run rolled back, applies only on the user's OK), nestjs-backend, qt-screen, react-screen, reviewer (tools Read + Bash, read-only).
- **Sonnet agents:** api-tester (share `test_*_api.py`), qt-tester (offscreen QtTest), web-tester (playwright-core + snap chromium, real VKPOS login), docs-notes (notes-NNN, status file, session-resume, commit message drafts), quick-fix (minor changes; stops at DB / route / posting / >5 files).
- **Flows the main session runs** (subagents cannot start subagents):
  - feature: architect → user "go" → db-schema → nestjs-backend → qt-screen + react-screen → api / qt / web testers → reviewer → docs-notes → user tries it and says "commit";
  - minor change: quick-fix (→ reviewer if it touches posting or several files);
  - bug: quick-fix reproduces; if the cause is in schema or posting, it goes to architect.
- **Settings APPLIED 2026-10-29 by the user** (the classifier refuses Claude editing its own settings, so the user ran the script with `!`). Backup: `~/.claude-viji/settings.json.bak-2026-10-29`. Adds additionalDirectories (ERP server, ERP client, share), memory read rules, `ask` on git commit/push, drops the nexerp-api-ro deny, and widens autoMode trusted repos to all three. Future settings changes: hand the user a script and let them run it with `!`.
- **Agreed next:** a trial run.
  - quick-fix on "grid 195 (menu 295 list) lacks scheme columns" from session-resume. That is really a grid-SQL change, so expect quick-fix to hand it to db-schema;
  - then one small feature through the full flow;
  - then tune the agent files and redraw `share/agents/nexerp_agents_team.png`.

**Why:** the user wants agents to implement features, workflows and minor changes across Qt, NestJS and React from one session.

**How to apply:** pick the agent by the flows above. Give each agent the plan path and the exact scope. Relay its questions to the user verbatim. Never let an agent commit. Related: [[after-move-facts]], [[session-resume]].

**Trial run 2026-10-29 (in progress), lessons for the agent-file tuning pass:**
- quick-fix: needs a hand-back template (current SQL / new SQL / which agent); a new grid output column also needs a fixed.grid_columns row (it wrongly assumed the list is generic).
- db-schema: the share copy for accounts/ledger SQL = numbered file at the share root (last 51); a script-registered grid has no seed row → keep the registration .py in step; drift = "no NEW rows vs a snapshot", not "empty"; modified stamps = `*_modified_by='system', *_modified_on=now()`, sync_on by trigger; test a fragment change with `prisma migrate diff`.
- nestjs-backend: capture the live "before" BEFORE the first edit (nest --watch reloads on save); create/update answer 201; DTO fields are ALWAYS own props after the house pipe → use isPresentField, never hasOwnProperty; read @map before DB checks (cusCollectionDays = sup_collection_days); prettier only on changed files; `git show HEAD:<path> > <path>` to undo own damage; dist/tsconfig.tsbuildinfo is untracked.
- reviewer: needs an "audit mode" (no plan/diff); allow jest + tsc; the wording table is fixed.error_message (erm_code/erm_title/erm_line/erm_todo), source prisma/seed/Error_Messages.json; server menu-rights checks on masters are an open gap, not yet decided.
- architect: mockup only when the layout changes; needs a "master change" plan variant; cloud sync = whole-row last-push-wins, CHECKs still on.
