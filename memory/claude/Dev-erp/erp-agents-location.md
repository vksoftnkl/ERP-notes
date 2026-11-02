---
name: erp-agents-location
description: Where the 11 custom ERP subagents live (project copy + ~/.claude-viji original + ~/.claude symlink); user wants them kept permanently
metadata:
  node_type: memory
  type: feedback
  originSessionId: 2bcc8b3c-6890-4461-a962-a8506fe28d80
  modified: 2026-10-31T04:58:29.592Z
---

The 11 custom subagents (architect, db-schema, nestjs-backend, api-tester, qt-screen, qt-tester, react-screen, web-tester, reviewer, quick-fix, docs-notes) exist in two places:
- `/home/vk/Dev/erp/.claude/agents/` — real copies, made 2026-10-31 at the user's request. Project-level files take precedence over user-level ones with the same name.
- `/home/vk/.claude-viji/agents/` — the original. `~/.claude/agents` is a symlink to it. The `claude-viji` alias in ~/.bashrc sets CLAUDE_CONFIG_DIR to ~/.claude-viji.

**Why:** the `/agents` wizard was removed from Claude Code. The user thought the agents were gone and asked for them to be kept "permanently here, do not remove after the tasks".

**How to apply:** never delete or move `.claude/agents/`. When editing an agent, say which copy you changed: the two copies are separate files and do not sync. See [[erp-workspace-layout-prathap-pc]].
