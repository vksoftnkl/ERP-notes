---
name: erp-server-query-boolean-false-ignored
description: FIXED server-side (6d385b73) — OptionalQueryBoolean used to ignore `false` (implicit conversion); now reads the raw value, so `?flag=false` works
metadata:
  type: project
---

**UPDATE 2026-09-25: fixed in the server code.** `OptionalQueryBoolean` now
reads the RAW value (`Transform(({obj, key}) => toOptionalBoolean(obj[key]))`,
since commit 6d385b73), so `"false"`/`"0"` arrive as `false`. Read from the
code, not re-probed live. New screens may send `false` (the Ledger Statement's
`includeCancelled=false` / `withBillRefs=false` rely on it); the omit-`false`
rule in `features/printing/api` is now merely harmless.

Found 2026-08-25 while wiring the print-templates filters. `?includeSystem=false`
and `?activeOnly=false` have NO effect on `GET /reports/templates` — the server
answers as if both were `true`. String query filters (`ptDocType`) work fine.

**Cause (server, app-wide):** `main.ts` sets
`transformOptions: { enableImplicitConversion: true }`, so class-transformer
converts the raw `"false"` to a `boolean` from the property's `design:type`
BEFORE the `@Transform` in `OptionalQueryBoolean` runs — and `Boolean("false")`
is `true`. Reproduced with a 15-line `plainToInstance` probe in the server repo:
the transform callback receives `true`, not `"false"`. Every DTO field using
`OptionalQueryBoolean` (`src/common/dto/dtoDecorators.ts`) is affected, not just
templates.

Likely fix: `@Type(() => String)` on those fields (or drop implicit conversion
for query DTOs) so the custom transform sees the string. NOT applied — blast
radius is every query-boolean endpoint and the user has not asked for it.

**Re-verified 2026-08-27 on `/print-templates/list`, and there is NO client-side
encoding that works:** `isPublished=` with `true`, `false`, `0`, `1`, `no`,
`off`, `FALSE` all return the published-only rows. A query boolean can therefore
only mean **true or absent**.

**Client rule that follows:** never serialise a `false` query boolean — OMIT the
key. `features/printing/api/{templates,assignments}.ts` `toParams()` does this
(`if (value === false) continue`). Sending `onlyOwned=false` is not "include the
shipped designs", it is the exact opposite and hides them all — which is how the
new Print Templates list first came up empty. A filter whose FALSE reading is
meaningful (`isPublished=false`, "not published") cannot be expressed at all and
has to be done client-side over the page in hand, and said so on screen.

The client sends the params correctly; nothing to fix on that side. Related:
[[erp-server-500-debugging]], [[erp-print-designer-ui]], [[erp-printing-module]].
