---
name: nestjs-source-readonly
description: "RETIRED 2026-10-29: nexerp-api-ro snapshot is gone; the real ERP server folder is local on .106 (see after-move-facts / prathap-server-access)"
metadata: 
  node_type: memory
  type: project
  originSessionId: 0b00a13b-6c14-4cbf-9fa7-d864e9a434dd
  modified: 2026-09-20T04:53:02.752Z
---

**RETIRED 2026-10-29.** The switch to .106 is final, and the snapshot is no longer used. Read and edit `/home/vk/Dev/erp/ERP server` directly under the rules in [[prathap-server-access]]: commit only my files, by path. History below.

As of 2026-09-20 the NestJS backend source is available locally at
`/home/vk/Projects/Qt6/nexerp-api-ro/` — an rsync snapshot pulled from
**`vk@192.168.0.106:/home/vk/Dev/erp/ERP server/`**. The path was confirmed by
the user on 2026-09-21; note the SPACE and the lower-case `server`, which is
why `--protect-args` is not optional.

**The rule: read only. Never modify a file in that tree, and never propose a
change as an edit.** It is Prathap's working code, not ours. Anything that looks
wrong goes to the user as a patch or a written note to pass on — never a write.
Enforced by `Edit`/`Write`/`NotebookEdit` deny rules on that path in
`~/.claude/settings.json`; those do NOT cover Bash, so no `sed -i`, no
redirects, no git operations in that tree either.

**The command the user actually runs** (2026-09-21), which is what keeps the
snapshot current — files appeared within minutes of him finishing A2:

```
rsync -az --protect-args --delete \
  --exclude node_modules --exclude dist --exclude .git --exclude .env --exclude '*.log' \
  vk@192.168.0.106:/home/vk/Dev/erp/ERP\ server/ /home/vk/Projects/Qt6/nexerp-api-ro/
```

A first pass once pulled `.env.bak`, `certs/localhost.key` and a 676M
`release/`, all since deleted — so if the tree is ever re-created from scratch,
widen the excludes to `--exclude '.env*' --exclude certs --exclude release
--exclude logs --exclude artifacts`. The user's shorter form is fine for
day-to-day re-syncs of an already-clean tree.

`/home/vk/Projects/ERP/ERP-Server/` is a STALE copy — the user said to ignore it.

This is what finally makes [[stock-engine-not-deployed]] answerable: the `.sql`
posting engine is a spec, NestJS is what actually runs, and the drift between
them can now be read rather than guessed. Same for [[api-endpoint-docs]] —
the real handlers beat inferring the contract from Swagger.
