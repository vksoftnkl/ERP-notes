---
name: print-designer-port-analysis
description: "2026-09-29 audit of the two React QT_DESKTOP_PLAN.md files (printing + print-designer canvas) vs React source, NestJS print-render/template, and NexERP's existing Qt printing slice"
metadata:
  node_type: memory
  type: project
  originSessionId: d0dff783-1781-4040-a175-d797688f376e
  modified: 2026-09-29T06:01:26.738Z
---

Plans audited: `nexerp-client/features/printing/QT_DESKTOP_PLAN.md` (master) and
`features/print-designer/QT_DESKTOP_PLAN.md` (band canvas). Both were written for a
`~/Downloads/files` "ErpDesktop" prototype that does not exist. The real host is
NexERP, which ALREADY has `src/modules/admin/printing/` (list + designer + payload,
menu 62, tests/printing) that neither plan mentions.

Facts that change the port:
- The server does NOT validate ptvBody against zod at save (only checks that it is a JSON
  object), so the Qt Validate port is the only save-time guard. zod is non-strict, so
  unknown keys are stripped at render and survive in storage.
- Hosted paper, datasets AND layoutMode come from the version (`ptvEngine` JSON_BANDS
  → GRAPHIC, ESCPOS_TEXT → GRID). Only bands + meta are read from the body.
- GRID engine places by col/row, never x/y, and uses w for alignment, so w must equal
  cols.
- Client `ctx.companyName/companyLogo/branchName` and `sys.renderedAt` do not exist
  server-side. Five transform signatures differ (fmtIntl, numToWords, gstSplit, truncate,
  mask). The server has no NotoSerif face.
- A web bug the plan misses: dragging a multi-selection moves only the clicked element.
- VERIFIED LIVE 2026-09-29: omitting `ptlPublishedRevId` on a save UNPUBLISHES
  (the pointer goes to NULL). FIXED the same day: Draft.publishedRevId is parsed and
  echoed on every non-publish save, never on a publish or a create/clone. Tests pass.
- There is no tenant check on print-template get/create/delete. print_log has 0 rows,
  printer_profile has 0 rows, and there are no ESCPOS_TEXT revisions live.
- NexERP ApiClient is callback + JSON-only (needs a bytes+headers path). There is no
  QFuture in src. Qt PDF IS installed on ~/Qt/6.10.2 (2026-09-29; the 6.10.3 kit was removed, build in build/Desktop_Qt_6_10_2-Debug).

See [[printing-schema-design]], [[printing-ui-screens]], [[nestjs-source-readonly]].

**The port is underway (user: "make it work in NexErp with the same features").** Plan:
`/home/vk/Shared/Prathap/printing/plan-qt-printing-port.md`, which overrides the two React plans:
phases P0-P6 with document print first, then raw, designer parity, assignments (menu 248), and
the canvas. P0 done so far: the pointer fix, plus `ApiClient::postForBytes` (a 2xx comes back as
ApiBytes{body, contentType, headers}; errors still decode as JSON). Qt PDF installed + Pdf/PdfWidgets linked in CMake, builds clean (no QPdf* code yet, so --as-needed drops the libs).
Still needed for P0: Svg, the endpoints header, and fonts. Ninja lives at ~/Qt/Tools/Ninja/ninja (not on PATH).

**Plan CHECKED 2026-09-29** against the NestJS source, React and NexERP. Companion file
`share/printing/plan-qt-printing-port-check.md` (base plan untouched): 25 corrections, the missing
web features per phase, P0 status and 10 new backend asks. The three that change the build:
(1) the omitted-key defect is **every DTO key** (ES2022 define semantics): template company goes
global, prompts are wiped, dataset SQL is nulled, and assignments too. So always echo every key;
retire of a published revision is probably refused; test it live first. (2) `/print` takes one
docId, batch = N calls. (3) the server never uses the printer: get ptaPrinterName from `/resolve`.
Also: always send accYear on /print (else :acc_year = null); a 15 s global timeout returns 408.
