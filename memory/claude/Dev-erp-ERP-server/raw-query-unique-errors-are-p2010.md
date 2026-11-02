---
name: raw-query-unique-errors-are-p2010
description: "A $queryRaw/$executeRaw unique violation is Prisma P2010 with meta.code 23505, not P2002; isUniqueConstraintError covers both since 2026-10-12, but other raw INSERT sites were never audited for an uncaught 500"
metadata:
  node_type: memory
  type: project
  originSessionId: e7d8bb81-5dab-4765-ad1d-61d5d53fdc96
  modified: 2026-10-12T04:52:59.371Z
---

Prisma 6.19 maps a 23505 from an ORM write to `P2002`, but a raw write (`$queryRaw` / `$executeRaw`,
which every register and posting INSERT in this repo uses) surfaces as `PrismaClientKnownRequestError`
code `P2010` with `meta = { code: '23505', message: 'Key (a, b)=(1, x) already exists.' }`. Until
notes 104 (2026-10-12) `isUniqueConstraintError` checked P2002 only, so raw unique clashes were 500s.

**Why:** notes 103/104 are both "an uncaught unique-index error becomes a 500". The expense post's
GSTR-2 register insert was one; `isUniqueConstraintError` and `violatedConstraintOf` in
`src/common/utils/module-shared.utils.ts` now read the P2010 shape too (columns parsed from the
"Key (...)" detail). Only the expense module's create/post were wired to a 409
(`EXPENSE_GST_BILL_DUPLICATE`); the voucher register, sales bill, return and DC register writes
still let a unique clash through as a 500.

**How to apply:** when a raw INSERT can hit a unique index, catch with `isUniqueConstraintError`
and map `violatedConstraintOf(error)` (column list) to a field for a 409. Probe a new Prisma
version's shape with a temp table in a rolled-back `$transaction` before trusting the code mapping.
Related: [[concurrent-claude-sessions]] (a sibling built dist and staged the work minutes after the
edit), [[e2e-suites-steal-default-safe]].
