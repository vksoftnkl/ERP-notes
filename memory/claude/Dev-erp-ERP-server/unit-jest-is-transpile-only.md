---
name: unit-jest-is-transpile-only
description: Unit jest runs ts-jest with isolatedModules (no type-check) since 2026-10-23 — type-checking OOM-killed workers; spec type errors now only show in tsc --noEmit
metadata:
  node_type: memory
  type: project
  originSessionId: 1ee7d0b7-0daf-4c92-98a2-9c09edf827f8
  modified: 2026-10-23T07:03:48.428Z
---

Since 2026-10-23 the `jest` block in package.json transforms with
`["ts-jest", { "tsconfig": { "isolatedModules": true } }]` — transpile only, no type-check.
The e2e config (test/jest-e2e.json) still type-checks. The same jest block also runs
`test/unit-logger.setup.ts` (setupFilesAfterEnv): Nest logs ERROR only in unit runs, `TEST_LOGS=1` shows all.

**Why:** with type-checking, every worker loaded the 26 MB Prisma types ([[vscode-stale-prisma-types]])
and sat near 2 GB; 15 workers on the 31 GB box got one SIGKILLed by the kernel OOM killer
("A jest worker process … was terminated by another process: signal=SIGKILL" — first seen on
gst-crypto.service.spec.ts, which passes alone). gst went 78 s → 3 s; the whole unit suite runs in ~17 s.

**How to apply:** a spec with a type error now RUNS (and may pass) under jest — run
`npx tsc --noEmit -p tsconfig.json` (tsconfig includes src/**/*, so specs are checked) when
touching specs or the types they use. That tsc run rewrites the tracked dist/tsconfig.tsbuildinfo;
`git checkout --` it afterwards. A SIGKILLed jest worker is memory, not the spec: check
`journalctl -k | grep -i oom` before debugging the test. See [[pre-existing-test-failures]].
