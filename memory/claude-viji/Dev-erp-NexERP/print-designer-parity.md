---
name: print-designer-parity
description: Print Designer (menu 62) web-parity pass coded 2026-09-29 — what was added, the two real bugs fixed in parse/build, the tests, and what is still left (retire/delete revision, canvas)
metadata:
  type: project
---

Coded 2026-09-29, uncommitted. Five printing suites, 180 PASS lines in `tests/printing/run.sh`.

Added to PrintDesigner:
- a **used-by chip**: /print-template-assignments/list?ptaTemplateId. The click opens PrintAssignments via the
  new `showRowsForTemplate()` (All-assignments tab + a "design: X ✕" filter);
- **Compare…**: the rail's selected revision beside the on-screen layout, both pretty-printed;
- **Preview… (F9)**: `print_designer_preview_dialog.*` asks for the doc uuid (remembered for the session), acc
  year, output mode, prompt answers, and "unsaved layout" (DRAFT only). It has **Test datasets** =
  /preview inspect:true (SAVED queries only) and opens DocumentPreviewDialog through its new raw-request constructor;
- **ESC/POS output** is shown as a readable dump (`DocumentPrint::escPosText`);
- the **prompts grid is now EDITABLE**: cell widgets on user-intent signals; Add, Remove, "Declare them";
  findings from `print_designer_rules` (bound :names, context aliases such as :comp_id→company_id, unused,
  required-can't-be-answered);
- **provider completer** from /print-render/providers;
- **publish refused with 0 datasets** (the web's rule);
- a **close guard** (reject() override).

Two real bugs fixed in print_template_payload:
1. A design with ONLY published revisions loaded an EMPTY working copy: the page showed A4/0 margins,
   Compare compared against nothing, and New draft copied an empty layout. parse() now loads the
   published revision (else the newest) with ptvId EMPTY (so it stays read-only), plus
   Draft.viewingPtvId/RevNo/Status. The chip reads "reading rev N · PUBLISHED".
2. A new revision (NewDraft, clone, create: no ptvId) now STRIPS ptdId from its datasets. Before, a
   clone or new draft sent the source revision's row ids.

NexDialog is WA_DeleteOnClose: a test must `new` a PrintDesigner/PrintAssignments, never put one on
the stack (a double free shows as "munmap_chunk(): invalid pointer").

Still left: retire / delete revision (the server probably refuses a pure retire; see the check-file row 2; test
live first), the band canvas (P5). See [[print-assignments-screen]], [[document-print-built]].
