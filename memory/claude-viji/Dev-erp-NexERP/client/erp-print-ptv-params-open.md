---
name: erp-print-ptv-params-open
description: ptv_params may declare ANY name including context ones — the six context params are a fallback default, not a closed set (changed 2026-09-04)
metadata:
  type: project
---

Changed 2026-09-04, both repos. `ptv_params` is now the whole parameter
declaration for a print template revision: **any** name may be typed into the
Data tab's Prompts grid, context names included.

**Before:** `company_id, branch_id, acc_year, doc_id, user_id, device_id` were a
CLOSED SET. Declaring one was refused at save (`checkVersionParams`), refused at
render (`readParamSpecs`), and the Data tab carried a read-only CONTEXT section
(`ContextNote.tsx`) whose only job was to warn against it.

**Now:**
- `PTV_CONTEXT_PARAMS` still exists, but only as the DEFAULT the dataset runner
  binds when the revision declares no prompt of that name — so every pre-existing
  template still prints without editing.
- A declared prompt OVERRIDES that default (`bindableValues` writes context
  first, then params). An unanswered (null) context-named prompt does not
  overwrite — it falls back.
- A declared context name is never "required and unanswered": the render has a
  value either way.
- `PTV_SERVER_OWNED_PARAMS = ['company_id']` is the one thing left. It may be
  declared (documents the binding) but an ANSWER to it is refused — the tenant
  guarantee. Empty that array to hand company selection to the author.
- Client mirrors it in `features/printing/domain/context.ts`:
  `hasContextDefault` (was `isContextParam`), `isServerOwnedParam`,
  `contextParam`. `ContextNote.tsx` is deleted; `buildPreviewRequest` strips
  server-owned keys out of `params`.

**The trap this creates (hit 2026-09-15, Saleinvoicea5):** a document screen's
print path answers NOTHING — `PrintOptionsDialog` sends only `versionId`,
`docId` and the document's `accYear`, because company/branch/counter are token
claims. So a revision that declares a **required prompt under a custom name**
(`comp_id` labelled COMPANY_ID, bound as `:comp_id` in its datasets) is
unprintable from any screen: `resolveRenderParams` skips the required check only
for the six context names, so it 400s with "'COMPANY_ID' is required by this
revision and was not answered." The design was fine otherwise — answering
`params: {comp_id: <uuid>}` by hand rendered the same PDF byte-for-byte.

Fix is in the TEMPLATE, not the client: rename the prompt and every `:binding`
to the context name (`company_id`), which the runner fills from the session and
refuses to let a caller set. Others still carrying this shape in the local DB:
`A4MTC` and `dasf` (required `wantdelete`, `comp_code`, `comp_short`,
`comp_name`, `legal_name`, `search`), `QUOTATIONA5 (copy)` (`quote_id`, optional
so it still renders).

**Why:** the closed set meant a report that needed the operator to pick the
document (`:doc_id`) had no way to say so.

**How to apply:** when adding a parameter rule, put it in `render-params.ts`
(render) and `print-template-invariants.ts` (save) together — they are the two
halves, and the client's `PromptsGrid` lint is a third, non-blocking copy.

See [[erp-print-template-contract]], [[erp-print-designer-ui]].
