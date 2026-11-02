# Purchase Entry (Qt) fixes — user list of 2026-11-02

Source: the user's pasted list, 2026-11-02. Qt only (NexERP `main`), like the 11-02 PO work.
The web Purchase screen gets no new columns in this round.

## Already done earlier on 11-02 (do NOT redo)
PO entry points 1–6 (credit-days width, no boxes in cells, landing-cost columns, charges in
landing cost before/after tax, even quick icons, totals beside charges):
NexERP `main` 721dabe + bcd844a, ERP server `DEV-v1` bbef3f41. See memory `po-entry-landing-cost`.

## Purchase Entry (menu = Purchase Invoice, `src/modules/purchase/invoice/`)
| # | User asked | What to build | Layer |
|---|---|---|---|
| P1 | Buttons as quick buttons like PO | From GRN / PO, Sister-company bill, Return from bill, Pay this bill, History → the PO's icon quick bar (same icon size/look as PO 721dabe) | Qt |
| P2 | Remove e-way bill & e-invoice details for now | Hide the E-way bill / IRN / Ack fields (keep code + JSON keys so they come back later; send them blank/unchanged) | Qt |
| P3 | Agent as dropdown | Agent becomes a dropdown (searchable combo) of the Agents master instead of the grid-194 popup | Qt (+ route if none) |
| P4 | Options below the table are unwanted | Remove the "More columns (column chooser, saved per user)" row; show/hide columns only via NexTable's own layout | Qt |
| P5 | No hint inside the table | No placeholder / hint text in line-grid or charges-grid cells/editors | Qt |
| P6 | Box shown inside the cell (Rate, Disc …) and in the charges table | Same fix as PO 721dabe — frameless in-cell editors in the lines and charges grids | Qt |
| P7 | Cols Disc %, Disc, Sch %, Sch, Extra disc | Add Disc (amount) and Sch (amount) next to Disc % / Sch %; typing an amount clears the % and vice versa (server already takes discAmt / schAmt) | Server keys + Qt |
| P8 | Markup for all four price levels | Markup A–D (today only A); typed markup → Sell X, typed Sell X → markup; same basis as today's Markup A (landed cost without tax) | Server + Qt |
| P9 | Cols HSN, MSP | HSN (from the item, read-only) and MSP = `item_price_master.ipm_min_price` (editable, saved back under the same rule as Sell A–D price update) | Server + Qt |
| P10 | Charges on the item line like Sale Entry | Line columns Chrg before tax, Chrg after tax, Landing cost, Landing cost incl. GST — same as PO bcd844a / Sale Entry (ChrgBeforeTax / ChrgAfterTax) | Server keys + Qt |
| P11 | No line-wise calculation when qty / rate changes | BUG — find why the line figures do not refresh after an edit and fix it | Qt and/or server |
| P12 | Button: GST-wise break-up | A button that pops up the bill's GST break-up by rate: Taxable, CGST, SGST, IGST, Cess, Total per rate + total row | Server summary + Qt |
| P13 | Remove title "HEADER" | Drop the "THE HEADER" section heading (and its empty space) | Qt |
| P14 | Totals frame in a different style | Same style as the PO now: totals strip beside the charges table (`PurchaseTotalsStrip::setGridColumns`) | Qt |
| P15 | Focus and tab order in PO and Purchase | Enter/Tab walks the header fields in visual order, then into the first line cell; Esc/Tab out of the grid goes to charges, then totals/save; same on the PO | Qt |

## Defaults chosen (tell the user; change if they object)
- Qt only; web untouched.
- Markup B–D on the same basis as the existing Markup A.
- MSP saved back to the item under the same setting that already governs Sell A–D updates.
- Totals: copy the PO's 11-02 layout.

## New grid columns → ui_table 56
Every new line column must be appended at the END of `PurchaseInvoiceCols` (order = ui_table column
numbers) and then registered in ui_table 56 by db-schema. Also register PO cols 23–27 in ui_table 57
(open item from the PO job).
