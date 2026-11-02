---
name: erp-print-flow-layout
description: "Band `flowLayout` + element `positionType`/`removeLineWhenBlank` (built 2026-10-13) — where it runs, the keepTogether exception, and the SaleInvoiceA5 rev 10 draft that uses it"
metadata:
  node_type: memory
  type: project
  originSessionId: 7102eab9-29b7-4d7a-be7d-a14231289db7
  modified: 2026-10-13T08:05:30.608Z
---

Built 2026-10-13 from a pasted plan (written for a C++/Qt renderer; here the
only place is the server layout pass, `engine/layout/flow-layout.ts`, wired
into `layout.engine.ts` bandHeight/emitBand/emitCrosstabBand/drawBandElements).
GRAPHIC only; GRID ignores the flag (designer warns).

- Shifts are in DESIGN coords: visible crosstab → `fullHeightMm - h` at `y+h`;
  merged hidden `removeLineWhenBlank` spans with no visible occupant → minus
  their height. FLOAT moves by every shift at/above its top; LINE top = min(y1,y2).
- **keepTogether is honoured for flow bands only.** The engine never read it:
  a band with a sole crosstab that doesn't fit goes down the split path, which
  draws on the current page if header+1 row fit — so FIXED crosstab footers
  (e.g. SaleInvoiceA5 rev 9) can print totals past the page bottom. Left as-is
  for fixed bands because the plan demanded byte-identical non-flow output.
- `soleCrosstab` now counts VISIBLE crosstabs only (the split path used to draw
  a hidden one).
- Split flow band: FLOAT elements below the table print after its last row,
  re-anchored to the table end; tail that doesn't fit → next page.
- Client: `canvasBridge.normaliseBand` whitelists band keys — new band keys
  MUST be added there or open+save strips them. Element keys pass through raw.
- SaleInvoiceA5 **rev 10 = DRAFT** with the flow footer (heightMm 53.12 per the
  plan; case 6 then is 63.32mm vs rev 9's 62 — use 51.8 for exact parity).
  Publishing it needs the server change deployed first.

Related: [[erp-print-crosstab]], [[erp-print-template-save-clears-header]].
