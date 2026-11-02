---
name: erp-prisma-schema-hand-edit-trap
description: Server prisma/schema.prisma is GENERATED from prisma/<schema>/*.prisma fragments; commit a772c172 hand-edited StockCostRevaluation into it, so any schema build silently drops that model + 3 back-relations
metadata:
  type: project
---

`npm run prisma:schema:build` (also run by prisma:generate, prisma:migrate:*, and
`postinstall`) rebuilds `prisma/schema.prisma` from the per-schema fragment files.
As of 2026-10-28, commit a772c172 (P13 stock revaluation) put `model
StockCostRevaluation` and its back-relations on ItemMaster / BranchMaster /
StockLot ONLY into the generated file — there is no fragment. A rebuild drops all
four, `prisma generate` still succeeds, and the next `migrate dev` would try to
DROP `stock.stock_cost_revaluation`.

**Why:** the build output looks authoritative and validates, so the loss is silent.

**How to apply:** after any schema build, compare model lists:
`diff <(git show HEAD:prisma/schema.prisma | grep "^model " | sort) <(grep "^model " prisma/schema.prisma | sort)`.
If a model vanished, restore HEAD's schema.prisma and splice in only your own
block (done this way 2026-10-28 for PrintShareLink), then `npx prisma generate
--schema prisma/schema.prisma --generator client` WITHOUT the build. The real fix
(fragment files for StockCostRevaluation + back-relations) was left to the user —
re-check whether it has landed before assuming the trap is still live.
