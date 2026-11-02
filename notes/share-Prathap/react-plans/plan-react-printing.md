# Printing — React plan

Scope: the printing module, React/TypeScript against `17_printing.sql` and the
`/api/v1/print-templates/*` endpoints that already exist.

This is **not a port**. Every other plan in this folder ports a Qt screen; this
one has nothing to port. `printQuotation`, `printSaleOrder` and `printSaleBill`
(`txn_doc_types_sales.cpp:96,226,267`) are "Printing isn't available yet" stubs,
and there is no `QPrinter`, `QPdfWriter` or `QTextDocument` anywhere in the
client. That is an advantage: nothing has to be un-learned, and §12 exists to
make sure nothing gets invented either.

The one idea, and it is the whole subsystem:

> **Nothing about printing is compiled into the client. Not the list of
> printable things, not the layouts, not the queries behind them, not which
> design a counter uses, not where the paper comes out.**

If the build ends up with a `FORMATS = [...]`, a `PURPOSES = [...]`, or a
`switch (docType)` choosing a layout, it has failed — that is precisely 3.0,
where `PrintUtil(this, 9)` meant a sales bill at ten call sites and nothing
anywhere said so.

Read §2 before anything else.

---

## 1. What gets built

Five routed screens. Mockup: `printing_ui_mockup.py` → `.png` in the share.

| Screen | Table | What the operator does |
|---|---|---|
| Print Templates (list) | `print_template` | Find a design, create or clone one, see what is published |
| Print Designer | `print_template` + `_version` + `_dataset` | Three tabs, **one save** |
| Print Assignments | `print_template_assignment` | Which design each branch and counter uses |
| Printer Profiles | `printer_profile` | Register the counter's printer |
| Print History | `print_log` | Nothing — read-only, it records itself |

Print Purposes is a sixth screen over `print_purpose`. Twelve rows ship; most
sites never open it. Build it last.

---

## 2. Read this first — two questions, and a four-rung ladder

Printing looks like one problem and is two, and the schema is organised around
keeping them apart:

**Which design?** — `print_template_assignment` resolves it.
**What does that design contain?** — the template's *published version* and its
datasets.

Neither is answerable from the other, and the client must never try.

### The design chain

```
print_purpose            WHAT can be printed
   ^ ptl_purpose_id
print_template           identity only — a name, an owner, a published pointer
   ^ ptv_template_id                                    (CASCADE)
print_template_version   the body, the page. Frozen once published.
   ^ ptd_version_id                                     (CASCADE)
print_template_dataset   where the rows come from
```

Two consequences the UI has to respect:

- **`ptlPublishedRevId` is not "the newest version".** A template can hold rev 5
  as a DRAFT while this still points at rev 3. Every list and every render joins
  on the pointer, never on `max(ptvRevNo)`.
- **`ptlPublishedRevId === null` means the design resolves for nobody.** It is a
  legitimate state — a design with only a draft — and it must be **loud** in the
  Templates list. Otherwise it is discovered at a till.

### The resolution ladder — four rungs, not three

`pta_specificity` is generated in the database and is **narrowest wins**:

```
3  one counter
2  one branch
1  one company
0  every company        (pta_company_id IS NULL — a shipped design for everyone)
```

Rung 0 exists because `pta_company_id` is nullable. **Do not hard-code the
decode**; if you must render the word, derive it:

```ts
const RUNG = { 3: 'Counter', 2: 'Branch', 1: 'Company', 0: 'Every company' } as const;
```

The live API still documents this as `2 counter, 1 branch, 0 company` (§9.3). A
client that trusts the Swagger description will label every counter override as
a branch one.

---

## 3. The API contract, and the six ways to get it wrong

Swagger: `https://192.168.0.106:3011/api/docs/print-template`
Spec: `/api/docs/print-template-json`. Bearer access-token, as everywhere else.

### 3.1 The one call — `POST /api/v1/print-templates/create`

Creates **and** updates. One request carries all three tables:

```jsonc
{
  "ptlId":        "…",            // absent = create
  "ptlCompanyId": "…",            // null = shipped design
  "ptlPurposeId": "…",
  "ptlCode": "INV_A4", "ptlName": "Invoice A4",
  "ptlSortOrder": 10, "ptlIsActive": true,

  "versions": [{
     "ptvId":     "…",            // absent = append the next revision
     "ptvStatus": "DRAFT",        // PUBLISHED moves the pointer, server-side
     "ptvEngine": "JSON_BANDS",
     "ptvBody":   { "bands": [] },
     "ptvPaperCode": "A4", "ptvOrientation": "PORTRAIT", "ptvLang": "en-IN",
     "ptvParams": [{ "name":"from_date","type":"DATE","required":true,
                     "label":"From date" }],

     "datasets": [
       { "ptdDatasetNo":0, "ptdRole":"MASTER", "ptdName":"header",
         "ptdSourceKind":"PROVIDER", "ptdProviderCode":"sales.bill.header" },
       { "ptdDatasetNo":3, "ptdRole":"DETAIL", "ptdName":"batches",
         "ptdSourceKind":"SQL", "ptdSql":"SELECT b.sbi_id, …",
         "ptdParentNo":1, "ptdLinkFields":"sbi_id=line_id" }
     ]
  }]
}
```

**Trap 1 — `"datasets": []` deletes every dataset on that revision.** Omitting
the key leaves them alone. These are not the same thing, and the difference is
one line in `buildSavePayload`. An array that is *present* replaces the set:
rows with `ptdId` update, rows without insert, rows on the version but absent
from the array are soft deleted.

**Trap 2 — a version missing from `versions[]` is left alone, never deleted.**
The history is append-only; removal is an explicit `ptvIsDeleted: true`, and the
server refuses it for a PUBLISHED revision and for the one the template
currently points at.

**Trap 3 — never send `ptlPublishedRevId`.** Publishing is `ptvStatus:
"PUBLISHED"` plus `ptvApprovedBy` on the version, and the server moves the
pointer. If you *do* send it, it must name a PUBLISHED, undeleted version **of
this template** — a rule `fk_ptl_published_rev` does not itself enforce. There is
no reason for this client to ever send it.

**Trap 4 — identity-only saves must omit `versions` entirely.** Renaming a
template while the whole draft round-trips is how a publish gets silently
reverted: `print_template` has **no `ptl_row_version`**, so nothing detects the
lost update. Send `ptl*` alone. (§7.)

**Trap 5 — `ptvBody` is polymorphic.** A JSON object for `JSON_BANDS`, a string
for `HTML_CSS` / `QTRPT_XML` / `ESCPOS_TEXT` / `RAW`. Type it as
`Record<string, unknown> | string` and switch on `ptvEngine`, never guess from
the value.

**Trap 6 — only a DRAFT may be updated.** A `ptvId` naming a PUBLISHED revision
is refused. The UI must not offer the edit at all (§7).

### 3.2 The other three

| Route | Screen |
|---|---|
| `GET /print-templates/list` | Templates list, paginated |
| `GET /print-templates/get` | opening one in the Designer — returns every revision and each revision's datasets |
| `DELETE /print-templates/delete` | Retire — soft, with every revision and dataset |

`get` returns the **whole** history. The Designer renders one working version and
a read-only rail; do not put every revision's body in component state.

### 3.3 Assignments — `POST /api/v1/print-template-assignments/create`

Plus `list`, `get`, `delete`, and **`GET /print-template-assignments/resolve`**
(`companyId`, `branchId`, `deviceId`, `purposeId`, `outputMode`) which returns
the template, its published revision, the printer and the copy count.

`resolve` is what fills the "Effective here" column in §8. **Do not reimplement
the ladder in the client.** The server has the generated column and the covering
index; a front-end that re-derives narrowest-wins will drift from it.

This controller is **behind the schema and currently cannot insert a row** — see
§9. Phase it accordingly (§11).

### 3.4 Not built at all

`print_purpose`, `printer_profile` and `print_log` have no routes. `print_log` is
fine — it writes itself server-side. The other two are hard blockers for their
screens.

---

## 4. Module layout

```
src/features/printing/
  routes.tsx
  templates/
    TemplateListScreen.tsx
    components/ TemplateTable.tsx  PublishedBadge.tsx  UsedByCell.tsx
  designer/
    DesignerScreen.tsx            // three tabs, ONE save
    tabs/ TemplateTab.tsx  LayoutTab.tsx  DataTab.tsx
    components/
      VersionRail.tsx             // history + publish / roll back / compare
      DatasetGrid.tsx             // no. / role / source / parent / link
      SqlEditor.tsx
      PromptsGrid.tsx             // ptvParams — version level, NOT per dataset
      ContextNote.tsx             // the six context names, read-only
  assignments/
    AssignmentMatrixScreen.tsx    // §8 — scope-shaped, not design-shaped
  printers/  PrinterProfileScreen.tsx
  history/   PrintHistoryScreen.tsx
  api/
    templates.ts  assignments.ts
  domain/                         // PURE, tested first
    buildSavePayload.ts           // §3.1, all six traps
    draft.ts                      // new draft / publish / rollback transitions
    ladder.ts                     // specificity decode, 4 rungs
    linkFields.ts                 // parse + validate parent=child
    sqlLint.ts                    // mirrors the CHECKs; server is authority
  types/ printing.ts              // generated from the OpenAPI spec, not typed by hand
```

`buildSavePayload`, `draft`, `ladder`, `linkFields` and `sqlLint` are pure and
written **before any component**. Every trap in §3 and every rule in §7 lives in
one of them.

Generate `types/printing.ts` from `/api/docs/print-template-json`. Hand-written
DTOs will drift, and this contract has 40+ fields across three nested levels.

---

## 5. The Designer is one screen

Three tabs — **Template**, **Layout**, **Data** — and **one Save**. The tabs are
a reading convenience, not three forms: they are two halves of a single request
body. Label each section with the payload slice it fills, as the mockup does:

| Section | Tab | Fills |
|---|---|---|
| Identity | Template | `ptl…` |
| The Page | Template | `versions[0].ptv…` |
| Version rail | Template | `versions[]` |
| The Layout | Layout | `versions[0].ptvBody` |
| Datasets | Data | `versions[0].datasets[]` |
| Prompts | Data | `versions[0].ptvParams` |
| Context | Data | *(nothing — see §6)* |

There is **no separate create screen**. "New template" on the list opens the
Designer blank: no `ptlId`, no `ptvId`, and the one call creates the template and
rev 1 DRAFT together.

---

## 6. Parameters — one bag, declared once

The single most duplicated thing in a naïve build.

**CONTEXT parameters are declared nowhere.** `:company_id`, `:branch_id`,
`:acc_year`, `:doc_id`, `:user_id`, `:device_id` are a closed set with one fixed
type each, forever. The server holds the registry and works out which a query
uses by *reading the query* — exactly as `ck_ptd_sql_company_scoped` already
greps `ptd_sql_norm` for `:company_id`. They appear in no table and in no
payload. Render them as a read-only note so an author knows what is available;
never send them.

**USER prompts are `ptvParams`, on the version.** They are asked **once** for the
whole render, so they are declared once. They were on the dataset and it was
wrong: two datasets could declare `from_date` as `DATE required` and `TEXT
optional`, and there is no answer to what the screen should ask.

Corroborating evidence, if it is ever questioned: `plg_params` is one JSONB
object per render, not one per dataset.

**Nested bands do not pass parameters either.** `ptdParentNo` + `ptdLinkFields`
(`parent=child`, comma separated, no spaces) express nesting. The child query
runs **once for the whole render**, bound with the same context, returns the
parent key as an ordinary column, and the renderer groups on it. `ptdRowLimit`
and `ptdTimeoutMs` measure the *whole band* on that basis. A UI that suggests
per-row execution will lead someone to write a query that is N+1 against
partitioned tables.

---

## 7. Rules the client must enforce, because the database will not

**There are no triggers in `17_printing.sql`.** Every rule below is prose in the
schema and nothing else. This screen is the enforcement.

1. **A PUBLISHED revision is read-only.** The only affordance on one is *New
   draft*. If the editor lets someone type into a published row, the audit
   guarantee behind `plg_version_id` — a real FK to the exact bytes rendered —
   stops being true.
2. **New draft copies the datasets forward too.** Datasets belong to the version;
   a new draft with an empty `datasets[]` renders nothing.
3. **`ptdDatasetNo` is immutable once published.** It is *the binding* a band
   points at. Reordering the list writes **`ptdSortOrder`** and nothing else, and
   the two must look like different operations on screen. 3.0 had one column
   doing both jobs, which is why reordering rows silently rebound every band to
   the wrong query.
4. **Roll back writes forward.** A new revision whose body is the old one, then
   publish it. Never move the pointer backwards — the history must stay
   append-only, so a rolled-back design is still evidence.
5. **Identity saves omit `versions`** (§3.1 trap 4).
6. **Publishing needs an approver.** `ck_ptv_published` refuses PUBLISHED with a
   null `ptvApprovedBy`. Capture it deliberately; do not silently default it to
   the current user, or the signature means nothing.

---

## 8. The Assignments screen is scope-shaped

Selectors on top — **branch**, **counter**, **output mode** — then one row per
purpose against three scope columns and an **Effective here** column fed by
`/resolve`:

```
Purpose         Company-wide   Branch: Adyar   COUNTER-3      Effective here
Tax Invoice     Invoice A4     —               Thermal 3in    Thermal 3in  ← counter
Delivery Slip   Slip A5        Slip-Adyar      —              Slip-Adyar   ← branch
Quotation       —              —               —              ⚠ none set
```

That last row — a purpose with nothing configured — is a till that prints
nothing, and this is the only screen where it is visible. Make it loud.

**It cannot be a tab on the Designer, and this is not a style preference.** A
per-design view is scoped to one `ptaTemplateId`; the row that *overrides* it
lives on a **different template**. Open Invoice A4, see a company-wide row, and
conclude every counter uses it — COUNTER-3 does not, and the evidence is not on
that screen. Resolution is a question about a scope.

On the Designer, show instead a **read-only** "used by 3 counters in 2 branches"
chip that links here filtered by template. That is the question a designer
actually has: is this safe to change?

---

## 9. Backend gaps this module inherits

Diffed field-by-field against the live spec on 27-08-2026. The template side is
in sync — `ptvParams` on the version, no `ptdParams`, no sample columns, and all
five vocabularies match their CHECKs. The assignment side is not:

1. **`ptaTemplateCompanyKey` is missing from the DTO, and the column is `NOT
   NULL` with no default.** The create endpoint cannot insert a row at all. Hard
   blocker for the Assignments screen.
2. **`ptaCompanyId` is required**, in the save DTO and in `resolve`. Rung 0 —
   an assignment for every company — cannot be expressed.
3. **`ptaSpecificity` is documented `2 counter, 1 branch, 0 company`.** It is now
   3/2/1/0 (§2).
4. **`ptaPrinterName` means two different things.** The payload DTO already uses
   it for the *joined display name* of the printer profile; the new column is a
   *fallback queue name*, mutually exclusive with `ptaPrinterId` via
   `ck_pta_printer_one_of`. One of the two must be renamed before either ships.
5. **`ptaOutputMode` mismatches both ways.** The API offers `FILE`, which the
   CHECK refuses; the schema allows `PDF`, `WHATSAPP` and `ESCPOS`, which the API
   cannot send. Output mode is part of the resolution key, so this is not
   cosmetic.
6. **No endpoints for `print_purpose` or `printer_profile`.** Both screens are
   blocked; the Designer's purpose dropdown needs the first one.
7. **`ptvParams` element shape is unvalidated** — the only constraint is
   "must be a JSON array". Nothing checks that the names declared match the
   `:names` in `ptdSql`, so a mismatch saves cleanly and fails at render. Mirror
   the check client-side (`sqlLint.ts`) and treat it as a lint.

---

## 10. Tests

Pure units first. They cover every trap:

- **`buildSavePayload`** — identity-only omits `versions`; a new draft omits
  `ptvId`; publish sets `ptvStatus` + `ptvApprovedBy` and **never** emits
  `ptlPublishedRevId`; `datasets` omitted vs `[]` produce different bodies;
  `ptvBody` object for `JSON_BANDS` and string for the others.
- **`draft`** — new draft copies body *and* datasets; roll back produces a new
  revision, never a backwards pointer; a PUBLISHED version is not editable.
- **`ladder`** — 3/2/1/0 decode, and that nothing anywhere maps 2 to "counter".
- **`linkFields`** — accepts `sbi_id=line_id` and `sb_id=bill_id,sbi_slno=slno`;
  rejects spaces, upper case, a bare name and a trailing comma. Mirror
  `ck_ptd_link_fields_shape` exactly; the regex is in the schema.
- **`sqlLint`** — one case per guard, especially the two that are subtle: a
  literal appearing *before* a real parameter must pass, and `:name` *inside* a
  literal must fail.

Then one integration test against a running backend, in order:

```
create, no ptlId, one version no ptvId    -> template + rev 1 DRAFT
save identity only, no versions key       -> name changes, rev 1 untouched
save the draft with its ptvId             -> updated in place
publish it                                -> ptlPublishedRevId now names rev 1
new draft                                 -> rev 2 DRAFT, datasets copied
"datasets": [] on rev 2                   -> rev 2 has none, rev 1 still intact
```

That last pair is the one worth writing first. It is the difference between the
two omission semantics, and it is silent when wrong.

---

## 11. Phases

1. `domain/` + tests. Generate `types/printing.ts` from the spec. No UI.
2. Templates list — `list`, the published badge, the **not published** state.
3. Designer read-only — `get`, all three tabs rendering, version rail, nothing
   editable.
4. Draft editing + Save, with all six traps, then the integration test.
5. Publish / roll back / new draft (§7.1–7.4, 7.6).
6. Datasets: grid, SQL editor, `sqlLint`, nesting with `linkFields`; prompts.
7. **Assignments — blocked on §9.1–9.5.** Build the read-only matrix against
   `list` + `resolve` first; the editor only after the DTO is fixed.
8. Printer Profiles and Print Purposes — blocked on §9.6.
9. Print History — read-only, last. It needs no endpoint work but it also
   unblocks nobody.

Phase 4 before phase 5, deliberately: saving one draft correctly is worth more
than a publish button, and every trap is in the save.

---

## 12. Do not build

- **A client-side print pipeline.** No `jsPDF`, no `pdfmake`, no
  `window.print()` of a hidden div. Rendering is server-side, and that is what
  deletes 3.0's `pstyl_platform` problem — every template existed twice, Linux
  and Windows, with identical SQL and two XMLs to keep in step, only because
  QtRPT rendered on the client. A React renderer re-creates that on a new axis.
- **Any hard-coded list of purposes, formats, engines or paper sizes.**
  `print_purpose` is a table precisely because the list is not closed — a sweet
  shop wants a Kitchen Order Ticket. Vocabularies come from the OpenAPI enums,
  which come from the CHECKs.
- **Client-side ladder resolution.** Use `/resolve`. §3.3.
- **A per-design "where it prints" editor.** §8.
- **Any `is_default` concept.** Default-ness *is* the existence of a `pta_` row.
  A boolean in the client will reintroduce the clear-then-set bug the schema
  removed — 3.0's live data holds several defaults for one group because of it.
- **A print-count cache.** `sb_print_count` is a server-side cache of
  `COUNT(DISTINCT plg_render_id)` filtered to PRINT/ESCPOS + PRINTED, moved once
  per render and never per copy. The client displays it and never computes it; a
  three-copy tax invoice would otherwise move it by three and the duplicate
  watermark would lie.

