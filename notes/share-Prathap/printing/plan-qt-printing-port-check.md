# Printing port: plan check and completion (rev 2 of the working plan)

**Written:** 2026-09-29. **Checks:** `plan-qt-printing-port.md` (the base plan, left untouched)
against three sources, read line by line:

- the NestJS source `nexerp-api-ro/src/modules/settings/print-*`;
- the React source `nexerp-client/features/{printing,print-designer}`;
- NexERP as it stands today (uncommitted working tree).

**How to read this file:** the base plan still holds except where a row below overrides it. §1 lists
the corrections, §2 the things the base plan did not cover, §3 the updated phase status, §4 the new
backend asks, and §5 the new open decisions. Source paths are short forms:
S = `nexerp-api-ro/src/modules/settings`, W = `nexerp-client/features`.

> **Headline.** The base plan is mostly right. There are three changes that matter:
> 1. **The omitted-key defect is much wider than the pointer** (§1 row 1). Every update DTO in
>    printing nulls whatever it is not sent. The Qt client must echo every key on every update,
>    and **retire / delete revision probably cannot work at all** until the server is fixed.
> 2. **`/print` is single-document.** Batch printing through `/print` does not exist (row 9).
> 3. **The server never looks at the printer.** `ptaPrinterName` is not in any response
>    header, so the client has to call `/resolve` itself (row 11).

---

## 1. Corrections to the base plan

| # | Base plan § | Base plan says | Verified fact | Evidence |
| --- | --- | --- | --- | --- |
| 1 | §0 P0-1, §6 ask 2 | an omitted `ptlPublishedRevId` is read as null | **True, and it applies to every key.** tsconfig is ES2022 with no `useDefineForClassFields:false`, so every DTO field is an own property even when the client omits it, and each `hasOwnProperty(dto,k)` branch takes "present = null". The same thing happens to:<br>• template: `ptlCompanyId` (**a company design becomes shipped/global**), `ptlDescription`, `ptlForkedFromId/Rev`, `ptlIsActive` (goes back to true);<br>• draft version: `ptvFontFamily`, `ptvNote`, `ptvParams` (**prompts wiped**);<br>• dataset: `ptdProviderCode`, `ptdSql`, `ptdParentNo`, `ptdLinkFields`;<br>• assignment: company/branch/device/printer ids and names, and every optional field | S/print-template/print-template.service.ts:873-879, :365-431, :835-845, :1019-1025; S/print-template-assignment/…service.ts:457-485; src/common/utils/module-shared.utils.ts:176 |
| 2 | §3 "a PUBLISHED revision refuses design keys" | refuses design keys | True. But because of row 1, **every** version object "has" all 17 design keys, so a pure `{ptvId, ptvStatus:'RETIRED'}` on a published revision is probably refused too. **Retire is unproven: test it live before building it** | S/print-template/print-template.service.ts:590-617, key list :692-712 |
| 3 | §1 C §4.2 GRID | only `w == cols` matters | Placement uses `col/row`; alignment uses `w`. But **wrapping uses `cols ?? w`** and the schema overflow check uses `col + cols`, so both must be right. Exceptions: **LINE** uses x1/y1/x2/y2 even in GRID, and **RECT** uses w/h as cells | S/print-render/engine/layout/layout.engine.ts:1342, 1351, 1391-1395, ~1172; grid-renderer.base.ts:121 |
| 4 | §1 C §2 order | NO_DATA prints when a DETAIL has 0 rows | Only for the **first** DETAIL with 0 rows. In addition, REPORT_HEADER, PAGE_HEADER, PAGE_FOOTER, SUMMARY, REPORT_FOOTER and NO_DATA may each appear **at most once**, and `groupLevel` is 0–1 | layout.engine.ts:244-251; template-definition.schema.ts ~638-655, :502 |
| 5 | §1 C §6.5 | the web uses a 12-preset font combo | The 12 presets are **sizes** (6…32). The families are NotoSans, NotoSansMono and **NotoSerif**, so the web offers a serif face the server cannot render. The server ships NotoSans (R/B/I/BI), NotoSansMono (R/B, **no italic**) and NotoSansTamil (R/B), and switches Tamil text to NotoSansTamil automatically. The Qt family list stays NotoSans + NotoSansMono; **disable italic for Mono** | W/print-designer/lib/vocabulary.ts:156,159; S/print-render/engine/fonts/font.registry.ts:64-86 |
| 6 | §1 C §8 limits | identifier regex applies (implied: to element ids) | The identifier regex `^[A-Za-z_][A-Za-z0-9_]*$` (≤60) applies to **dataset names and `band.dataset`**. An element `id` is any string of 1–60 characters, and must be **unique across the whole definition**. Limits the base plan missed:<br>• padding 0–50, strokeWidthPt/widthPt 0–20, radiusMm 0–50;<br>• crosstab maxColumns 1–200, mm ±10000, cells 0–2000, heightRows 0–500.<br>Structural rules it missed:<br>• DETAIL/GROUP need a dataset, and GROUP needs groupBy;<br>• a repeating band cannot use a `one` dataset;<br>• CROSSTAB is allowed only in the four singleton bands, and only in GRAPHIC;<br>• GRID needs paper.columns and col/row on every non-LINE/PAGEBREAK element;<br>• GRAPHIC elements must stay inside the page width.<br>Add all of these to the Qt validator | S/print-render/definition/template-definition.schema.ts:131-147, 440-448, 502-533, ~638-828 |
| 7 | §1 C §8 datasets ≤20 | datasets ≤20 | The zod render schema allows ≤20, but the **save DTO allows 100**, so 21–100 datasets save and then fail at render. The Qt validator must cap at **20** | S/print-template/dto/save-print-template-version.dto.ts:29 |
| 8 | §1 C §7 trap 2 | NUMERIC is coerced | Also: **INT8 larger than 2^53 stays a string** | S/print-render/data/value-coercion.ts:74 |
| 9 | §3 "batch ≤50 docIds"; P1 "a batch prints in one render" | batch in one render | `docIds` exists **only on `/preview`**. `/print` takes exactly one `docId`. A batch through `/print` means N calls, one print_log row per document. A batch through `/preview` is one call but writes no print_log | S/print-render/dto/render-document.dto.ts:52; constants.ts:107,120 |
| 10 | §3 inspect | the Data tab can test a query with `inspect:true` | inspect runs the **saved** revision's datasets only, so it cannot test an unsaved query. The per-dataset numbers are from the **last document** only. **Inspect on `/print` still prints and writes print_log**, so only ever use it on `/preview`. The web never uses inspect ("Run it" is a disabled button), so this is a Qt-only feature. The flow is: save the draft, then run inspect | S/print-render/print-render.controller.ts:320-337; W/printing/designer/…/DatasetGrid.tsx:177-183 |
| 11 | P2 "`ptaPrinterName` honoured" | the render response says where to print | The server **never uses the printer**: the printer profile is not passed to the renderers, so codepage and columns take defaults and the cut is always on. `ptaPrinterName` is **not in any header**. It comes back only from `GET /print-template-assignments/resolve` (and from `/print` with inspect). The Qt delivery path is: `/resolve` → the printer name and copies, then `/print` → the bytes | S/print-render/print-render.service.ts:441-445 |
| 12 | §3 headers | `X-Print-Warnings` | It is a **count**, not the text. The texts come only from inspect (`warnings[{kind,message}]`) | S/print-render/print-render.controller.ts:297-303 |
| 13 | §3 content types | pdf or octet-stream | ESCPOS → octet-stream with extension `.bin`; ESCP_DOTMATRIX → `.prn`. Content-Disposition is `inline` for pdf and `attachment` otherwise. **ESCP_DOTMATRIX is reachable only by sending `outputMode` explicitly**; no assignment mode maps to it. GRID→PDF and GRAPHIC→ESCPOS are refused | escpos.renderer.ts:131; escp.renderer.ts:150; print-render.constants.ts:49-79 |
| 14 | §2 layout | `canvas_bridge` maps canvasBridge.ts | also map `domain/context.ts` (CONTEXT_PARAMS, used by the prompts grid), `domain/purposes.ts` and `api/purposes.ts` (the purpose fallback list), `api/templates.ts` and `api/assignments.ts`. "Effective here" is not in ladder.ts: it is `assignments/components/EffectiveCell.tsx` calling `/resolve` | W/printing/… |
| 15 | §1 C §11 | "resize snap can go under 2 mm" | **Wrong.** The reducer re-clamps to `MIN_ELEMENT_MM`; only the transient guide rect goes under 2. Drop it from the fix list | W/print-designer/store/designerSlice.ts:661; lib/geometry.ts:163-176 |
| 16 | §1 C §11 | "colour editor merges undo per band" | **Different bug:** the colour box commits every keystroke with **no** coalesce key. That makes N undo entries per keystroke (via patchEach), and it commits invalid partial colours. Fix: commit on a valid `#rrggbb` or on editingFinished only. Separately, TEXT, BARCODE and QR share one `value-${bandIndex}` coalesce key | W/print-designer/components/panels/controls.tsx:354,364; FontSection.tsx:107-118 |
| 17 | §1 C §11 GRID align | uses heightMm (0) | Understated. It uses **paper mm width** (80), not columns (48), and writes x/y only, **never col/row**, which are what the engine reads. Align in GRID must work in cells and write col/row | W/print-designer/store/designerSlice.ts:963-965 |
| 18 | §1 C §11 Escape | Escape does not cancel a drag | Worse: Escape hides the preview, but pointer-up **still commits** the move | W/print-designer/components/DesignerShell.tsx:290-295; BandBody.tsx:297-316 |
| 19 | §8 print-delivery.ts | "the iframe workaround" | It is the iframe workaround and is **live** (document-preview-dialog imports it). "Do not port" still stands | W/printing/components/document-preview-dialog.tsx:48 |
| 20 | §8 rungLabel() | use the server's labels as the web does | The web uses `ptaScope` / `scope`, but it also derives the rung from the scope id columns (`rungOf`). Port `rungOf`, not the specificity decode | W/printing/domain/ladder.ts:83-91 |
| 21 | P3 "every action on the web's designer" incl. retire, delete revision, inspect | web parity | The web rail has New draft, Publish, Roll back and **Compare**. Retire, delete revision and inspect have **no web UI**; they are Qt additions (and retire is blocked by row 2). **Compare is missing from P3** |W/printing/designer/components/VersionRail.tsx:95-117 |
| 22 | §3 "no needs-a-dataset-to-publish rule" | server has none | True on the server, but **the web enforces it** (`checkPublishable`: an approver plus ≥1 dataset). Qt keeps the web rule | W/printing/domain/draft.ts:395-405 |
| 23 | §4 fonts: "assets/fonts/ … from the API repo" | new assets/fonts dir | NexERP already bundles NotoSans and NotoSansTamil (`src/shared/fonts`, `src/resources.qrc` prefix `/fonts`). Only **NotoSansMono R/B** is missing; add it there, not in a new dir. The fonts are for canvas fidelity only; the server renders the PDF | NexERP src/resources.qrc:2-14 |
| 24 | §0 stub line refs | `txn_doc_types_sales.cpp:96,226,267` | Now `printQuotation` :107, `printSaleOrder` :241, `printSaleBill` :283, the DC lambda :859-864; the TODO comment is at :102. `quotation_entry.cpp:3464` is still right | NexERP |
| 25 | §2 api signature | `postForBytes(path, body, done(bytes, headers, contentType), fail)` | Built as `postForBytes(endpoint, body, ApiBytesSuccess(const ApiBytes&), ApiFailure)`. `ApiBytes{status, body, contentType, headers (lower-cased), header(), isJson()}`. Keep it | NexERP src/core/api/api_client.h:29-41, 89-90 |

## 2. Not covered by the base plan

### 2a. Server behaviour the client must handle

1. **Always send `accYear` on `/print`.** Without it, the datasets bind `:acc_year = null`: the
   service fills the year into print_log only, not into the render context. `/preview` fills it
   correctly. (Server defect: see §4 ask 8.)
2. **The 15 s global timeout.** The global `TimeoutInterceptor` returns **408 after 15 s**, but a render
   may take up to 30 s per document. So set the client timeout above 15 s for render calls and
   treat a 408 as "try fewer documents", not as a network error.
3. **The full render body:**
   - both routes: `companyId, branchId, deviceId, accYear, params, outputMode, copies, filename, inspect`;
   - `/preview` adds `body, docIds`;
   - `/print` adds `purposeId` (required), `srcModule, srcDocType, assignmentOutputMode`
     (PRINT|PREVIEW|PDF|EMAIL|WHATSAPP|ESCPOS) and `isReprint`. `isReprint` is refused when the
     purpose has `ppoAllowReprint = false`, so F11 must read that flag and grey out when it is false.
4. **Prompts** (`ptvParams`):
   - types are TEXT, NUMBER, DATE, DATETIME, BOOLEAN and UUID;
   - an answer to an undeclared prompt is refused (400);
   - a prompt named like a context key overrides the context value, except `company_id`.

   The document print path (options dialog) must ask for the **required prompts**. The web does not,
   so this is a Qt decision (§5 D1).
5. **Providers:** `company.profile`, `branch.profile`, `sales.bill.header`, `sales.bill.items` and
   `sales.bill.tax_summary`. `/providers` returns `{code,label,cardinality}` with **no fields**, so the canvas
   field tree is empty for provider datasets. `company.profile` already carries company_name,
   legal_name, gstin and address_block, and `branch.profile` carries branch_name and the address. This
   is the real replacement for the fake `ctx.companyName` and `ctx.branchName`.
6. **Image sources:** only `data:` URIs and files under the server's allowed roots. http(s) is off. So
   a company logo must be a data URI in a dataset column, and **no provider has a logo column**
   (§4 ask 9).
7. **The full `/resolve` contract:**
   - query: purposeId, companyId, branchId, deviceId, outputMode (company, branch and device default from the token);
   - returns: publishedRevId, printer id/name, printerSource, copies, copyLabels.
8. **Assignment rules:**
   - `ptaCompanyId` must be **present on create**; send an explicit null for "every company", which only a shipped template may take;
   - printer id and printer name are mutually exclusive;
   - a company may not assign another company's private template (fork it first);
   - `ptaCopies` is 1–9, while render copies are 1–10;
   - the list takes `includeGlobal`, `globalOnly` and filters;
   - rows carry server labels (scope, specificity, names, `ptaTemplateIsShipped`, `ptaPrinterProfileName`).
9. **Version rules:**
   - delete revision = `ptvIsDeleted:true`, refused for PUBLISHED or the pointed-at revision;
   - retiring the pointed-at revision **nulls the pointer**;
   - `ptvRevNo` is never reused;
   - ≤20 versions per request;
   - PUBLISHED can only go to RETIRED;
   - one publish per request, and the approver is auto-filled from the token.
10. **Template list/get/delete:**
    - the list filters are search/page/limit, ptlCompanyId, onlyOwned, ptlPurposeId, engine,
      isPublished, ptlIsActive and includeVersions;
    - get takes `includeDeletedVersions`;
    - delete takes `ptlModifiedBy` and is refused while an assignment points at the template.
11. **ESC/POS output** cuts after every page (partial cut plus feed) and ends with ESC @. There is no
    cash-drawer kick; if the till needs one, `RawPrinter` appends it (§5 D4).
12. **Band flags and element kinds the canvas must round-trip:**
    - `printOn`: ALL_PAGES, FIRST_PAGE, LAST_PAGE, NOT_FIRST_PAGE, NOT_LAST_PAGE;
    - band flags: autoGrow, keepTogether, keepWithNext, keepWithLastDetail, visible;
    - element kinds include BARCODE (code128/ean13/ean8/upca/code39/itf14), QRCODE and CROSSTAB.

### 2b. Web features missing from the phases

| Feature | Web source | Goes in |
| --- | --- | --- |
| **Compare revisions** (read-only JSON of a chosen revision against the working body) | printing/designer/components/VersionRail.tsx | P3 |
| **Save details only** (identity save, no `versions` key; Qt already has `SaveIntent::IdentityOnly`) | designer/useDesigner.ts:207 | P3 |
| Publish bar: approver field, "Sign as me", checkPublishable | domain/draft.ts:395 | P3 |
| **Used by** column + jump to Assignments filtered by template | assignments/AssignmentsScreen.tsx:114 | P3 + P4 |
| Assignments **All assignments** tab (paged 20; scope, purpose, output, active-only, search) | assignments/… | P4 |
| Assignment printer source DEFAULT / NAME / PROFILE (PROFILE only when the row already has one), queue name, copies | assignments/… | P4 |
| Shipped vs owned: owner "Shipped — every company", **Ours only** filter, clone re-owned to the session company, every-company scope lists shipped designs only (`templatesForScope`) | templates/…, domain/assignmentForm.ts | P3 + P4 |
| Purpose fallback list built from templates + assignments; picker falls back to a raw id | domain/purposes.ts | P3 |
| Provider-code combo from `/providers` | designer/…/DatasetGrid.tsx | P3 |
| DatasetGrid: link-field and nesting-incoherence findings, number locked once published, move up/down (sort order only) | domain/linkFields.ts, sqlLint.ts | P3 |
| PromptsGrid findings + "Declare them", CONTEXT_PARAMS list | domain/context.ts | P3 |
| **Layout tab keeps the raw JSON editor** for ESCPOS_TEXT and any body the canvas cannot open, with the invalid-JSON / not-an-object warnings. "Layout becomes the canvas launcher" means a button **beside** it, not a replacement | designer/… | P3 |
| Discard-changes confirm when switching revision; the canvas unsaved guard | useDesigner.ts:275 | P3 / P5 |
| Canvas PreviewDialog asks for prompts, output mode and acc year | print-designer/… (C §9.1) | P5 |
| Document preview: auto-print after render, Re-render, filename stem, "N documents" chip; the format chooser stores the choice and returns | components/document-preview-dialog.tsx (M §6) | P1 |
| Keyboard sheet (`?`), Ctrl+G, Ctrl+;, Ctrl+E, nudge keys | C §12 | P5 |
| Web sale screens also open the print dialog from the bill-list modal (`features/sales/testbill/*`). In Qt that means **every** sales txn list, including the **DC** (F6 at txn_doc_types_sales.cpp:859) | — | P1 |

### 2c. More web bugs to fix, not copy (add to C §11)

- Escape during a drag commits the move (row 18).
- On multi-select pointer-down the selection is replaced by the clicked element (the drag "moves only one").
- The colour box keystrokes are uncoalesced, and invalid partial values are committed (row 16).
- GRID align works in paper mm and never writes col/row (row 17).
- TEXT, BARCODE and QR share one coalesce key.
- `normaliseBand` drops empty-string `visible`, `groupBy` and `dataset`.
- The web Pdf fallback (`printableAssignments`) filters by purpose only: not by branch, counter or
  **output mode**. The base plan already filters by branch and device; add output mode.
- The web still points at dropdown 47, and M §6.1 still says 47. Qt already uses 48.

## 3. Phase status (2026-09-29, uncommitted working tree)

| Phase | Done | Left |
| --- | --- | --- |
| **P0** | pointer echo (payload.cpp:199-201, tests updated); Qt PDF installed on 6.10.2, `Pdf PdfWidgets` in find_package and linked, builds clean; `ApiClient::postForBytes` + `ApiBytes` | **full-echo audit** of the payload against row 1. Template, version and dataset already send every key; verify the fork pair and add assignments when they are built. Also left:<br>• add **Svg** to CMake (installed, not linked);<br>• the endpoints header: render (preview/print/providers), assignment create/get/delete + a `list()` builder, remove the stale "create impossible" note, replace `rungLabel()` with `rungOf`;<br>• add **NotoSansMono R/B** to resources.qrc;<br>• `tests/printing/run.sh`: add `-lQt6Pdf -lQt6PdfWidgets` (and Svg) + include dirs, or the interaction test stops linking as soon as any object uses QPdf;<br>• the "done when" test: a raw `/preview` returns PDF bytes + headers |
| P1 | — | as the base plan says, plus:<br>• `/resolve` before `/print` (row 11);<br>• a batch = N `/print` calls, or one `/preview` (row 9, §5 D2);<br>• accYear is always sent;<br>• the prompts step;<br>• F11 is greyed when `ppoAllowReprint` is false;<br>• DC F6 is wired too;<br>• the sale bill `showTenderDialog` must honour `printAfter` (Q_UNUSED today, sale_bill_entry.cpp:822);<br>• the post/save hooks at sale_bill_entry.cpp:6956/7357 and sale_order_entry.cpp:3772 |
| P2 | — | as the base plan says; the printer name comes from `/resolve`. Codepage/columns/cut are the server's defaults (row 11), so the printer-profile settings do nothing yet |
| P3 | the version rail, New draft, Publish and Roll back (writes a new revision and publishes it) already exist | Compare, Save details only, the publish bar, Used by, the purpose fallback, the providers combo, the prompts grid, the DatasetGrid checks and inspect-after-save (row 10). Retire and delete revision **only after** the live test in row 2 |
| P4 | — | as §2b, plus the rules in §2a.8. Menu 248 is not registered yet (only 62 is) |
| P5 | — | base plan + rows 3-7, 15-18 and §2c |
| P6 | — | base plan; add the `tests/printing/run-live.sh` ZT- suite |

## 4. Backend asks (new; add to base §6)

The base asks 1-6 all hold (ask 1 and ask 2 are wider, see below).

1. **Tenant check (wider than base ask 1):**
   - template **list** with no filter returns every company's templates;
   - assignment get/list/create/delete do no token-company check;
   - on `/preview` and `/print`, `companyId` is taken from the **body**, so the `loadVersion` owner check can be passed by naming another company.
2. **Omitted keys (wider than base ask 2):** see §1 row 1. One fix covers it all: `useDefineForClassFields:false`
   in tsconfig, or `dto[k] !== undefined` in place of `hasOwnProperty`. Until then the Qt client
   echoes every key.
3. **Retire a published revision:** confirm `{ptvId, ptvStatus:'RETIRED'}` is accepted. By code
   reading it is refused (row 2).
4. **Cap datasets at 20 in the save DTO** (it says 100; render refuses above 20).
5. **Pass the printer profile to the renderers**, so codepage/columns/cut come from `printer_profile`.
6. **Return the resolved printer name** in an `X-Print-Printer` header on `/print`, so one call is enough.
7. **`docIds` on `/print`**, or confirm that a batch print is N calls.
8. **`/print` must render with the filled `accYear`,** not `request.context` (print-render.service.ts:189 vs :237).
9. **A logo column** on `company.profile` (a data URI, or a path under the image roots).
10. **Inspect on `/print` should not print:** refuse `inspect` there, or skip the log.

## 5. Open decisions (new; add to base §7)

- **D1 Prompts on document print:** ask for the required prompts in the options dialog (recommended),
  or refuse templates that declare required prompts for document purposes.
- **D2 Batch print:** N `/print` calls, one print_log each (recommended, because reprint counting needs it), or one
  `/preview` with no log.
- **D3 Retire/delete revision:** leave out of P3 until ask 3 is answered (recommended), or build it
  against a live test.
- **D4 Cash drawer:** does the till need an ESC/POS drawer kick after a cash bill? If so, `RawPrinter`
  appends it and it becomes a printer setting.

---

## 6. How the web prints a document (read from source 2026-09-29)

Files: `W/printing/components/print-options-dialog.tsx`, `components/document-preview-dialog.tsx`,
`print-delivery.ts`, `domain/printOptions.ts`, `domain/documentPrint.ts`.

1. **The caller.** A sales screen opens `PrintOptionsDialog` with a purpose code
   (`SALE_QUOTATION` | `SALE_ORDER` | `SALE_INVOICE`) and `targets[]`. Each target is
   `{docId, companyId, accYear, filename}`, and there are several when list rows are ticked.
   Example: `sales/salebill/components/sale-bill-entry-view.tsx:2082`. Callers are the quotation,
   sale-order and sale-bill list views/modals and the bill entry's Save & Print.
2. **The five-button dialog** (3.0's, kept for muscle memory). On open it fetches, in parallel:
   - purposes (code → id);
   - `/resolve` **twice**: `outputMode=PREVIEW` and no mode (= PRINT);
   - the assignment list for the purpose (`includeGlobal`);
   - the template list for the purpose (which feeds Format).

   | Button | Which design | Then |
   | --- | --- | --- |
   | Print | the PREVIEW winner, else the PRINT winner | preview + auto-print |
   | Preview | the same design | preview |
   | Format | picks from **every active design drawn** for the purpose (not only assigned ones) | returns to the buttons; the choice overrides the other buttons until the dialog closes |
   | Pdf | the **first assignment**: narrowest rung first, no `/resolve`, not filtered by output mode | preview |
   | Cancel | — | — |
3. **The preview popup** replaces the dialog, rather than stacking on it.
   - It loads the template and picks the revision: the one named → the published one → the newest
     (so an unpublished draft can still be looked at).
   - It calls `/print-render/preview` with `docId` (one document) or `docIds` (several, merged
     into one PDF), plus the document's own `companyId`/`accYear`.
   - It sends **no body, no prompts and no outputMode**.
   - It shows the PDF in an `<iframe>`; Download saves `<stem>.pdf` (for a batch, the stem is the slugified heading).
   - Refusals are shown verbatim, including the field the server named.
4. **Delivery** (`print-delivery.ts`). A hidden iframe, which lives outside React (so the blob survives
   the popup closing), loads the blob and calls `contentWindow.print()`, which opens the OS print dialog.
   The popup then closes itself.

**What the web deliberately does not do:**
- **It never calls `/print`** (per the comment, this was an instruction). So there is no print_log, and no
  copies or ORIGINAL/DUPLICATE labels. `useDocumentPrint` (`/print`) is intact but has no callers.
- **There is no raw path.** A GRID design renders as ESC/POS bytes, which get pushed into the PDF iframe, so
  thermal printing has never worked on the web.
- **There is no prompts step.** A required prompt fails the render; an optional one binds NULL.
- **The printer is always chosen in the OS dialog.** `ptaPrinterName` from `/resolve` is ignored.

**Qt port:** copy steps 1–3 as they are: the buttons, the design picking and the revision rule.
Replace step 4 with `PdfPrinter` (QPdfDocument → QPrinter, straight to the resolved printer, with the
dialog optional) and `RawPrinter` for ESC/POS. `/print` vs `/preview` is base §7 decision 1.

## 7. How the web print designer is built (read from source 2026-09-29)

Two layers: `W/printing/designer` (about 2.9K lines) owns the template, its revisions and its data.
`W/print-designer` (about 10K lines) is the band canvas, which edits the layout only.

### 7a. Templates list (`printing/templates/TemplateListScreen.tsx`)

- Find, clone, open and retire the selected row.
- The published state is loud: a null pointer means an amber row that "resolves for nobody".
- Clone opens the new-template path with the source design pre-filled.
- The purposes come from the rows (a stopgap), and the engines from the OpenAPI enums.

### 7b. Designer screen: three tabs, ONE save (`printing/designer/`)

Every tab fills part of one `POST /print-templates/create`: the template's `ptl*` identity and
`versions[0]` (`ptv*`). Only `useDesigner.ts` builds that request, calling into `domain/`; tabs and
components never post and never decide editability.

| Tab | Contents |
| --- | --- |
| **Template** | **Identity**: code, name, purpose (`PurposePicker`), owner (Shipped or a company), description, active, with its own **Save details only** (no `versions` key). **The page**: paper preset, orientation, size, margins, columns, font family and lang for this revision. **Version rail**: New draft, Publish, Roll back (writes a new revision), Compare (a read-only JSON diff against the working body) |
| **Layout** | the raw `ptvBody` as a JSON text area **and** an "open canvas" button. They are equal editors of one body; the text area stays for hand-written bodies, pastes and anything the canvas cannot parse, with warnings for invalid JSON or a body that is not an object |
| **Data** | **DatasetGrid**: No. (binding, rebinds bands, locked once published) vs Order (arrows, `ptdSortOrder` only), name, role, SQL or provider (datalist from `/providers`), parent + link fields, rows and timeout (measured over the whole band, because the child query runs once). **SqlEditor** + lint. **PromptsGrid**: `ptvParams` (name, label, type, required), cross-checked against the `:params` in the queries, with "Declare them" and `suggestContextParam` (for example, bind `:company_id`, not `:comp_id`) |

Around the tabs:
- **Chips:**
  - editing or reading rev N and its status;
  - "published: rev N", or a red "publishes nothing";
  - "used by N assignments ↗" (a link to Assignments filtered by template);
  - "cloned from X".
- **Publish bar:** "Approved by" plus a **Sign as me** link (never pre-filled).
- **Footer:** Close + Save. On a PUBLISHED revision Save becomes **New draft from this**.
- **Dirty confirm** when switching revision.
- **"New template"** is the same screen, opened blank; the first save creates the template and rev 1.

### 7c. Band canvas (`print-designer/`), hosted

The canvas was written for `/reports/*`, which answers 404. **The printing module hosts it:**
- `LayoutCanvasScreen` provides a `CanvasHost` context (load, save, preview, prompts).
- `domain/canvasBridge.ts` translates. On the way in, paper comes from the version, datasets from the
  `ptd_` rows (SQL datasets become provider `sql.<name>`, with all fields typed `string`; provider
  datasets have no fields), and only the bands come from the body. On save, the whole identity is
  sent, the pointer is defended, and **`datasets` are omitted**.
- A PUBLISHED revision opens fully interactive, but Save is refused.

**The shell** (`DesignerShell.tsx`):

| Area | Components |
| --- | --- |
| Top | `DesignerTopBar` (name, state, save), `DesignerMenuBar`, `DesignerToolbar` (grid size, zoom) |
| Menus | **File**: Save, Preview…, Export JSON, Close (Publish-as-default, Duplicate and Version history are quiet when hosted). **Edit**: cut, copy, paste, duplicate, delete, select all in band. **View**: zoom in/out/actual, grid, snap, show expressions, data panel, properties panel. **Arrange**: align L/C/R/T/M/B, distribute H/V, front, forward, backward, back. **Help**: keyboard shortcuts |
| Left | `DatasetTree`: bound datasets → fields, dragged onto a band with native drag-and-drop. `ReportTree`: bands → elements, sharing the selection both ways |
| Centre | `CanvasViewport`: `CanvasRuler`, a `BandBody` per band (with `BandCaption` and a resize edge), `ElementBox`, `SelectionOverlay` (8 handles), `AlignmentGuides`, marquee, snap (`lib/snap.ts`) |
| Right | `PropertyPanel` → sections Band, Position, Font, Format, Visibility, Kind (per element kind; `KindSection.tsx` is 956 lines). Expression fields use `ExpressionEditor`, with autocomplete over `row.*`, `ctx.*`, `sys.*`, datasets and the 38 transforms |
| Bottom | `StatusBar`: the selection, exact geometry, valid or not, and the problems list (from `lib/validate.ts`) opening upward |
| Overlays | `PreviewDialog`, `ShortcutSheet`, `UnsavedGuard` |

**Model and state:**
- Redux `store/designerSlice.ts` (1,343 lines) holds the definition, selection and interaction mode
  (IDLE, PLACING, DRAGGING, RESIZING, MARQUEE).
- `store/history.ts` provides undo/redo, merging edits that share a key within 400 ms.
- `store/selectors.ts`.

**Vocabulary** (`lib/vocabulary.ts`, all client constants):
- 9 bands: REPORT_HEADER, PAGE_HEADER, GROUP_HEADER, DETAIL, GROUP_FOOTER, SUMMARY, PAGE_FOOTER,
  REPORT_FOOTER, NO_DATA;
- 9 elements: TEXT, FIELD, LINE, RECT, IMAGE, BARCODE (code128/ean13/ean8/upca/code39/itf14), QRCODE,
  PAGEBREAK, CROSSTAB;
- 5 printOn values and the crosstab sorts;
- paper presets (A4/A5/A6/Letter… GRAPHIC, plus thermal/dot-matrix GRID);
- font families and 12 size presets.

**Two modes:** GRAPHIC works in mm. GRID works in character cells: col/row, one glyph per cell, and no point sizes.

**The canvas is approximate on purpose** ("F1: never build a second renderer"):
- text is drawn by the browser;
- barcodes, QR codes and remote images are labelled placeholders at their exact geometry;
- "show values" fills fields with synthetic values from `lib/sample-data.ts`, which is not authoritative.

**`PreviewDialog` is the authoritative view.**
- It goes through the host to `/print-render/preview` with the revision id, plus the unsaved bands on a
  DRAFT.
- It asks for the prompts (`ptvParams` minus company_id), the output mode, the acc year and a document id.
- PDF is shown in a viewer; ESC/POS is shown as a readable dump with the control codes marked.
- With no host, there is no Preview.

### 7d. NexERP against the web designer

| Web | NexERP today | Gap (phase) |
| --- | --- | --- |
| Templates list | `print_template_list.cpp` (menu 62): open, clone, retire | filters, the published column / amber row, "used by" (P3) |
| Designer screen | `print_designer.{cpp,ui}`: 3 tabs, rail, New draft, Publish, Roll back; dropdown 48 | Compare, Save details only, chips, publish bar, DatasetGrid checks + providers combo + move up/down, PromptsGrid, inspect-after-save, preview (P3) |
| Layout tab | raw JSON box | keep it and add "Open canvas" (P3 → P5) |
| Canvas | — | all of it (P5). Qt shape: `QGraphicsScene`/`QGraphicsView` (bands = items, elements = child items), a `QUndoStack` with `mergeWith` keyed per element and field, dock widgets for the trees and the property panel, `QPdfView` for preview. The effort is in the model + the validator (C0), not the scene |

Do **not** port the web canvas's own load/save path (`api/printTemplateApi.ts`, `hooks/useTemplateSave`
and `useTemplateActions` for the non-host branch), `RevisionsDrawer`, `PaperSetupDialog` or `EditLayoutLink`;
see base §8. Paper is read-only in the canvas because the revision owns it.
