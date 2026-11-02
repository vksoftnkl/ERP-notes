---
name: erp-opening-cancel-returns-201-not-200
description: "ERP POST /api/v1/stock/opening/cancel (and likely other stock voucher @Post routes) returns HTTP 201, not the 200 its Swagger @ApiOkResponse documents"
metadata: 
  node_type: memory
  type: project
  originSessionId: 272b4d7c-aa7f-4ad6-a475-2899a9c30282
  modified: 2026-09-09T05:23:07.567Z
---

`POST /api/v1/stock/opening/cancel` on the ERP server returns **201 Created** on success, even though the controller's `@ApiOkResponse` documents it as 200 OK. Cause: the handler has no `@HttpCode(200)`, so NestJS's `@Post` default (201) wins. The response body is correct (`{success, message, data:{status:'CANCELLED', rowsReversed}}`); only the status code disagrees with the docs. Verified 2026-09-09 via `test/opening-cancel-http.e2e-spec.ts`.

**Why:** any client or test asserting 200 on this route (and probably `post`/other stock-voucher @Post actions) will wrongly treat a real success as a failure.

**How to apply:** either add `@HttpCode(HttpStatus.OK)` to the handler to match the docs, or fix the Swagger decorator to `@ApiCreatedResponse`. When writing client/test code against these routes, expect 201. Related: [[erp-server-http-testing-without-credentials]].
