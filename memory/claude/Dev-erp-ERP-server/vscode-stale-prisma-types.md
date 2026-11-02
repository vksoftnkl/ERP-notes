---
name: vscode-stale-prisma-types
description: VSCode reports phantom "property does not exist on ItemMasterInclude"-style Prisma errors after a regenerate; verify with tsc before touching code
metadata:
  type: feedback
---

VSCode's TypeScript server routinely serves **stale diagnostics** for Prisma
types in this repo. Symptoms: errors like `'trackPreset' does not exist in type
'ItemMasterInclude'` or `Property 'itemTrackPresetId' does not exist`, even
though the field is in [[prisma-schema-is-generated]] and in the generated
client.

Cause: `node_modules/.prisma/client/index.d.ts` is ~26 MB. After a
`prisma generate` the TS server keeps the old copy cached instead of reloading
a file that size.

**Why:** Acting on these reports means "fixing" code that is already correct —
adding casts, deleting valid relation includes, or rewriting types to match a
client that no longer exists.

**How to apply:** Before changing anything in response to IDE-reported Prisma
type errors, run `npx tsc --noEmit` (exit 0 = the errors are phantom). A quick
tell: the `relatedInformation` line numbers into `index.d.ts` point at unrelated
models in the current file. The fix is a TS-server restart in VSCode
(*TypeScript: Restart TS Server*), not a code edit. Regenerate the client alone
with `npx prisma generate --schema prisma/schema.prisma --generator client` —
the bare `prisma:generate` script also runs the dbml and docs generators, and
the docs one writes to `../../docs`, outside the repo.

**A long-running `nest start --watch` does the same thing, for your own code.**
`tsconfig.json` sets `incremental: true`, so the watcher carries a cached
program in `dist/tsconfig.tsbuildinfo` / `dist/tsconfig.build.tsbuildinfo`.
After a few hours — especially when files are rewritten by a script rather than
by the editor, which can drop the inotify watch — it reports errors like
`has no exported member named 'X'` or `'y' does not exist in type 'Z'` for
symbols that are demonstrably present in the file. Same rule: `npx tsc --noEmit
-p tsconfig.json` AND `-p tsconfig.build.json`; if both exit 0 the watcher is
lying. The fix is restarting the watch process, not a code edit.

Repo-wide `eslint src/**/*.ts` reports ~1214 pre-existing errors across ~317
files (mostly prettier + `no-unsafe-*` in specs). That is baseline noise; don't
treat it as fallout from the change at hand. Compare with
[[pre-existing-test-failures]].
