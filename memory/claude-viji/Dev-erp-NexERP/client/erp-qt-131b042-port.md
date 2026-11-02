---
name: erp-qt-131b042-port
description: "Port of Qt NexERP 131b042 (2026-10-30, printing parity) — only the Qt-born items: menu 62 on grid 'MAIN LIST - PRINT TEMPLATES' and edit-published-in-place; committed as client 9fd1fe1"
metadata:
  type: project
---

Qt `131b042` was mostly Qt catching up WITH React (voucher prints, print log, share link, Ledger 258, Outstanding 279, designer features), so most of it owes React nothing. The two Qt-born items were ported on 2026-10-30 by one agent. Committed 2026-10-30 as client 9fd1fe1: 12 files plus `templates/templateList.{ts,test.ts,module.scss}`.

- **Menu 62:**
  - `features/printing/templates/TemplateListScreen.tsx` is now a `CrudMasterPage` over the registry entry `printTemplateList` ('MAIN LIST - PRINT TEMPLATES', fallback 196, by name), using `useConfiguredGridColumnsOnly`.
  - Pills: owner SHIPPED grey / OWN blue, state PUBLISHED green / NOT PUBLISHED amber, `ptl_is_active` Active / Inactive. They live in `templateList.module.scss`, because `printing.module.scss` pills need the shell's variables.
  - All six `iptl_*` tokens are always sent.
  - Clone fetches the full design first (`useLazyGetPrintingTemplateQuery`, newly exported), then runs the old clone code.
  - Delete is relabelled Retire and refused for a shipped design.
  - The hint bar carries Qt's three sentences.
  - Removed: the `usedBy()` fan-out, client paging, the client-side "not published" filter.
- **Edit published in place:**
  - `EDIT_PUBLISHED_IN_PLACE` + `editsPublishedInPlace()` in `features/printing/domain/draft.ts`; `buildPublishedInPlacePayload` in `buildSavePayload.ts`.
  - `useDesigner` `editable = isEditable || editingPublished`. The canvas save publishes and `router.replace`s to the new revision (it assumes the response carries `ptlPublishedRevId`).
  - Preview is forced to "stored" (`storedOnlyNote` on `CanvasPreview` / `PreviewDialog`).
- **Checks:** tsc clean; eslint clean on the changed files; vitest 371 files / 5947 tests pass. The final grid was verified by curl with the screen's params (28/27/1/3/1/27).
- **Not verified:** the page was never seen in a browser (the user was asked to look); the in-place save was never run against the API.
- **Deliberate differences from Qt:**
  - The purpose filter lists only dropdown purposes.
  - "New draft from this" leaves the footer in this mode (it stays in the revision rail).
  - Dataset numbers stay locked for datasets that came from the published revision.

Not ported, because they are Qt canvas niceties with no React equivalent asked for: crosstab placeholder totals and subtotals, report-tree page labels, the remembered unit, 60-character format rules.
