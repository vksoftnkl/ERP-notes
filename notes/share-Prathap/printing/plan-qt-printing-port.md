# Printing in NexERP (Qt) — port of the React printing module

**Written:** 2026-09-29. **Target:** `/home/vk/Projects/Qt6/NexERP` (Qt 6.10.2, C++17, CMake).
**Source of truth for behaviour:** the React client, `nexerp-client/features/printing` and
`features/print-designer`, which work against the live API today.

**Companion plans, NOT repeated here** (read them for detail; this file overrides them where
it says so):

- `nexerp-client/features/printing/QT_DESKTOP_PLAN.md`, the master plan (the M-plan). Use it for
  §4 contract, §6 document print, §7 delivery, and §8 screens.
- `nexerp-client/features/print-designer/QT_DESKTOP_PLAN.md`, the band canvas (the C-plan). Use it for
  §3–§8 and §12.

> **The whole job in one line:** the same printing features as the web (print a document from any
> sales screen, design templates on a band canvas, assign them per company/branch/counter), where
> the server renders and NexERP only edits, previews and delivers bytes.

---

## 0. Why this file exists: the two plans aim at the wrong desktop

Both React plans were written for a `~/Downloads/files` "ErpDesktop" prototype, and that
prototype does not exist. NexERP is the host. It already has a printing slice from 2026-08-27/28
that neither plan mentions:

| Existing | What it is | Fate |
| --- | --- | --- |
| `src/modules/admin/printing/print_template_list.*` | list, menu 62 | **extend** to web parity (§5 P3) |
| `src/modules/admin/printing/print_designer.{h,cpp,ui}` | Template/Layout/Data tabs; Layout is a raw JSON text box | **extend**: Layout becomes the canvas launcher |
| `src/modules/admin/printing/print_template_payload.*` | pure save-payload builder with tests | **keep**; it is the Qt `buildSavePayload` |
| `src/core/api/api_endpoints_printing.h` | templates plus read-only assignments | **extend**; the stale "create impossible" note must go |
| `tests/printing/run.sh` | payload and designer tests | keep, extend |

**Fixed 2026-09-29 (P0-1):** the payload never sent `ptlPublishedRevId`. We verified live that the
server treats an absent key as null, so **every Qt save unpublished the design**: publish rev 1,
save a rename, and `ptl_published_rev_id` became NULL. The payload now echoes the loaded pointer
on every non-publish save and never on a publish, as the web client does
(`preservePublishedRevId`). Clone clears it. Tests were updated in `test_print_template_payload.cpp`.

## 1. Overrides of the M-plan and the C-plan

| Plan § | Says | NexERP does instead |
| --- | --- | --- |
| M D2 | Qt 6.8, 6.5 floor | 6.10.2 is installed; CMake floor stays 6.5. Add the modules **Pdf, PdfWidgets** (from the `extensions.qtpdf.6102.<platform>` package; it is NOT under `addons`) and **Svg** |
| M D3, C §9 | `QFuture`/`QPromise` API and `CanvasHost` | NexERP has no QFuture anywhere. Use the house `ApiClient` callbacks (`ApiSuccess`/`ApiFailure`). `CanvasHost::save` and `render` take `std::function` done/fail callbacks |
| M D4 | a `printing_core` static library, Qt Test targets | one executable as today. Pure code stays widget-free and QtCore-only so `tests/printing/*.sh` can link its `.o` files directly (house test pattern) |
| M §3, C §2 | `src/printing/`, `src/printdesigner/`, PascalCase | `src/modules/admin/printing/{core,api,delivery,screens,canvas}` with snake_case files; one `.ui` per screen on `NexDialog`; every popup calls `EnterNavigation::install()` |
| M §4 | JSON-only API | `ApiClient` is JSON-only today. Add **`postForBytes(path, body, done(bytes, headers, contentType), fail)`** that returns the raw body and response headers, and decodes a JSON error body on non-2xx |
| M §4.3 #3 | "ptvBody always a string" | the server now also stringifies an object. We keep sending a string; it is still correct |
| M §8.4 | menus 66–70 | printing menus are **62** Printing Configuration (templates), **248** Printing Assignments (exists, not registered), **231** Print Settings (parent 7) and **32** Sticker Printing (out of scope). 66–70 are other screens |
| C §3.3 | body carries paper/datasets | the server reads **only `bands` + `meta`**. Paper, datasets, **layoutMode** (from `ptvEngine`: JSON_BANDS=GRAPHIC, ESCPOS_TEXT=GRID) and schemaVersion come from the revision. The Paper section is **read-only**. Closes C open question 1 |
| C §3.4 | normaliseBand lossless | the web's normaliseBand (`printing/domain/canvasBridge.ts:144`) actually drops unknown band keys and never fills element defaults. NexERP wraps `QJsonObject` losslessly **and** fills element defaults, so a TEXT without `value` cannot crash the validator |
| C §4.2 GRID | x/y/w/h are cells | the engine places GRID elements by `col`/`row` only, never x/y, but aligns with `w`. So `syncGridGeometry` must keep **`w == cols`**. It is required, not cosmetic |
| C §2 order | bands print in declaration order | true only for REPORT_HEADER/DETAIL/SUMMARY/REPORT_FOOTER. PAGE_HEADER/FOOTER, GROUP (matched to DETAIL by dataset + level) and NO_DATA (only when a DETAIL has 0 rows) are placed by the engine. The canvas still shows array order |
| C §4.4 | selection box has 14% fill | 1 px maroon border, no fill. The 14% fill is the **marquee** |
| C §6.4 | dropped number field → `fmt(…)`, 25 mm | SQL dataset fields are all `string` (no catalogue), so in hosted mode a drop is always 40 mm and plain. Keep the rule for when types arrive |
| C §6.5 Font | size 6–32, family NotoSerif | the web uses a 12-preset combo; the server allows 1–200. **The server has no NotoSerif** (it falls back to NotoSans), so list NotoSans and NotoSansMono only, unless the backend ships a serif face (§6) |
| C §7 | roots `ctx.companyName/companyLogo/branchName`, `sys.renderedAt` | **these do not exist on the server.** Real `ctx`: companyId, branchId, accYear, docId, docType, userId, deviceId, lang, copyNo, copyLabel, copies, params, plus answered prompts. Real `sys`: now, template, templateCode, revNo. The web's default IMAGE source `{{ ctx.companyLogo }}` prints nothing. Complete only what exists |
| C §7 | 38 transforms from `lib/expression.ts` | the list is in `lib/vocabulary.ts:270`. Five signatures differ from the server: `mask(value, visible=4, maskChar='X')` keeps the last N chars; `fmtIntl(value, pattern)`; `numToWords(n, currency, subCurrency)`; `gstSplit(…, interState)`; `truncate(…, ellipsis)`. Port the **server's** signatures |
| C §7 trap 2 | NUMERIC arrives as a string | mostly false: SQL and provider numbers are coerced. It still holds for numbers inside json columns and `::text`. `"0.00"` is truthy and `\|bool` is `Boolean()`. Keep the warning, narrowed |
| C §8 | port `validate.ts` exactly | the **server does not validate the body at save** (it only checks for a JSON object; zod runs at render). So the Qt validator is the only save-time guard, and it must hold **both** rule sets: the web's, plus the server's limits (bands ≤60, elements ≤500/band, datasets ≤20, extra row/col/measure ≤6/4/8, `#rrggbb` or `{{`, identifier regex ≤60, font 1–200, z 0–1000, spacing 0–20, schemaVersion ≤1, expressions ≤4000). PAGEBREAK width: the server uses `w ?? 0` and the web `w ?? 20`; follow the server |
| C §11 | the web's "fix, do not copy" list | add: multi-select drag moves only the clicked element; Escape does not cancel a drag; clicking an element while placing selects it; GRID single-element align uses `heightMm` (0); GRID LINE moves update x1/x2 but not x/y; resize snap can go under 2 mm; expression and colour editors commit per keystroke and merge undo **per band** (use per-element keys) |
| C §13 | parity list | add `shipped-templates` (11), `vocabulary` (12) and `printing/domain/canvasBridge` (35). The total is 153 designer cases plus 35 |
| C §1 z-order | paint by z | agreed. The fixtures have 24 (GST) and 72 (quotation) z-vs-array inversions, so C1's "matches the web canvas" check must expect those to differ |

## 2. Module layout

```
src/modules/admin/printing/
  core/                       QtCore only; linked by tests/printing
    print_template_payload.*  (exists) save payload, parse, history
    print_draft.*             toDesignerDraft / revisionForPreview   (draft.ts, printOptions.ts)
    print_ladder.*            assignment ladder + effective-here      (ladder.ts, assignmentForm.ts)
    print_sql_lint.*          dataset SQL lint + select-list columns  (sqlLint.ts, canvasBridge.selectListColumns)
    print_link_fields.*       parent=child parsing                    (linkFields.ts)
    print_render_request.*    preview / print request bodies           (previewRequest.ts, documentPrint.ts)
    canvas_bridge.*           version <-> definition                  (canvasBridge.ts)
  api/
    (endpoints stay in src/core/api/api_endpoints_printing.h)
    print_render_client.*     postForBytes wrapper + X-Print-* header parse (render.ts)
  delivery/
    pdf_printer.*             QPdfDocument -> QPrinter, silent or with QPrintDialog
    raw_printer.*             ESC/POS / ESC/P bytes to a queue (CUPS raw on Linux, WritePrinter on Windows)
    escpos_text.*             bytes -> monospace preview text
  screens/
    printing_module.*         facade: showPrintOptions(), printNow()
    print_options_dialog.*    the five-button popup (.ui)
    document_preview_window.* QPdfView / text view, Print, Save as, Re-render (.ui)
    print_template_list.*     (moved from ../)
    print_designer.*          (moved) Template / Data / Layout tabs
    print_assignments.*       menu 248 (.ui)
  canvas/                     the C-plan tree, snake_case: model/ document/ scene/ panels/ dialogs/
    designer_window.*         QMainWindow, one per revision
    canvas_host.h             the host interface, with callbacks
tests/printing/               run.sh (+ run-live.sh for the ZT- suite)
assets/fonts/                 NotoSans*, NotoSansMono*, NotoSansTamil* from the API repo
```

## 3. Backend facts the port depends on (verified 2026-09-29)

- **Routes** (all `/api/v1`, bearer token only, no menu-rights guard):
  - print-templates: `create` (create, update, publish, rollback, retire, delete revision: one call), `get`, `list`, `delete`;
  - print-template-assignments: `create`, `get`, `list`, `resolve`, `delete`;
  - print-render: `preview`, `print`, `providers`.
- **Render response:**
  - `application/pdf` or `application/octet-stream` (ESCPOS / ESCP_DOTMATRIX);
  - headers `X-Print-Template-Id/-Version-Id/-Rev-No/-Output-Mode/-Pages/-Copies/-Warnings`, and `/print` adds `X-Print-Log-Ids`, `X-Print-Scope`;
  - errors are JSON `{success:false,message,errors:[{field,message}]}`;
  - `inspect:true` returns JSON (pages, per-dataset rowCount, durationMs), which the **Data tab can use to test a query**;
  - copies 1–10, batch ≤50 `docIds`.
- **Preview of unsaved changes** works only on a DRAFT, and paper and datasets still come from the revision.
- **Save:**
  - `datasets` present replaces the set; absent leaves it alone;
  - a PUBLISHED revision refuses design keys;
  - pointer and publish are never sent in one request;
  - there is no concurrency check (last write wins);
  - there is **no** "needs a dataset to publish" rule on the server.
- **Live data:** 18 templates (12 published), 29 revisions, 88 datasets, 9 assignments. `printer_profile` and `print_log` are **empty**; nothing has ever gone through `/print`. There are no ESCPOS_TEXT revisions yet.

## 4. House rules this port must follow

- Every screen is its own `.ui` on `NexDialog`; popups call `EnterNavigation::install()`.
- Grids use `NexTable` with delegate `cellEdited` (never `itemChanged`); connect to user-intent signals only.
- Purposes come from **dropdown 48** (desktop). Never use 47, the web one.
- Always pass the document row's company and acc year; branch and device come from the token.
- Build to check it compiles; run the live suites at the end of a phase, not after every edit.

## 5. Phases

Document print ships first. It is the most-used path, and every sale-bill "Print" is a stub today
(`txn_doc_types_sales.cpp:102`, `quotation_entry.cpp:3464`).

| Phase | Scope | Done when |
| --- | --- | --- |
| **P0 Foundation** | ✅ pointer fix. Install and link Qt PDF, PdfWidgets, Svg. `ApiClient::postForBytes`. Complete the endpoints header (render, assignments CRUD, providers). Bundle fonts | a raw `/print-render/preview` returns PDF bytes plus headers into a test |
| **P1 Document print** | `PrintingModule` facade, `PrintOptionsDialog` (M §6.2, **Pdf fallback filtered by session branch and device**), `DocumentPreviewWindow` (M §6.3), `PdfPrinter`, Save as. Wire quotation / sale order / sale bill lists, F8 lists, bill Save & Print (F6), F11 reprint | a bill prints from the list and after save; a batch prints in one render |
| **P2 Raw delivery** | `RawPrinter`, `EscPosText` preview, `ptaPrinterName` honoured, copies (M §7) | a T80 receipt prints raw on a thermal queue |
| **P3 Templates and designer parity** | list filters, clone, delete, published chip. Designer: version rail with publish, rollback, retire, delete revision; prompts grid; Data tab datasets grid, SQL editor + lint + **inspect** test; Layout tab opens the canvas; Preview from the designer | every action on the web's designer works; a published revision is read-only |
| **P4 Assignments** (menu 248) | branch/counter/output selectors, purpose × scope rows, "Effective here", editor with the ladder rules (M §8.3) | create, edit and delete an assignment at each of the 4 rungs |
| **P5 Canvas** | the C-plan's C0–C5 with §1 applied. C0 (model + validator) goes first because it is the only save-time guard | the three fixtures load, validate as the web does, round-trip, and are editable; preview of the unsaved draft works |
| **P6 Hardening** | live ZT- suite, settings persistence, close guards, import/export bands | full live suite green |

## 6. Backend asks (for Prathap, not edits)

1. **Tenant check:** `print-templates` get, create and delete do no company check, so any logged-in user can read, edit or delete another company's template by id.
2. **Omitted DTO keys:** `ptlPublishedRevId` omitted is treated as null (the unpublish defect above). Either make "absent = leave alone" true (drop `useDefineForClassFields` or check for `!== undefined`), or document that the key must always be echoed.
3. **Validate at save:** run the zod definition schema on `ptvBody` in `create`, not only at render, so a broken layout cannot be published.
4. **Render context:** either add `ctx.companyName`, `ctx.companyLogo`, `ctx.branchName` and `sys.renderedAt`, or confirm that clients should drop them.
5. **Fonts:** ship a NotoSerif face, or confirm that serif is not offered.
6. **Nice to have:** a field catalogue for provider datasets; a read route for `print_log`; printer-profile CRUD.

## 7. Open decisions (owner)

1. **Print button:** `/print` (writes `print_log`, enables reprint counting) or `/preview` (what the web does, so `print_log` stays empty). Recommended: `/print` for Print, `/preview` for Preview and Pdf.
2. **Lint blocks Save** (C §15.2): keep the web behaviour (block) until decided.
3. **Undo back to saved = clean?** Recommended: yes, via `QUndoStack::cleanChanged`.
4. **Windows build:** the Qt PDF package must be installed per kit (`extensions.qtpdf.6102.win64_*`), and `windeployqt` picks it up.

## 8. Do not port

- `features/print-designer/api/printTemplateApi.ts` (every route is dead `/reports/templates/*`), `list/page.tsx`, `new-page.tsx`, `designer-page.tsx`, `PaperSetupDialog`, `RevisionsDrawer`, `EditLayoutLink`, `createStarterDefinition`.
- `print-delivery.ts` (the iframe workaround; `PdfPrinter` owns its bytes).
- `hooks/useDocumentPrint.ts` (unused on the web).
- The old `rungLabel()` hand-decoding of `pta_specificity` in `api_endpoints_printing.h`; use the server's labels as the web does.
