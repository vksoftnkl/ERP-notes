---
name: erp-bulk-pdf-batch-render
description: "Printing several documents as one merged PDF — /print-render/preview docIds[], and CrudMasterPage's opt-in multi-select"
metadata: 
  node_type: memory
  type: project
  originSessionId: 12ca79ee-0768-43fd-86e8-582c0dfddfe1
  modified: 2026-09-21T08:22:05.224Z
---

Built 2026-09-21. Tick rows on a list screen, press Print, get ONE multi-page
PDF through the normal five-button dialog (Format/Preview/Pdf all work on a
batch). Live on the Sale Bill list (grid 86); every other master is unchanged.

## Server: `docIds[]` on `/print-render/preview` only

The engine already merged several layout trees into one PDF — that is how
`copies` emits ORIGINAL/DUPLICATE as a single stream (`mergeTrees`). A batch is
the same loop one level out, so there is no new pipeline and no client-side PDF
library.

- `RenderPreviewDto.docIds` (max `MAX_BATCH_DOCS` = 50). **Mutually exclusive
  with `docId`** — the controller's `batchFrom()` refuses both by name rather
  than picking one. Empty list falls back to `context.docId`.
- `renderDefinition` loops `docs` × `copies`. **`runDatasets` MUST stay INSIDE
  the doc loop** and `ctx.docId` must be the per-document id: hoisting either
  renders the first bill N times and still produces a perfectly valid PDF, so
  bytes/page-count assertions cannot catch it. The spec asserts the dataset
  runner's recorded `docId` list instead — that is the only test that bites.
- Timeouts scale: `RENDER_COPY_TIMEOUT_MS * trees.length` inner,
  `RENDER_TIMEOUT_MS * docs.length` outer. A flat 30s fails a batch for being
  large rather than wrong.
- `pagesPerCopy` now means one entry per LAYOUT (docs × copies). Safe because
  `/print` never sends `docIds` — see below.
- Company/year/design are bound ONCE for the whole render, so every document in
  a batch must share them. The CLIENT enforces that.

**`/print-render/print` was deliberately NOT given `docIds`.** Bulk stays on the
preview path, matching what the single-bill button already does (nothing is
written to `print_log` — see [[erp-document-print-button]]). A batched `/print`
would owe one log row per document per copy; that is separate work.

## Client

- `buildDocumentPreviewRequest` takes `docIds: string[]` and sends `docId` for
  one, `docIds` for many — the single-document wire is byte-identical to before.
  `PrintOptionsDialog` takes `targets: DocumentPrintTarget[]` (7 call sites);
  `DocumentPreviewDialog` takes `docIds` and keys its render-once ref on
  `versionId + docIds.join(",")`, or a new selection reuses the old render.
- `print-delivery.ts` untouched, and that is the point: it holds ONE iframe and
  blob per tab and revokes the previous on each `sendToPrinter`, so a client-side
  loop of N prints would revoke each document before the printer finished.
- **`filename` does NOT name the saved file.** The bytes become a blob URL at the
  fetch boundary and a blob carries no `Content-Disposition`, so the preview
  dialog got its own **Download** button using an `<a download>`. Verified:
  saves as `3-bills.pdf`.

## `CrudMasterPage` multi-select — opt-in, three props

`enableMultiSelect` + `onBulkPrintAction(rows)` + `bulkPrintDisabledReason(rows)`.
Off by default, so no other master grows a checkbox.

- Selection is a **column**, built in the shell — `components/ui/table.tsx` needed
  NO change (a column's `header`/`render` are ReactNode).
- It is kept OUT of `renderedColumns` and added to a separate `tableColumns`.
  `renderedColumns` is the DATA columns and feeds the column-filter row and
  Export Excel; prepending there puts a checkbox in both. This is why
  `isMasterFilterDataColumn` and `handleDownloadRows` needed no edit.
- `sortable: false` on it, or the table's global `sortable` puts a dead sort
  arrow over the tick box.
- State is `Map<rowId, row>`, not a Set: ticks must survive paging and
  `renderedRows` is only the current page. `checkedRowList` returns this page's
  rows in LIST order, then off-page ones — insertion order would print in click
  order. Cleared on `listStateResetKey` (company/branch/year/date change).
- The row checkbox needs `stopPropagation()` or every tick also drags the
  single highlighted selection.
- ONE button, two arms: nothing ticked → today's `onPrintAction(selectedRow)`;
  ticked → `Print (3)` → `onBulkPrintAction`.

Verified end to end 2026-09-21 against the live stack: 3 bills → one 3-page PDF,
pages 1/2/3 carrying bil00030/29/28 with different customers and totals, exactly
ONE `POST /print-render/preview`. Related: [[erp-printing-module]],
[[erp-client-ui-browser-verification]].
