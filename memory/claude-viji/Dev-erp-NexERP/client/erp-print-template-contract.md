---
name: erp-print-template-contract
description: Print-template definition contract gotchas — page-relative x, GRID cell units, declaration-order bands, endpoint paths
metadata:
  type: reference
---

Four facts about `reports.print_template` definitions that the schema's own field
names and doc comments get wrong. All verified against the running engine and the
six shipped templates (2026-08-24).

1. **`x` is PAGE-relative, `y` is band-relative.** `layout.engine.ts` emits
   `baseX = element.x` with no margin offset, and `baseY = bandTop + element.y`.
   The schema comment says both are band-relative; it is wrong about `x`. The
   shipped GST A4 invoice proves it: 8mm margins, frame at `x: 8, w: 194`.
   So the designer canvas draws bands at full page width and clamps `x` to the
   page, not the printable area; the right margin sits at `widthMm - margins.right`.

2. **In GRID mode `x`/`y`/`w`/`h` hold CHARACTER CELLS, not millimetres** — they
   mirror `col`/`row`/`cols` exactly (thermal receipt: `x: 28, w: 20, h: 1`).
   Treating them as mm collapses a 48-column receipt into the first 3mm.

3. **Bands are emitted in DECLARATION order, not band-type order.** That is what
   lets one template carry two sequential DETAIL bands — item lines, then the
   Rule 46 HSN/rate summary. Group bands are matched to a detail band BY DATASET.
   Sorting bands by type in a UI misrepresents every GST invoice.

4. **Endpoints** live under `/reports` — IN THE ENGINE THIS DOCUMENTS, which is
   NOT mounted in the ERP server: every path below 404s there, and the client
   calls none of them any more (see [[erp-printing-module]]; rendering is
   `POST /print-render/preview`, which takes a revision id, not a definition).
   The paths, for whenever that engine is revived: templates CRUD at `/reports/templates`,
   vocabulary at `/reports/templates/schema`, dataset catalogue at
   `/reports/templates/datasets/catalogue` (mounted under templates, not
   `/reports/datasets`), and preview at `/reports/preview` (POST a definition,
   returns a PDF or raw printer bytes). No `validate-expression` endpoint exists.

**CORS gotcha:** the API sends no `Access-Control-Expose-Headers`, so
`X-Report-Template-Id` / `-Version` / `-Page-Count` / `-Render-Ms` are unreadable
from a browser cross-origin. Production is same-origin behind nginx
([[erp-frontend-vps-deploy]]) so they work there; local dev (client :3001,
API :3011) sees nulls.

Client implementation: [[erp-print-designer-ui]].
