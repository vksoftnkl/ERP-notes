---
name: project-restructure-progress
description: "Status of ERP client folder structure improvement — what phases are done, what changed"
metadata: 
  node_type: memory
  type: project
  originSessionId: 19e40c87-13b6-490e-9295-36d95728da41
---

Phases 1–3 of the ERP client restructure plan are complete (plan file at /home/vk/.claude/plans/cuddly-mapping-waterfall.md).

**Why:** Reduce technical debt in a 263-file Next.js ERP app — backup files, monster files >5K lines, 12+ utility duplicates.

**What was done:**

Phase 1 (done): Deleted .backup files, stale refactoring docs, empty dirs, orphaned `app/item-master/page.tsx`.

Phase 2 (done) — large file splits:
- `components/library/ui/dynamic-modal-form.tsx` → split into `.types.ts`, `.constants.ts`, `.utils.ts`, `.parts.tsx`
- `components/master/crud-master-page.tsx` → extracted `crud-master-page.types.ts`
- `components/ui/table.tsx` → extracted `table.types.ts`
- `components/layout/erp-header.tsx` → split into `erp-header-menu.tsx`, `erp-header-tab-strip.tsx`
- `features/masters/item/item-master-page.tsx` (5428 lines) → extracted `item-master-page.constants.ts` (122 constants)
- `features/masters/customer/page.tsx` (2764 lines) → extracted `customer-master.constants.ts` (74 constants)
- `features/stocks/physical-stock/page.tsx` (3456 → 2437 lines) → extracted `physical-stock.types.ts`, `physical-stock.constants.ts`, `physical-stock.utils.tsx`
- `features/stocks/opening-stock/page.tsx` (3477 → 3220 lines) → extracted `opening-stock.column-settings.ts`

Phase 3 (done):
- Created `types/index.ts` as global re-export hub (ListMeta, ApiSuccessResponse, ApiError, MasterOption, NormalizedListResponse, LookupDefinition, ItemPrice* types)
- Created 4 service files in `services/`: `auth.service.ts`, `lookup.service.ts`, `grid-columns.service.ts`, `master-crud.service.ts`
- Eliminated 12+ duplicate `getFirstDefinedValue`, `toDisplayValue`, `extractRows`, `toSelectBoolean` definitions in: unit, account-ledger-groups, item-section, godown, city, tender, item-category, area pages + gridColumnsSlice → all now import from `@/features/masters/shared/value-mappers` and `@/features/masters/shared/normalizers`
- Replaced local `COLLECTION_DAY_OPTIONS` in `area/page.tsx` with import from `@/utils/constant`

**Context note:** A previous session (May 16) also completed Phase 4.5 partial work:
- `components/library/` was already renamed to `components/design-system/` — all imports now use `@/components/design-system/`
- Opening-stock PascalCase files renamed to kebab-case: `OpeningStockAuditNotesModal.tsx` → `opening-stock-audit-notes-modal.tsx`, `OpeningStockListModal.tsx` → `opening-stock-list-modal.tsx`, `StockTableRow.tsx` → `stock-table-row.tsx`, `StockToolbar.tsx` → `stock-toolbar.tsx`, `UseOpeningStock.ts` → `use-opening-stock.ts`, `Utils.ts` → `opening-stock.utils.ts`, `Types.ts` merged into `opening-stock.types.ts`

**Next phases (not yet done):**
- Phase 4 remainder: merge `utils/` into `lib/`, add barrel index files for `hooks/`, `store/`, `store/api/`
- See plan file for full details.

**How to apply:** Before touching any of the above files, check they exist and have the expected structure — the extractions changed import paths throughout.
