# ERP notes and memory: consolidated copy

Snapshot taken 2026-10-31 and last refreshed 2026-11-02 (after the 11-02 PO landing-cost / Qt parity session). It holds 1,125 files (8.9 MB). Every file was copied unchanged with `rsync -a`, so timestamps are kept, and each source has its own folder.

**This is a copy, not the master.**
- The master for plans and notes is `/home/vk/Shared/Prathap`.
- The masters for memory are the Claude memory folders listed below.
- Claude Code does not load memory from this folder. Edits made here do not flow back to the sources.

GitHub: `vksoftnkl/ERP-notes-memory` (private). `notes/share-Prathap/notes.txt` is in `.gitignore` and is never pushed (see "Credentials" below).

## agents/: the 11 custom Claude subagents

`architect`, `db-schema`, `nestjs-backend`, `api-tester`, `qt-screen`, `qt-tester`, `react-screen`, `web-tester`, `reviewer`, `quick-fix`, `docs-notes`. They are copied from `/home/vk/Dev/erp/.claude/agents/`, which is byte-identical to the original in `~/.claude-viji/agents/` (`~/.claude/agents` is a symlink to it). Their shared rules are in `memory/claude-viji/agent-house-rules.md`.

To restore them on a new machine: `mkdir -p <project>/.claude/agents && cp agents/*.md <project>/.claude/agents/` (project level), or copy them to `~/.claude/agents/` (user level).

## memory/: Claude memory

| Folder here | Source | Files |
|---|---|---|
| `claude/Dev-erp` | `~/.claude/projects/-home-vk-Dev-erp/memory` | 15 |
| `claude/Dev-erp-ERP-client` | `~/.claude/projects/-home-vk-Dev-erp-ERP-client/memory` | 135 |
| `claude/Dev-erp-ERP-server` | `~/.claude/projects/-home-vk-Dev-erp-ERP-server/memory` | 115 |
| `claude/Dev-erp-NexERP` | `~/.claude/projects/-home-vk-Dev-erp-NexERP/memory` | 5 |
| `claude/home-vk` | `~/.claude/projects/-home-vk/memory` (the ERP memories saved when Claude ran from `~`) | 32 |
| `claude/ERP-client-mobile` | `~/.claude/projects/-home-vk-ERP-client-mobile/memory` | 3 |
| `claude/VKNexERP-NexERP` | `~/.claude/projects/-home-vk-VKNexERP-NexERP/memory` (prod/dev topology, prod DB rules) | 3 |
| `claude/Dev-vknex-dbsync`, `claude/Dev-vknex-dbsync-nodejs` | dbsync tool memory (ERP DB sync to the VPS) | 3 + 2 |
| `claude/plans` | `~/.claude/plans` (4 ERP plans: unsaved-entry guard, VPS auto-deploy, Qt vs React sale bill, `/bills/amend` TEMP_CR fix) | 4 |
| `claude-viji/Dev-erp-NexERP` | `~/.claude-viji/projects/-home-vk-Dev-erp-NexERP/memory`, Viji's memory: top level (Qt) + `client/` + `server/` + `workspace/` | 510 |
| `claude-viji/agent-house-rules.md` | `~/.claude-viji/agent-house-rules.md` | 1 |

**Overlap.** On 2026-10-30 Viji's memory was merged from the `~/.claude` folders, so most of these files appear twice:

| In Viji's memory | Matches in `~/.claude` | Files only in one copy |
|---|---|---|
| `claude-viji/.../client/` | `claude/Dev-erp-ERP-client` | `erp-qt-131b042-port.md` and `erp-qt-e782e59-user-copy-port.md` are only in Viji's |
| `claude-viji/.../server/` | `claude/Dev-erp-ERP-server` | `grid-sql-no-comments.md` and `boot-seed-undoes-row-deletes.md` (11-02) are only in `claude/` |
| `claude-viji/.../workspace/` | `claude/home-vk` + `claude/Dev-erp` | `erp-server-http-testing-without-credentials.md` and every `claude/Dev-erp` memory written after the merge (`erp-agents-location`, `erp-notes-memory-consolidated-copy`, `erp-mobile-app-react-native-decision`, `user-qt-developer-new-to-react`, `erp-finished-work-must-land-in-checkout`, `qt-party-outstanding-react-port`, `qt-user-menu-react-look-v49`, `qt-parity-screens-in-main`, `po-entry-landing-cost`) are only in `claude/` |

Every file the two copies share is byte-identical. Viji's three other project folders (`-home-vk-Dev-erp`, `-ERP-client`, `-ERP-server`) are symlinks to the NexERP folder, so they were not copied again.

## notes/: plans and notes

| Folder here | Source | Files |
|---|---|---|
| `share-Prathap` | `/home/vk/Shared/Prathap`: every `.md` and `.txt` file, in the same folder layout (130 `notes-NNN` files plus handovers, plans and test results; 11-02 added `purchase/plan-purchase-entry-fixes-2026-11-02.md`) | 291 |
| `other/Documents` | `~/Documents/*.txt`: ERP scratch notes (audit-screen schema sketch, iflag-7 SQL, app-setting resolve URL, Nest log, print-template JSON) | 5 |
| `other/company_master_table_documentation.md` | `~/Downloads` | 1 |

## Not copied

- **From the share**: PNG mockups and diagrams (485), `.py` mockup sources, `.sql` migrations, `.json`/`.qss` grid and theme files, PDFs, `NexERP` (207 MB binary) and `NexERP.zip`. Notes that reference mockup images point to the share; open the images there.
- **Notes already in this repo**: `ERP server/plan` (18), `ERP server/docs` (13), `ERP client/plan` (3), `NexERP/docs` (2), `docs/`.
- **`~/VKNexERP/NexERP`** (`plan.md` and 2 docs): byte-identical to the files in `NexERP/`.
- **`~/Downloads/Pasted text*.txt`**: pasted React code and JSON, not notes.
- **`~/Desktop/api.txt`**: it contains a plaintext GST API password.
- **Non-ERP memory**: testapp3, campaignhub, water-buddy, workspace-mdi, agent-office-kit.

## Credentials inside the copied notes

- `notes/share-Prathap/notes.txt` (the share's scratch pad) holds a GitHub token, the CloudJiffy admin and account passwords, and the `postgresql://webadmin:` connection string. It is kept locally, but `.gitignore` keeps it out of git.
- `notes/share-Prathap/gst/notes-86-gstin-search-params.md` was checked on 2026-11-02. Its `password=` values are already redacted (`***` / `…`), so it is safe to commit.
- `notes/share-Prathap/schema/stock/BUG_2026-09-08_opening_create_audit.md` shows the local dev test login `VKPOS`/`VKPOS`. This is acceptable in a private repo only.

Before each push, run this check. It should print nothing:

```bash
git grep -nE 'ghp_|github_pat_|://[^/ ]+:[^@ /]+@|^Password:' -- . ':!README.md'
```

## Refreshing

Re-run from `/home/vk/Dev/erp`. The commands only add or update files and never delete anything:

```bash
D=notes-and-memory; C=~/.claude/projects
for p in Dev-erp Dev-erp-ERP-client Dev-erp-ERP-server Dev-erp-NexERP ERP-client-mobile VKNexERP-NexERP Dev-vknex-dbsync Dev-vknex-dbsync-nodejs; do
  rsync -a "$C/-home-vk-$p/memory/" "$D/memory/claude/$p/"; done
rsync -a "$C/-home-vk/memory/" "$D/memory/claude/home-vk/"
rsync -a ~/.claude/plans/ "$D/memory/claude/plans/"
rsync -a ~/.claude-viji/projects/-home-vk-Dev-erp-NexERP/memory/ "$D/memory/claude-viji/Dev-erp-NexERP/"
cp -a ~/.claude-viji/agent-house-rules.md "$D/memory/claude-viji/"
rsync -a --prune-empty-dirs --exclude='.git/' --include='*/' --include='*.md' --include='*.txt' --exclude='*' \
  /home/vk/Shared/Prathap/ "$D/notes/share-Prathap/"
rsync -a .claude/agents/ "$D/agents/"
```

Then commit and push from inside `notes-and-memory` (it is its own git repo):

```bash
cd notes-and-memory && git add -A && git status --short | head
git grep -nE 'ghp_|github_pat_|://[^/ ]+:[^@ /]+@|^Password:' -- . ':!README.md'   # must print nothing
git commit -m "Refresh notes and memory" && git push
```
