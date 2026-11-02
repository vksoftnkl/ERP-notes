---
name: document-print-built
description: Document print (P1, web parity) coded 2026-09-29 — five-button dialog + server PDF in QPdfView + QPrinter; files, wiring, test, and what the live box has configured
metadata:
  type: project
---

Built 2026-09-29 on the user's go ("do exactly what the web does first"), uncommitted.
It copies the web's print path (see [[print-designer-port-analysis]] and check-file §6):
every button renders through **/print-render/preview**, so there is no print_log and no copy labels, as on the web.

Files (src/modules/admin/printing, flat folder):
- `document_print.*`: PURE rules (itemsOf, findPurposeId via dropdown 48, formatOptions,
  firstAssignment by ptaSpecificity, revisionForPreview, previewRequest docId/docIds);
- `print_options_dialog.*`: the five buttons (Print, Preview, Format, Pdf, Cancel); resolves PREVIEW
  then PRINT; Format's OK returns to the buttons;
- `document_preview_dialog.*`: QPdfView; `printTo(QPrinter&)` has no dialog and rasterises at ≤300 dpi,
  full page;
- `printing_module.*`: `PrintingModule::printDocuments(parent, purposeCode, targets, label, startAtFormat)`.

Endpoints: `PrintAssignments::list()` and the `PrintRender` namespace were added to api_endpoints_printing.h.

Wired:
- txn_doc_types_sales `printFromList`: quotation, order and bill get F6 Print / F7 Format; the DC gets F6;
- SaleBillEntry `printBill()`: Save&Print (only when not auto-post), Post&Print, Amend+print, and F11 (the last
  bill saved or posted on this screen in the session).

Tests: `tests/printing/run.sh` now has a third suite, test_document_print (pure + live on the newest
Acme bill, 37/0). `PRINT_SNAP=<dir>` saves PNGs of both windows. **All 29 test runners that link the
app objects got `-lQt6Pdf -lQt6PdfWidgets`**, which a new runner needs too.

Live box config (not code): SALE_INVOICE → SaleInvoiceA5 (published); SALE_QUOTATION → QUOTE_A4;
SALE_ORDER has 1 design but no assignment (only Format works); DELIVERY_CHALLAN has no design.

Entry screens were wired the same day. QuotationEntry Save&Print prints the saved sqId, using the
response's own company and year. SaleOrderEntry Save&Print and F11 printLastOrder use m_lastOrder,
from the soId/soCompanyId/soAccYear in the create response. The quotation has no F11.

Not done: raw ESC/POS (only Save .bin), /print + print_log, prompts.

**Till Z / X prints (notes 117 §7), 2026-10-14:** templates TillSessionZ80 / TillSessionX80 / TillDayZ80 (JSON_BANDS, paper T80, ptvHeightMm NULL = continuous roll PDF), PROVIDER datasets (till.session.* / till.day.* + company.profile + branch.profile as DETAIL, read via `(company|first).x`), published + assigned company-scope PRINT for Acme. Generator: /home/vk/Shared/Prathap/print/till_z_x_templates.py (--post adds a new published revision). Traps: in a flowLayout band EVERY element (lines, captions too) needs positionType FLOAT or it overlaps collapsed rows; providers send dates as 'DD-MM-YYYY HH:mm' text the `date` transform can't parse → AM/PM built by string indexing (`v[11]+v[12]`) in jexl; branch.profile returns {} unless the request carries the document's branchId → DocumentPrint::Target gained `branchId` (sent only when set). Wired: Cockpit Print Z, X read popup Print, Business Day Print day X / store Z. Probe tests/till/probe_printz.cpp (read-only).
