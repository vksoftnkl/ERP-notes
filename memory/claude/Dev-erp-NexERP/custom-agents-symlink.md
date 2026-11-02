---
name: custom-agents-symlink
description: The 11 NexERP subagents live in ~/.claude-viji/agents; ~/.claude/agents is a symlink to that folder (2026-10-30)
metadata:
  node_type: memory
  type: reference
  originSessionId: d2a416c9-c420-4888-a853-d6940ff083ec
  modified: 2026-10-30T10:35:53.899Z
---

The user's 11 subagents are architect, db-schema, nestjs-backend, qt-screen, react-screen, reviewer, api-tester, qt-tester, web-tester, docs-notes and quick-fix. They are defined in `~/.claude-viji/agents/*.md`, the claude-viji profile (`CLAUDE_CONFIG_DIR=~/.claude-viji`).

On 2026-10-30, `~/.claude/agents` was made a symlink to that folder, so the default profile sees the same 11. An edit in either path changes the one copy.

For a multi-agent job, use them instead of general-purpose agents: db-schema for migrations, qt-screen for Qt screens, qt-tester for tests, reviewer for the merge check. `agent-office-kit/README.md` maps them to seats in the office view.
