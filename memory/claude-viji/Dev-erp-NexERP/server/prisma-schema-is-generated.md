---
name: prisma-schema-is-generated
description: prisma/schema.prisma is generated from prisma/fixed fragments — hand edits get overwritten; user sometimes edits applied migrations and re-syncs checksums
metadata: 
  node_type: memory
  type: project
  originSessionId: 6e166879-6b51-46ab-8412-0c1cd0b06010
---

In the ERP server repo, `prisma/schema.prisma` is built from the fragment files under `prisma/fixed/` (and other dirs) by `scripts/build-prisma-schema.js` (`npm run prisma:schema:build`, also runs on postinstall and before every prisma:* script). Hand edits to `schema.prisma` are silently overwritten — the fragments are the source of truth. The user has hand-edited `schema.prisma` directly before (June 2026 grid_columns UUID rename), causing fragment/schema/client three-way drift.

Other conventions observed: Prisma model fields are camelCase with `@map()` to snake_case DB columns; API JSON keys are snake_case. The user also hand-edits already-applied migrations and repairs them with `npm run prisma:migrate:sync-checksums`.

**Why:** Diagnosing compile errors against generated Prisma types requires checking fragment vs schema.prisma vs node_modules/.prisma client — any of the three can be stale here.

**How to apply:** When Prisma types don't match code, compare `prisma/fixed/*.prisma`, `prisma/schema.prisma`, and the generated client before editing code; fix the fragment, rebuild, regenerate. Never hand-edit `schema.prisma`.
