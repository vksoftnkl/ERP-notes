---
name: concurrent-claude-sessions
description: Several Claude sessions edit this repo at once; check file mtimes before touching shared files like stock-voucher.service.ts
metadata: 
  node_type: memory
  type: project
  originSessionId: bbf0fdbd-daff-44d4-8565-255e08d43666
  modified: 2026-09-07T07:47:45.304Z
---

The user runs many Claude Code sessions against `/home/vk/Dev/erp/ERP server`
simultaneously (8+ processes seen on 2026-09-07, across the VSCode and
Antigravity extensions). They routinely land overlapping edits in the same
files, minutes or seconds apart, all uncommitted in the one shared working tree.

**Why:** on 2026-09-07 a session implementing `plan/plan-nestjs-physical-stock.md`
and another doing full-column coverage were both rewriting
`src/modules/stocks/stock-voucher/stock-voucher.service.ts`,
`types/stock-voucher.types.ts` and the shared `dto/save-stock-voucher*.dto.ts`.
Both sets of edits survived only by luck; either side's next Write could have
dropped the other's work silently.

**How to apply:** before a multi-edit task on a heavily shared file — anything in
`stock-voucher/`, `common/`, `app.module.ts` — run
`ls -l --time-style=full-iso <files>` and compare against `date`. An mtime within
the last few minutes that you did not cause means another writer is live. Say so
and ask before proceeding; the user's stated preference in that case was to
isolate in a git worktree and merge later. Note `worktree.baseRef` defaults to
`fresh` (origin/main), which on this repo is ~315 commits behind `DEV-v1` — set
it to `head` in `.claude/settings.local.json`, or create the worktree by hand
with `git worktree add -b <branch> <path> HEAD`. See
[[pre-existing-test-failures]] for the baseline to compare test runs against.

**NEVER run `eslint --fix` (or prettier) over a directory wider than the files
you edited.** This repo's committed code is NOT prettier-clean while its eslint
config treats `prettier/prettier` as an error, so `--fix` rewrites essentially
every file it touches. On 2026-09-18 an `npx eslint --fix src/modules/accountsModule
src/modules/sales` reformatted 68 files, 67 of which another live session had
already modified — their content survived, but their diffs are now buried in
reformatting that cannot be separated out afterwards. Scope the command to the
exact paths you changed: `npx eslint --fix <file> <file>`.

**A pasted `tsc --watch` error list can be stale within minutes.** On 2026-09-25
the user pasted 12 `avhPartyId: string | null` errors stamped 18:51; a sibling
session (the voucher-register work that made `avh_party_id` nullable) had
already narrowed it once in `loadHeaderOrThrow` / `StoredHeader` by 18:53, and
`tsc --noEmit` was clean before any edit of mine. Re-run `npx tsc --noEmit -p
tsconfig.json` FIRST and diff mtimes against the paste's timestamp; only edit
the sites that still fail. See [[vscode-stale-prisma-types]] for the mirror
case where VSCode reports errors tsc does not.

**A sibling's mid-edit file blocks every e2e suite.** `test/jest-e2e.json` runs ts-jest WITH
type-checking over the whole AppModule import graph, so one half-written file elsewhere (seen
2026-09-29: `stock-voucher-posting.helper.ts`) fails every suite with "Test suite failed to run".
Do not touch their file: run with a scratchpad copy of the config whose transform is
`["ts-jest", {"diagnostics": false, "isolatedModules": true}]` (absolute `rootDir` /
`moduleNameMapper`), and keep `npx tsc --noEmit` as the separate type check.

**The shared index can hold a sibling's STAGED work.** On 2026-10-08 the index already carried 52 staged
files (employee master / staff advance / secret-hash) that were not mine, so a plain `git commit` would
have shipped them. Commit only your files through a temporary index: `GIT_INDEX_FILE=<scratch>/idx git
read-tree HEAD`, `update-index --cacheinfo` your blobs (for a file with sibling WIP, `hash-object -w`
HEAD's version plus only your hunks), `write-tree`, `commit-tree -p HEAD`, then `git update-ref
refs/heads/<branch> <new> <old>`. The real index keeps their staging untouched.

**The index can also be emptied, or the work committed, under you.** On 2026-10-09 a sibling session
(vk-61) reset the whole index and committed the party-outstanding report (d79c5b8d) in the ~1 minute
between my `git status` and my `git commit -- <paths>`, which then failed with "pathspec did not match
any file(s) known to git". The files themselves were untouched. Right before committing, re-run
`git log -1` + `git status`, look at `ListAgents` for a busy sibling, and `pgrep -af 'jest|git '` for
its live jobs. Commit with `git add -- <paths> && git commit -- <paths>` (only those paths), never a
plain `git commit` of whatever the index holds.
