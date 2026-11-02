---
name: printing-parity-2026-10-30
description: Printing parity with the web, built 2026-10-30 — what exists, the switches, the grid, what is still open
metadata:
  type: project
---

Built 2026-10-30 with six parallel worktree agents, merged and committed as Qt `131b042`. It extends [[document-print-built]], [[print-designer-parity]], [[print-canvas-build]] and [[print-assignments-screen]].

**What exists**
- **Voucher prints:** RECEIPT_VOUCHER / PAYMENT_VOUCHER / EXPENSE_VOUCHER, with Post & Print, F11 reprint, and F6 / F7 on the registers. Drafts don't print.
  - On the dev box, PAYMENT_VOUCHER has no design.
- **Print log:** `DocumentPreviewDialog` POSTs `/print-render/log` for PREVIEW (paper shown), PRINT and FILE.
  - `Target.isReprint` exists but is never sent: only `/print-render/print` reads it, and Qt never calls that.
  - `Target.params` goes top-level in `/preview`.
  - The live part of `test_document_print` now writes a PREVIEW `print_log` row.
- **WhatsApp:** `PrintingModule::fetchPdfWithLink` adds a `/print-share/create` link on the sale bill and the PO; the clipboard PDF is kept.
- **New screens:**
  - Ledger Statement, menu 258 (`accounts/report/ledger_statement`).
  - Party Outstanding, menu 279 (`accounts/report/party_outstanding`). Ctrl+L opens 258. Export is CSV only, since Qt has no xlsx writer. Ctrl+R and Enter-to-voucher are not wired.
- **Designer:**
  - element features in `canvas_styling.*` and `canvas_properties_styling.cpp`;
  - page features in `canvas_page_*`, `canvas_chart.*`, `canvas_report_pages.*` and `canvas_units.*`;
  - the ◆ highlight badge, DOUBLE outlines, crosstab totals / subtotal rows, P-labels in the report tree, the unit kept in QSettings `printCanvas/inches`, 60-character formats checked.
- **Edit published in place:** `PrintDesignerRules::kEditPublishedInPlace = true`, for the testing phase with one client. A published design with no draft opens editable, and Save sends the next revision PUBLISHED in one call.
  - Preview of unsaved edits is impossible in this mode (the server takes a body only for a DRAFT).
  - Setting it to false restores read-only → New draft → Publish.
- **Print Templates (menu 62):**
  - A `NexCrudMasterPage` over the grid 'MAIN LIST - PRINT TEMPLATES', found BY NAME through `/grid-details/get?search=` (fallback 196). A missing grid shows a notice; it does not load a dead id.
  - Server migrations: 20261030150000 (grid), 160000 (widths), committed as server `41bf7124`. 170000_print_template_list_grid_badges is WRITTEN, but not applied or committed as of this note.
  - Tokens `iptl_company_id / active / purpose_id / state / owned / engine`, all always sent.
  - Pills: owner SHIPPED/OWN, state PUBLISHED/NOT PUBLISHED, `ptl_is_active` boolean. The hint strip says why Retire is off on a shipped design.
  - Test: `tests/printing/test_template_list_grid_live.cpp`, read-only.
- **React:** an agent was moving the React list onto the same grid and adding edit-in-place, uncommitted in ERP client `dev`.

**Why:** the user asked for every gap between the React and Qt printing to be closed, and for published designs to be editable while one client tests.

**How to apply:** check the items marked open above before claiming them; the session-resume note has the current state.
