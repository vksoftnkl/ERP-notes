---
name: prathap-profile-agents-hooks
description: On 2026-10-30 Prathap's plain-claude profile (~/.claude) was given the 11 agents (symlink) and the Agent Office hooks; the user's own profile is claude-viji
metadata:
  type: project
---

On 2026-10-30 a session started with plain `claude` (`~/.claude`, which is Prathap's config, see [[after-move-facts]]) worked on printing. At the user's request it:
- symlinked `~/.claude/agents` to `~/.claude-viji/agents`, so plain `claude` sees the same 11 agents ([[agent-team]]);
- replaced dead office hooks in `~/.claude/settings.json` that pointed at `~/Downloads/agent-office-kit`, with the six hooks of `~/Dev/erp/agent-office-kit/install-hooks.sh`;
  - backups: `settings.json.bak-office-20261030-160803-pre-swap` (original) and `…-160803`;
- started the office server detached: `setsid nohup node server.js` in agent-office-kit, logging to `server.log` ([[agent-office-monitor]]).

That session's four memory notes were then copied here; the claude-viji folder is the user's memory.

**Why:** the user runs both profiles. Plain `claude` loads none of this memory, which is why it knew nothing of the house rules.

**How to apply:** tell the user to start with `claude-viji` from `/home/vk/Dev/erp/NexERP`. If Prathap's profile should not carry the agents or hooks, undo by removing the symlink and restoring the `-pre-swap` backup.
