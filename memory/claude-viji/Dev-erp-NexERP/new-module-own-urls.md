---
name: new-module-own-urls
description: A new report/module plan gets its OWN new routes — never reuse existing URLs, DTOs or DB functions, because those will change
metadata:
  type: feedback
---
When planning a new screen's backend, give it its own route family (e.g. `/api/v1/reports/ledger-statement/*`), including its own picker and ledger-facts routes. Don't point it at `/account-ledger-masters/*`, `/opening-balances/*`, grid/dropdown runners, or existing DB functions.

**Why:** the user said on 2026-09-25: "write into a separate urls dont use any old urls because its gonna change bite later".

**How to apply:** in every backend plan, list the new routes and add a line saying that no existing route is called. Shared decorators and Prisma models are fine. See [[ledger-report-mockup]].
