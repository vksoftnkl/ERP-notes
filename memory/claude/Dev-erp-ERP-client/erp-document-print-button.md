---
name: erp-document-print-button
description: How a screen prints a saved document (POST /print-render/print) and the three config things that must be true for it to resolve
metadata:
  type: project
---

Wired 2026-08-28: quotation Print (list-page toolbar + F8 modal) and Save &
print (F6) call `POST /print-render/print` via
`features/printing/hooks/useDocumentPrint.ts`.

`CrudMasterPage`'s toolbar Print button was hard-coded `disabled` with no
handler (like Import still is). It is now OPT-IN: a page supplies
`onPrintAction` (+ `isRowPrintDisabled` / `rowPrintDisabledReason` /
`printBusy`) and the shell gates on `menuPermissions.canPrint` and a selected
row. Without the prop the button stays inactive, which is right for every
master that lists something that is not a document.

Client contract (all in `features/printing/domain/documentPrint.ts` + `api/render.ts`):
- Screens name a purpose **CODE** (`PURPOSE_CODE.SALE_QUOTATION`); the id is
  resolved through the purposes dropdown 47. Never hard-code `ppo_id`.
- **2026-09-15: the client sends the DOCUMENT's `companyId` again** (row's
  `sb_company_id`/`sq_company_id`/`so_company_id`), on preview, print AND the
  dialog's `/resolve` calls. Branch, counter and (usually) year stay server-derived;
  see "Scope is the session's" below and the token-vs-header trap at the end.
  Historical note, because the trap is still real for anyone re-adding a field:
  two device ids exist and only `userInfo.deviceId` is a `fixed.device_master`
  row; `getOrCreateClientDeviceId()` is a localStorage `randomUUID` for
  transaction holds and names no row, so sending it resolved no counter rung and
  failed `fk_plg_device` *after* rendering.
- No `templateId` either (the assignment ladder decides).
- Delivery is a hidden off-screen iframe + `contentWindow.print()` — the object
  URL only exists inside an async callback, so `window.open` is popup-blocked.

**A print only resolves when all three hold** — each is data, not code:
1. An assignment exists at a rung the session can reach (see the device-id note
   above — with the real counter id the seeded COUNTER rows do resolve).
2. The assigned template has a **published** revision (`ptl_published_rev_id`).
3. Its datasets bind `:doc_id`, not a prompt. `:company_id :branch_id
   :acc_year :doc_id :user_id :device_id` are the server-held context binds
   (`PTV_CONTEXT_PARAMS`); anything else must be a declared `ptv_params`
   prompt, which a document print button cannot answer.
   RECURS whenever a new design is drawn from the old source SQL — fixed twice
   (2026-08-28, 2026-08-29). The fix is TWO columns: `ptd_sql` `:quote_id` ->
   `:doc_id` AND removing the `quote_id` entry from `ptv_params` (a declared
   required prompt fails first, before dataset binding). `ptv_params` is JSONB,
   not text. Sweep with:
   `select ... from print_template_dataset where ptd_sql like '%:quote_id%'`
   plus `ptv_params <> '[]'`. NOTE `regexp_matches(sql, ':([a-z_]+)')`
   over-matches `::uuid` casts — use `'[^:]:([a-z_]+)'`.
   Still broken elsewhere as of 2026-08-29: DELIVERY_CHALLAN rev
   01a046a7-6e09… and SALE_ORDER rev 01a046d0-cd25… declare required prompts
   (`wantdelete`, `comp_code`, `search`…) scavenged from master-grid SQL.
   FIXED 2026-08-28: all 5 quotation revisions' 15 datasets bound `:quote_id`;
   rewritten to `:doc_id` by direct SQL (published revisions are frozen to the
   API, and `print_log` was empty so no record pointed at those bytes). The
   quotation now renders a real PDF. Restore script:
   `scratchpad/quotation-datasets-backup.sql` (session-local).

**The dialog.** The toolbar Print opens the legacy five-button popup
(`components/print-options-dialog.tsx`): ALL FOUR buttons render through
`/print-render/preview` into `components/document-preview-dialog.tsx` IN PLACE —
no navigation, the operator never leaves their list. Print is Preview plus
`autoPrint`: it hands the blob to `print-delivery.ts` and CLOSES the popup, so
nothing sits behind the print dialog.

`print-delivery.ts` owns the printing iframe + blob at MODULE scope, not in a
hook or component — the print dialog keeps reading the document while the
operator picks a printer, so a React owner would revoke the blob on unmount and
print a blank page. `sendToPrinter` takes ownership of the URL; the previous
one is only reclaimed when the next print replaces it. `useDocumentPrint`
(F6 Save & print) uses the same module.

Headless Chromium CRASHES on `print()` against its PDF plugin, so auto-print
cannot be verified headlessly — stub `window.print` via `addInitScript` to test
everything around it. NOTHING writes to `print_log` any more: `/print-render/print` was
swapped out on request 2026-08-28, so copy counts and ORIGINAL/DUPLICATE labels
are not applied either. `useDocumentPrint` still exists and is still fully
wired, but as of 2026-08-29 NO screen calls it — the list page, F6 Save & print
and the F8 picker all go through the dialog. Swapping it back restores logging. FORMAT DOES NOT RENDER: OK stores the chosen design
(`selectedFormat`) and returns to the five buttons, which then act on it —
choosing a format answers "on what", the buttons answer "and then what". With
no format chosen the buttons fall back to their defaults: the ladder's winner; the operator's choice from
`/print-templates/list?ptlPurposeId=` (every DESIGN drawn for the purpose,
scoped by `ptlCompanyId`); and the first ASSIGNMENT. Format and Pdf read
different sets on purpose — "print it on something else" vs "the usual way".

`includeVersions=false` on that list CANNOT be sent: it defaults to true and
`enableImplicitConversion` coerces "false" to `true`, so the pick list always
arrives whole (~360 KB for 4 designs). Verified 2026-08-28 — identical bytes
with the key absent, `=false`, and `=0`. Server change to fix.

The preview popup is NOT the designer's PreviewDialog: it asks nothing, sends no
body (`buildDocumentPreviewRequest`), and renders on open. It calls
`/print-template/get` to choose a revision — named, else published, else newest
(`revisionForPreview`).

**Output mode is part of the resolution key.** Preview resolves `PREVIEW` first
and falls back to the no-mode (PRINT) resolve, because an installation may
assign a different design per mode. Deleting the PRINT assignment makes
`/print-render/print` 404 while Preview still works.

## Scope is the session's, not the caller's (2026-08-29)

Company, branch, counter and accounting year are **no longer sent by any screen**
and are no longer required by any endpoint. On the server:

- `AccessTokenClaims` gained `branch_id` + `device_id`, signed at login from the
  `fixed.device_master` row the login already matched, and carried through
  `refresh` from the refresh payload (re-resolving is impossible — no device is
  presented on a refresh, and dropping them would silently demote a counter
  session to a company-wide one 15 minutes after login).
- `RequestContextService` gained `getBranchId()` / `getDeviceId()`, set by
  `AccessTokenGuard` alongside the company.
- `print-render.controller.contextFrom` falls back to those; `RenderDocumentDto`
  `accYear` is now OPTIONAL and `PrintRenderService.withCurrentAccYear` fills it
  from `fiscal_years.fy_is_current` (via `PrintLogService.currentAccYear`).
- `/print-template-assignments/resolve` — `companyId`/`branchId`/`deviceId` all
  optional, filled from context in the CONTROLLER. **The company must be filled
  even to null-check it**: the service builds
  `OR: [{ptaCompanyId: null}, {ptaCompanyId: <x>}]` and Prisma DROPS an
  `undefined` from a where clause, which turns the second arm into "match
  anything" and would resolve another tenant's assignment.
- Explicitly-sent values still WIN — the Assignments screen's effective-design
  matrix depends on it ("what would a DIFFERENT counter print").

**MIGRATION CONSEQUENCE: everyone must log in again.** A token minted before
this carries no `branch_id`/`device_id`, normalizes to null, and resolves the
ladder from the company up — so COUNTER-scoped assignments (most of the seeded
ones) 404 until re-login. Verified both ways locally 2026-08-29.

Verified live 2026-08-29: `POST /print-render/print` with a body of only
`{purposeId, docId}` renders a real quotation PDF and writes a `print_log` row
carrying branch + device from the token and `plg_acc_year` 2026-2027 from
`fy_is_current`. **Beware Redis `http-cache:`** when testing two tokens against
`/resolve` — the second request returns the first's answer; flush
`http-cache:*` between them or the test silently passes.

No stored `ptd_sql` binds `:acc_year` today (`select count(*) … like '%:acc_year%'`
= 0), so the year currently only decides the `print_log` partition. The quotation
screens still pass the ROW's own `sq_acc_year` anyway, because it costs nothing
and is correct the day a design does bind it.

Sale Order has the identical stub (`onSaveAndPrint` toast) and is now a
two-line change. Related: [[erp-printing-module]], [[erp-print-template-contract]],
[[erp-quotation-entry-port]].

## Token company ≠ header company → BLANK paper (fixed 2026-09-15)

`AccessTokenClaims.company_id` is `user_master.usr_company_id` — the user's HOME
company — while the header company picker (`components/layout/business-context.tsx`)
is pure client state that every screen honours via `grid_param`/body fields. VIJAY
is a THE SCM SILK user working in Acme Foods: the bill list (grid 86) showed Acme
bills, but `/print-render/preview` bound `:company_id` from the token, every
dataset (`header`, `items`, `taxes`, `charges`) filters on it, and the preview
rendered 200 OK with an empty SALES INVOICE. `/resolve` without a company also
picked the token company's assignment (SA1) instead of Acme's (Saleinvoicea5).

Fix: `RenderPreviewDto`/`RenderDocumentDto` gained optional `companyId`, and
`print-render.controller.contextFrom` prefers it over the token (same rule as
`/resolve`). Client: `DocumentPrintTarget.companyId` + `DocumentPreviewInput.companyId`
→ `print-options-dialog` uses `target.companyId || header companyId` for resolve and
the preview. All six `<PrintOptionsDialog>` call sites pass the ROW's company.

Diagnose fast: `audit.user_login_sessions.uls_company_id` for the latest login vs
the row's `*_company_id`; mint a token with that company (scratchpad `mint.mjs`
takes `COMP`/`BR` env) and POST preview with/without `companyId`.
