---
name: user-run-commands-with-bang
description: "When the user must run a command (e.g. a Prisma migration), give it with the `!` prefix and verify afterwards"
metadata:
  node_type: memory
  type: feedback
  originSessionId: d2a416c9-c420-4888-a853-d6940ff083ec
  modified: 2026-10-30T10:30:39.052Z
---

When a step needs the user to run a command (my DB role `claude` cannot write `fixed.*`, so server migrations are theirs to apply), give it as `! <command>` and say why. Afterwards, check the effect yourself (e.g. `_prisma_migrations` / the grid row) rather than trusting "ran the cmd".

**Why:** On 2026-10-30 the user twice replied "ran the cmd" for `npm run prisma:migrate:deploy` and nothing had been applied on this machine (no npm log, no `_prisma_migrations` row). Run with the `!` prefix, it worked first time and the output landed in the chat.

**How to apply:** Put the exact `! cd … && …` line in the message, name what success looks like, and verify with a read-only query before going on. Related: [[qt-test-runners-cwd]], [[live-vps-sync-migration-p3009]].
