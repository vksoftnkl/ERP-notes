---
name: sales-e2e-specs-rights-fixture
description: The four sales-chain e2e specs (test/sales/* + bill-posted-http) grant tester1 um_can_post etc. as a fixture and must run --runInBand; nobody on the DB holds um_can_post
metadata: 
  node_type: memory
  type: project
  originSessionId: e9c9b22e-6446-4278-a2b8-28a014a35bbc
  modified: 2026-09-22T11:04:13.241Z
---

On the local ERP DB (2026-09-22) NO user_menus row has um_can_post/cancel/amend/override/retender true (0 of 1,111), so every sales `/post` answers 403 SALES_RIGHT_POST for real users. `test/sales/sales-e2e.harness.ts` inserts rows for tester1 on menus 12/13/182 in beforeAll and deletes them in afterAll.

**Why:** the harness is the only way to exercise the posting chain without an admin granting rights; a concurrent spec that finds the rows "already there" loses them when the first spec's afterAll deletes them.

**How to apply:** run `npx jest --config ./test/jest-e2e.json --runInBand test/bill-posted-http.e2e-spec.ts test/sales/*.e2e-spec.ts`; never run the four in parallel workers. Fixtures are permanent, labelled E2E-<suite>-<tag>, fresh item per run. See [[erp-server-http-testing-without-credentials]].
