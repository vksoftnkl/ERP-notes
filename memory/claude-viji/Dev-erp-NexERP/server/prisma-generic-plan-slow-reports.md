---
name: prisma-generic-plan-slow-reports
description: "Prisma prepares every $queryRaw; after 5 runs PG may pick a generic plan and `$x IS NULL OR …` / `= ANY($ids)` report SQL goes from ~20 ms to ~3 s"
metadata:
  node_type: memory
  type: project
  originSessionId: a9c46fef-191b-41d7-9a42-b241bd58e91a
  modified: 2026-10-12T05:37:07.199Z
---

Measured 2026-10-12 on the Account Day Summary ([[account-day-summary-module]]):
the money-figures query ran in ~20 ms the first few times, then ~3 s on every
call once the same statement had run five-plus times in the process. Cause:
Prisma sends `$queryRaw` as a prepared statement, Postgres switches to a
generic plan after five executions, and a generic plan cannot see through
optional-filter predicates like `(${branchId}::uuid IS NULL OR v.av_branch_id = …)`.

Fix used: inside the report's own transaction, `SET LOCAL plan_cache_mode =
force_custom_plan` (allowed in a READ ONLY transaction, scoped to it).

**Why:** a one-shot probe or the first e2e test never shows it — only a suite
or a live screen hitting the route repeatedly does, so it looks like box
contention.

**How to apply:** when a raw-SQL report is fast alone but slow in a long e2e
run or in production, suspect this first. The Ledger Statement
(reports/ledger-statement) uses the same predicate pattern WITHOUT the fix and
probably degrades the same way — not changed as of 2026-10-12.
