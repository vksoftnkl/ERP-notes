---
name: agent-office-monitor
description: "Agent Office live monitor at /home/vk/Dev/erp/agent-office-kit customised 10-29 for the NexERP team: 12 seats (lead + 11 subagents), seat = hook agent_type, http://localhost:4000"
metadata:
  type: project
---

2026-10-29 the user asked to watch all agents in /home/vk/Dev/erp/agent-office-kit (pixel-art office, server.js on :4000 → SSE → public/index.html; not a git repo of its own, untracked in the erp parent).

- Customised: 12 desks (4×3), seats `lead` + the 11 [[agent-team]] ids; hooks/report.js maps `agent_type` → seat, Agent/Task PreToolUse gives the target its task title, SubagentStop = done, Stop = lead done; server replays last state per agent. Originals kept as *.orig.*.
- Hooks are installed by the USER via `! bash /home/vk/Dev/erp/agent-office-kit/install-hooks.sh` (Claude can't edit its own settings).
- Unverified until the first live run: that Claude Code 2.1.312 puts `agent_type` on hook payloads from subagents. If every event lands on Lead, run with OFFICE_DEBUG=1 and read hooks/debug.jsonl.
