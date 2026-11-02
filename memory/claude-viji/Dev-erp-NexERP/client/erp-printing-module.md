---
name: erp-printing-module
description: "features/printing (17_printing engine) — module map, and the three server defects it works around"
metadata: 
  node_type: memory
  type: project
  originSessionId: 76c16588-a3a4-4629-8391-6ed5a5e6715f
  modified: 2026-08-27T09:09:49.886Z
---

Built 2026-08-27 from the "Printing — React plan". `features/printing/` on
`/settings/printing/**`, over `/print-templates` + `/print-template-assignments`
(ptl/ptv/ptd/pta). `features/print-designer/` is the CANVAS this module hosts
(see [[erp-print-designer-ui]]); its own `/reports/templates` half — list,
designer route, preview — was retired to redirects on 2026-08-28, see "Nav, and
the dead `/reports` twin" below.

Screens follow `printing_ui_mockup` (title + subtitle, tab strip, section head
naming the TABLE and the payload SLICE, toolbar, dense table with an "add a row"
ghost line, tinted notes, footer bar). `components/screen-shell.tsx` holds
`ScreenShell` / `SectionHead` / `Note` / `Chip`; the version rail is a TABLE on
the Template tab, not a sidebar.

Shape: `domain/` is pure and tested first (buildSavePayload, draft, ladder,
linkFields, sqlLint, purposes, context); `types/printing.generated.ts` comes from
`npm run gen:printing-types` (reads the two OpenAPI docs; repairs Nest's
nullable-degradation, fails on conflicting enum/type declarations).

## Three server defects the client works around

1. **Any non-publishing save NULLS `ptl_published_rev_id`** — silently
   unpublishes a live design; every counter stops printing. `{ptlId, ptlName}`
   alone ALSO blanks `ptl_description`, resets `ptl_is_active` to true, and —
   worst — **nulls `ptl_company_id`, which means "shipped, visible to EVERY
   company"**: a rename hands one tenant's private design to all of them.
   Cause: the service asks `hasOwnProperty(dto, key)` whether a key was sent, but
   the DTO is a class-transformer instance and the server's `target: ES2022`
   turns on `useDefineForClassFields`, so every declared field is an own
   `undefined` property — the check is ALWAYS true. Fields with an `existing`
   fallback survive; those with `?? null` / `?? true` do not.
   **Workarounds:** every payload sends the WHOLE `ptl*` identity, never a diff
   (that covers company/description/active — `identityPayload` must never grow
   an "only what changed" mode while this stands); and `buildSavePayload`'s
   opt-in `preservePublishedRevId` echoes the pointer back (never with a
   publish — the server rejects both together).
2. **`ptvBody` must be pre-stringified**, despite the DTO's `toBodyText`
   transform: `enableImplicitConversion` coerces an object to `"[object Object]"`
   first, so sending `{bands:[]}` 400s with "did not parse as JSON".
3. Query booleans — see [[erp-server-query-boolean-false-ignored]].

## The canvas bridge (2026-08-27)

**`/reports/*` DOES NOT EXIST** — templates, schema, datasets/catalogue and
preview all 404 with a valid token; there is no reporting module in the server
at all (re-probed 2026-08-28: `/reports/templates` 404, `/print-render/*` 401,
i.e. present). So `features/print-designer` was a canvas with no backend, and its own
list page at `/settings/print-templates` is dead too.

`features/printing` now hosts it: `/settings/printing/templates/<ptlId>/layout?rev=<ptvId>`
opens the canvas on a revision, and Save writes `ptvBody`.

- `domain/canvasBridge.ts` — the translation. ON THE WAY IN the VERSION wins for
  paper and datasets, the body only for bands (a canvas-invented dataset would
  bind to nothing at render).
- `print-designer/host/canvas-host.tsx` — a context, not props: `useTemplateSave`
  is called from the shell, top bar and menu bar, so the host is ambient. With no
  host mounted the designer behaves exactly as before.
- `DesignerShell` gained `mode: "EMBEDDED"` (definition + datasets handed in,
  `seedKey` re-seeds). Preview/Publish/Revisions hide when hosted — all
  `/reports/*`.
- `selectListColumns` reads a SQL dataset's SELECT list to give the data panel
  field names; PROVIDER datasets get none (no catalogue endpoint).
- Saves use `includeDatasets: false` — the canvas does not own the dataset rows.
- **Gotcha found the hard way:** a stored `ptvBody` with a `bands` array is NOT a
  canvas definition (`{"bands":[{"kind":"HEADER"}]}` is a normal stub) and crashed
  it on `band.elements is not iterable`. `normaliseBand` fills defaults and drops
  unrecognised types.
- `draftStarted` takes `dirty` now: a hosted revision arrives already stored, so
  it must not open flagged as unsaved.

## Nav, and the dead `/reports` twin (2026-08-28)

`Settings → Configuration → Printing Configuration` used to point at
`/settings/print-templates`, the print-designer's OWN list — a `/reports/*`
client, so the list 404'd, the designer it opened 404'd, and that designer's
Preview posted `/reports/preview` (the 404 the user reported). It now points at
`/settings/printing/templates`.

**REPOINTING an existing bound label is the way past the seeding trap** in
[[erp-menu-permission-gating]]: `fixed.menu_master` 62 "Printing Configuration"
is already seeded and already granted (all 7 users with menus have it), so the
route stays governed by a grant people actually hold — no new row, no lock-out.
Governance is prefix-matched, so `/settings/printing/templates/**` (the designer
AND the layout canvas) is covered by the same grant. `/settings/printing/
assignments` is still unbound (no row) and reached from a design's "used by"
chip.

`/settings/print-templates`, `/print-designer/new` and `/print-designer/[id]`
are now REDIRECT stubs into the printing module — the feature code behind them
(`list/page`, `new-page`, `designer-page`) is left in place but unreferenced.
The legacy `renderPrintPreview` mutation and `PREVIEW_ENDPOINT` are DELETED from
`api/printTemplateApi.ts`, and `PreviewDialog` is host-only: no host preview, no
dialog (`DesignerShell` gates the mount, Ctrl+P and both toolbars on
`host?.preview`). There was never anything to repoint a hostless canvas at — the
renderer takes a revision id and nothing else.

Verified in a headless browser 2026-08-28: nav lands on the templates screen, no
`/reports/*` request fires anywhere, the three old URLs redirect, and Preview on
a published revision posts `/print-render/preview`. That render came back 400 —
`datasets.company.ptdSql` binds `:wantdelete, :company_name, :comp_name`, which
is that stored design's own SQL, shown verbatim by the dialog with its field
path. Fixable on the Data tab; not a wiring fault.

## Whose company a render runs as (2026-08-28)

TWO COMPANIES EXIST AND THEY ARE NOT THE SAME. The header picker is client state
and auto-selects `companies[0]`; the SERVER always acts as the access token's
`company_id`, minted at login from `usr_company_id` (`AccessTokenGuard` is the
only thing that sets the request context's company — no login-time choice, no
switch endpoint, no header override). The printing screens scope by the PICKER,
so a design can be authored into a company its own author can never render:
`POST /print-render/preview` answers "Revision … belongs to another company", and
the branch sent with it is the picker's too.

A fix for this (a `useSessionCompanyId` hook, the printing screens moved onto the
token's company, a pre-flight ownership refusal in `domain/previewRequest.ts`,
and the canvas dataset panel made read-only when hosted) was built and then
REVERTED at the user's request on 2026-08-28 — the behaviour above is what the
code does today. Related: [[erp-item-company-token-override]], where the server
silently overwrites a saved company with the token's.

A second trap found the same day and also left standing: the canvas's dataset
panel can bind a dataset the host never saves (`includeDatasets: false`, and the
version's list wins on reload), so an element bound to it resolves to nothing —
that is how SA1 got a field reading `{{ company2.comp_id }}` with no `company2`.

## Other verified facts

- `ptaSpecificity` numbering is UNSETTLED (applied migration 2/1/0; the
  authoritative 17_printing.sql 4/2/?/0; Swagger says 2/1/0 too). `domain/ladder.ts`
  therefore NEVER decodes the integer — the rung comes from the row's own scope
  columns, and a resolved winner's label from `/resolve`'s server-decoded `scope`.
- `print_purpose`, `printer_profile`, `print_log` have NO endpoints. **Purposes
  solved 2026-08-27 with a CONFIGURED DROPDOWN instead of a controller**:
  `fixed.dropdown_details` id **47** ("PRINT PURPOSES") + 4 `dropdown_columns`
  rows, read via `features/printing/api/purposes.ts`. It is RUNTIME DATA, not a
  migration — the SQL to recreate it is in the module manual, and the picker
  falls back to the derived list when the row is absent. The stored SQL scopes
  with `prm_company_id`, which `dropdown_param` substitutes BY NAME, so the
  param must ALWAYS be sent (omitting it leaves the bare token in the SQL and
  errors) — that is why `useLazyConfiguredDropdown` is not used here, it
  documents that it never sends `dropdown_param`. This also let the Assignments
  matrix finally list purposes with nothing configured, which is the "⚠ none
  set" row the screen exists for. Printer Profiles and Print History still not
  built.
- Assignments WAS read-only; since server migration `20260827140000` it is fully
  writable (`AssignmentEditor` creates, edits and deletes at every rung; re-checked
  2026-09-29). `ptaPrinterName` means the joined display name on reads but a fallback
  queue name on writes.
- `ERP server/prisma/staging/README-17-printing-correction.md` is the pending
  drop-and-recreate for pta/printer_profile/print_log; most §9 gaps close there.
- Integration test `domain/save-contract.integration.test.ts` is skipped unless
  `PRINTING_IT_{API,TOKEN,PURPOSE_ID,USER_ID}` are set. `PRINTING_IT_USER_ID`
  must be a REAL `user_master` row (`ptv_approved_by` is a true FK).

Related: [[erp-menu-permission-gating]] (routes deliberately unbound),
[[erp-auth-endpoint-verification]], [[erp-client-ui-browser-verification]].
