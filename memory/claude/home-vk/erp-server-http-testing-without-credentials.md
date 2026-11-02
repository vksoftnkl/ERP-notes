---
name: erp-server-http-testing-without-credentials
description: "How to drive ERP server routes end-to-end in-process when no login password is available (stub TokenService + AuthSessionService, not the guard)"
metadata: 
  node_type: memory
  type: project
  originSessionId: 3624e8cc-77ec-4771-a54c-df3f710ba639
  modified: 2026-09-09T10:37:03.238Z
---

ERP server (`~/Dev/erp/ERP server`, NestJS + Prisma, Postgres `ERP` on localhost:5432, HTTPS on :3011) has no documented dev password, and creating a user row with a password hash gets blocked. Working approach (2026-09-08): boot `AppModule` under jest with `overrideProvider(TokenService)` returning fixed claims for a dummy bearer and `overrideProvider(AuthSessionService)` with a no-op `assertAccessTokenIsActive`, then supertest with `Authorization: Bearer <dummy>`. `overrideGuard(AccessTokenGuard)` does NOT work because the guard is registered via `APP_GUARD`.

**Why:** the real guard, request context, validation pipe and exception filters all run, so results match the live server.

**How to apply:** jest config needs `moduleNameMapper` for `^src/(.*)$` and ts-jest with the project tsconfig; use `--forceExit` (app leaves handles open). Preload env at the top of the spec with `import '../src/env.preload';` BEFORE any `src/` import, or `API_VERSION` is undefined and `@Version(API_VERSION)` routes never match `/v1`. Mirror `src/main.ts` global setup on the test app: `ValidationPipe({whitelist,forbidNonWhitelisted,transform,transformOptions:{enableImplicitConversion:true}})`, `enableVersioning({type:URI})`, `setGlobalPrefix('api')`. Raw-SQL params comparing a uuid column need an explicit `::uuid` cast (`WHERE svh_id = $1::uuid`). `stock.stock_ledger` is append-only (trigger), so posted test vouchers cannot be cleaned up; label fixtures (E2E-*). Working example specs: `test/opening-cancel-http.e2e-spec.ts` and `test/physical-stock-http.e2e-spec.ts` (the latter runs a net-zero create→post→cancel lifecycle). Related: [[erp-opening-stock-known-defects]], [[erp-opening-cancel-returns-201-not-200]].
