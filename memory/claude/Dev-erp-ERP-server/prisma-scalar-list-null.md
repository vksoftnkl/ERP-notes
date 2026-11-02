---
name: prisma-scalar-list-null
description: "Passing null to a Prisma String[] column produces the misleading \"Argument `<relation>` is missing\" validation error"
metadata: 
  node_type: memory
  type: project
  originSessionId: 6273fff3-5a3d-4ccb-80f3-bd057d22abc7
  modified: 2026-08-04T11:34:45.655Z
---

Prisma scalar lists (`String[]`, e.g. the `uuid[]` columns `sbSalesmanId` / `sbLoadmanId` /
`sbPackedId` on `SaleBill`) have **no nullable form**. Passing `null` for one makes the data fail
to match the `*UncheckedCreateInput` variant, Prisma falls back to the checked variant, and the
reported error is the unrelated-looking **"Argument `customer` is missing"** — naming whichever
relation the checked variant requires, never the array field that actually caused it.

**Why:** the DTO layer is the choke point — nullable-field helpers that return `null` for `null`/`''`
are correct for scalars but wrong for array columns.

**How to apply:** when a `.create()`/`.update()` dies on "Argument `<relation>` is missing" while the
FK scalar is clearly present in the dumped data, look for a `String[]` field set to `null`, not at
the relation. Fix it in the DTO transform: `null`/`''` must normalise to `[]`, not `null`
(`toUuidArray` in `save-bill.dto.ts`). See [[sale-bill-acc-year-partitions]] — the same bill-save
path had a second, independent blocker.
