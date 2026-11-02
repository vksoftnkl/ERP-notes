---
name: stock-voucher-numbering-lock-is-a-noop
description: stock-voucher-numbering.helper.ts took its serial-number advisory lock inside an UNREFERENCED CTE (never executed, so no lock); FIXED 2026-09-08 in the DEV-v1 working tree with count(pg_advisory_xact_lock(...)) — do not move the call back into a CTE
metadata:
  type: project
---

`lockSlnoScope` in `src/modules/stocks/stock-voucher/stock-voucher-numbering.helper.ts` runs
`WITH advisory_lock AS (SELECT pg_advisory_xact_lock(hashtext(ns), hashtext(key))) SELECT 1::int AS locked`.
A non-data-modifying CTE that nothing references is not evaluated, so **no lock is taken**
(verified 2026-09-08 on PG 18: `pg_locks` shows 0 advisory locks after that statement inside a
transaction; `SELECT count(pg_advisory_xact_lock(...))::int` shows 1).

**Why:** `pg_advisory_xact_lock` returns void, which Prisma's `$queryRaw` cannot deserialise
("Failed to deserialize column of type 'void'"), so the author hid the call in a CTE — and lost it.

**How to apply:** call the function inside an aggregate or a comparison the outer SELECT
actually evaluates, e.g. `SELECT count(pg_advisory_xact_lock(hashtext(${ns}), hashtext(${key})))::int AS locked`
— that is the form `applyItemCost` in stock-voucher-posting.helper.ts uses. **Fixed 2026-09-08 in the
DEV-v1 working tree** (verified with pg_locks and the rolled-back e2e). If a future edit reintroduces a
`WITH ... AS (SELECT pg_advisory_xact_lock(...))` shape, the lock is gone again.
Related: [[stock-engine-ddl-lives-outside-repo]], [[concurrent-claude-sessions]].
