---
name: erp-all-endpoints-smoke-findings
description: "ERP full-surface smoke test (302 routes): fixed 500-crash on missing id in tender-type/state-code masters; open findings on public routes and a no-required-fields create"
metadata: 
  node_type: memory
  type: project
  originSessionId: 272b4d7c-aa7f-4ad6-a475-2899a9c30282
  modified: 2026-09-09T11:57:28.141Z
---

Ran a whole-surface HTTP smoke test on 2026-09-09 (`test/all-endpoints-smoke.e2e-spec.ts`): boots AppModule, enumerates every route from the express router, hits each with a stubbed token, flags any 5xx. 302 routes; after fixes 0 return 5xx. Also checks auth (401 without token).

**Fixed — 500 crashes on a missing id (4 routes, 2 modules, same bug):** `GET /tender-type-masters/get`, `DELETE /tender-type-masters/delete`, `GET /state-code-masters/get`, `DELETE /state-code-masters/delete` all 500'd with `TypeError: Cannot read properties of undefined (reading 'trim')` when the id query param was omitted. `@Query('ttmTypeId') id: string` is undefined when absent, and the service called `id.trim()` directly. Fixed `parseTenderTypeId` (tender-type-master.service.ts) and `normalizeStateCode` (state-code-master.service.ts) to `(value ?? '').trim()`, so an omitted id is a 400, not a 500.

**Open finding — validation gap:** `POST /sale-loading-charges/create` returns 201 on an EMPTY body — every field in `SaveSaleLoadingChargeDto` is optional, so it creates an all-null/zero row. Not fixed (adding required fields is a client-facing decision). The smoke spec self-cleans the junk rows it creates in afterAll.

**Open finding — unauthenticated GETs (by design?):** 6 GET routes do not 401 without a token: `/health` (expected), and `/users`, `/users/get`, `/user-administration/get`, `/device-list-masters/get`, `/device-list-masters/list`. Likely `@Public` for the login screen (user/device pickers). `GET /users` returning 200 unauthenticated exposes the user list — worth confirming that is intended.

Fixes need `npm run build` + restart of the manual `node dist/src/main` :3011 server. Related: [[erp-server-http-testing-without-credentials]], [[erp-stock-transfer-module-state]].
