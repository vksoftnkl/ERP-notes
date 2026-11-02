---
name: erp-notes-memory-consolidated-copy
description: "A read-only snapshot of all ERP Claude memory and the share's md/txt notes, taken 2026-10-31 and refreshed 2026-11-02, is at /home/vk/Dev/erp/notes-and-memory; it is not the master and nothing loads it"
metadata:
  node_type: memory
  type: reference
  originSessionId: d853685e-51f6-4d85-a8a0-5a6a5f025324
  modified: 2026-10-31T05:11:39.268Z
---

`/home/vk/Dev/erp/notes-and-memory/` is a snapshot copy, made 2026-10-31 at the user's request and last refreshed 2026-11-02 after the PO landing-cost session: 1,125 files, 8.9 MB.
- `memory/claude/<project>/`: every ERP `~/.claude/projects/*/memory` folder, including `-home-vk` (32 ERP memories) and `~/.claude/plans`.
- `memory/claude-viji/`: Viji's 510-file memory and `agent-house-rules.md`.
- `notes/share-Prathap/`: all 291 `.md`/`.txt` files from `/home/vk/Shared/Prathap`, in the same folder layout.

Its `README.md` lists the sources, the overlap between the `~/.claude` and Viji copies, what was left out and why, and the rsync commands to refresh it.

**Why:** the user wanted one place inside the repo holding every note and memory.

**How to apply:**
- Read from the masters: the share ([[erp-workspace-layout-prathap-pc]]) and the live memory folders. This copy goes stale; refresh it with the README commands when the user asks ("update the last session" meant this). Dry-run first with `rsync -ai --dry-run`, and add `--delete` to the dry run to spot files removed at the source. Then fix the README's date, counts and overlap table.
- It is untracked in the parent repo, which has no remote. Two copied share notes hold credentials (notes-86 GSP password, notes.txt webadmin DB URL), so don't commit or push it without asking.
