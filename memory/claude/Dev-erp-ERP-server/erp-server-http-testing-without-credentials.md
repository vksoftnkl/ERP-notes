---
name: erp-server-http-testing-without-credentials
description: Hit authenticated ERP endpoints in a test by overriding TokenService and AuthSessionService providers; tsx cannot boot the Nest graph
metadata: 
  node_type: memory
  type: project
  originSessionId: aecc2b87-d1f3-4731-a1c6-7622bd15b23f
  modified: 2026-09-29T06:03:26.748Z
---

To exercise a guarded route end-to-end without a password, build the testing
module from `AppModule` and `.overrideProvider(TokenService)` (return the claims
you want from `verifyAccessToken`) plus `.overrideProvider(AuthSessionService)`
(no-op `assertAccessTokenIsActive`). Stub at the PROVIDER level, never the
guard, so the real `AccessTokenGuard`, the global `ValidationPipe`, URI
versioning, the `api` prefix and the exception filters still run and the status
codes match the live server. Mirror `src/main.ts` when creating the app or the
route 404s. Working example: `test/opening-cancel-http.e2e-spec.ts`, run with
`npx jest --config ./test/jest-e2e.json`.

**Why:** login needs a real scrypt password nobody has in a dev session, and
Redis sessions expire, so there is otherwise no way to reproduce a reported HTTP
status locally.

**How to apply:** reach for this whenever a bug report names a URL and a status
code. Note these suites drive the REAL database from `.env`, so a happy-path
test mutates live dev rows — target obvious throwaway documents only.

**A one-off route call on a live document** (e.g. cancelling a stuck voucher
the user asked to clear, 2026-09-29 pmt00398) goes through the route, not SQL,
from a spec in the SCRATCHPAD so no sibling session's `jest test/` picks it up.
Its jest config needs `"roots": ["<scratch dir>"]`, `"rootDir": "<repo>/test"`,
`"modulePaths": ["<repo>/node_modules"]`, the transform
`["ts-jest", {"tsconfig": "<repo>/tsconfig.json", "diagnostics": false, "isolatedModules": true}]`
(the tsconfig carries decorator metadata) and the absolute `^src/` mapper. Import
the suite helper by absolute path (`test/helpers/payment-e2e` gives `bootApp`,
`keys`, `PaymentFixtures.grantRights` + `teardown()` for the menu rights).
Never run it while another suite is granting the same user_menus row.

Do NOT try to boot the Nest graph with `tsx`: esbuild does not emit decorator
metadata, so dependency injection fails with a misleading
`Cannot read properties of undefined (reading 'get')` inside a constructor such
as `RedisCacheService`. Use the jest e2e config, or `require()` the compiled
`dist/` build from plain `node`. See [[vscode-stale-prisma-types]] for the other
build-artifact trap in this repo, and [[concurrent-claude-sessions]] before
editing shared files.
