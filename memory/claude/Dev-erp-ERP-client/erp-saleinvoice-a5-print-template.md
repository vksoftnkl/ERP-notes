---
name: erp-saleinvoice-a5-print-template
description: "Company-scoped \"Saleinvoice A5\" (ESTIMATE) template — rev 2 row-alignment fix 2026-10-15: one column grid, face = L×W+rate, flow-footer geometry traps, offline render harness recipe"
metadata:
  node_type: memory
  type: project
  originSessionId: 4a1bb783-ac55-4a63-ab78-95e1a972bb67
  modified: 2026-10-15T06:24:47.949Z
---

`SaleInvoiceA5` / "Saleinvoice A5" (ptl `01a12f38-784d…`, company Amazon Retail
`019c8ea6…`, forked from the shared SaleInvoiceA5 rev 9) is what the Sale Bill list
previews in DEV. Rev 1 = the user's design; **rev 2 (`01a13e3b-8a1c…`) published
2026-10-15** via `POST /print-templates/create` (full header, no ptlPublishedRevId —
see [[erp-print-template-save-clears-header]]); rev 1 kept for rollback. Dev DB only.

What rev 2 fixed (user: "correct the row wise adjustments"):
- Headings and figures share ONE box per column (Qty x56 w26 right, Unit x84, Rate
  x97 w19 right, Amount x116 w27 right → edge 143 = grand total's edge). Rev 1 had
  headings centred/right in different boxes and qty overflowing an 11mm box.
- Every table row 4.6mm, text y0.2 h4.2 middle (item heading 5.2).
- **Items SQL summed `sbi_rate` over the L×W** (printed 1,244 for 400+400+44+400).
  Now level-1 group = `face_key` (L×W + rate, matching the Size Entry sheet's
  regroup rule [[erp-quotation-multi-size-entry]]), `lxw_rate` = the line rate;
  `own_figures` (unsized line prints its own qty/rate/amount) and `print_line`
  (band visible) columns added. Charges SQL drops `cd_net_amt = 0`.

Engine traps found here:
- A crosstab row is `max(rowHeightMm, font line height)` — at 10pt that is
  **4.805mm**, so `rowHeightMm: 3.4` is only a floor; other footer rows must use the
  same pitch. Blank 6.5pt-bold header row = 3.123mm.
- Flow `removeLineWhenBlank` occupancy is by VERTICAL span only (any x). The tall
  grand-total box (y0 h20.8) therefore keeps a hidden Discount line open when Cash
  Discount shows — accepted, noted in the body's meta.
- A FLOAT row only moves for a crosstab whose design bottom (y+h) is at/above its
  top — rev 1 had the crosstab at y2.98 so Discount (y4.6) overprinted charge 1.
- Group headers do not repeat on a new page (cuts continue headless on page 2).

**Offline render harness** (fast iteration, no saves): a `tsx` script run from the
server dir that `require`s `print-render/engine` (FontRegistry, LayoutEngine +
TextMeasurer, PdfKitRenderer), `data/dataset-sql-binder.bindDatasetSql` and
`data/value-coercion.coerceResultRows`, runs each dataset's SQL via `pg` (set
`types.setTypeParser(1082, v => v)` like PgService), parses with
`templateDefinitionSchema` using the VERSION row's paper/margins, then `pdftoppm`.
Matched the server's `/print-render/preview` output exactly.
