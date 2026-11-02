# Purchase phase 1 — FRONTEND plan (Qt6 client)

Date: 2026-10-27. From: Viji / Claude. Built in `/home/vk/Projects/Qt6/NexERP` (main), step by step with the backend (`plan-purchase-backend.md`); each screen goes live against my API instance (:3012) before the next.

**Inputs:** the phase-1 plan + addenda 1 and 2 + the mockups (they are the visual spec — [[mockup-fidelity]]).

## 0. House patterns to reuse (do not reinvent)

| Need | Reuse |
|---|---|
| Entry screen frame | Opening Stock's `.ui` layout (title + scope + status pill + audit line; header card in 4 pairs; NexTable grid; footer strip; `frameButtons`) on `NexDialog` |
| Keys | `TxnKeys`: Ctrl+N / S / Enter, F2, Alt+X, Ctrl+H, Ctrl+O, Esc, F1, Ctrl+PgUp / PgDn (`/txn/adjacent`); purchase extras F12, F4, F9, Ctrl+L, Ctrl+A, Ctrl+P, Ctrl+F11 / F12, Ctrl+F8 |
| Grid entry | NexTable + a delegate per screen (`cellEdited`, never `itemChanged`), popup cells, `EnterNavigation` in popups, `setFocusStops` / focus-cell filter, queued barcode commit (never a modal inside a delegate commit), LineTicket for per-line replies |
| Lists | `TxnMainView` + new `txn_doc_types_purchase.{h,cpp}` (grids from backend B10) |
| Charges | `ChargeGridController({'P', docType})` + new On-bill / Payee columns |
| Hold | `shared/txn/txn_hold_*` (module PURCHASE) |
| History / audit | `TxnHistoryDialog` + `TxnAuditSpec` per doc |
| Transport | `txn_transport_band` / shipping dialog |
| Party adjust | `shared/adjustments/party_adjust_*` (return / advance set-off on the bill) |
| Stock pick (returns) | `StockPickDialog` pattern |
| Textile | Opening Stock's Size / Colour / Model columns + `takesBatchDetail()` + dd 70 / 71 quick-add |
| Errors | `api_error_text.cpp` plain-English map for the purchase codes |
| Theme | one block per screen in the DB template (next version after v40), published via the theme route |

## 1. Module layout (`src/modules/purchase/`)

```
common/      purchase_cols.h (shared line column enum for PO/GRN/INV/RET)   purchase_calc.{h,cpp} (mirror of the server calc; golden vectors)
             purchase_api.{h,cpp} (endpoint wrappers)   txn_doc_types_purchase.{h,cpp}   purchase_settings.{h,cpp}
             purchase_alloc_dialog.{h,cpp,ui}  (F12: batch rows + serial scan mode + textile cols)
             purchase_ratio_pack_dialog.{h,cpp,ui}   purchase_source_picker.{h,cpp} (GRN / PO / sister bill)
             purchase_supplier_context.{h,cpp} (chips + notes popup + pending PO offer)   purchase_item_context.{h,cpp} (last-rates strip)
             purchase_payee_split_dialog.{h,cpp} (coolie per loadman)   purchase_label_print.{h,cpp}
order/       purchase_order_entry.{h,cpp,ui} + delegate
grn/         grn_entry.{h,cpp,ui} + delegate
invoice/     purchase_invoice_entry.{h,cpp,ui} + delegate   payment_hold_dialog.{h,cpp}
return/      purchase_return_entry.{h,cpp,ui} + delegate
dues/        claims_shortage_view.{h,cpp,ui} (claims list + settle dialog; shortage receipt entry)
master/      supplier_item_list.{h,cpp} + popup   (supplier + item additions go into the existing supplier form / item card)
settings/    purchase_settings_view.{h,cpp}
```

Registered in `core/navigation/form_purchase_menu.h`:
- `fm.reg(23)` PO, `fm.reg(24)` invoice, `fm.reg(25)` return, `fm.reg(195)` shortage receipt;
- new ids for GRN, Claims, Supplier–Item Map, Purchase Settings.

## 2. The line grid (one column enum for all purchase docs)

`PurchaseCols`: Barcode, Item, Unit, Batch / Serial, Expiry, Billed, Recd, Dmg, Short, Free, Rate, Disc, Taxable, GST %, ITC, Tax, Total, Cost w/o, Markup %, Sell A, MRP, Check. Then the More columns (Gross, Disc ₹, Cess %, Cess / unit, Mfg, Godown, Weight, Sell B–D, Old prices, PO / GRN ref, Remarks) and textile (Model, Size, Colour, Bill line). Plus hidden ids (item, uom, tax, lot, line id, alloc json).

- **ui_table registration per screen** (share/grids script; upsert by id, never post columns twice without ids). Column chooser = ui_table visibility saved per user.
- **Column visibility rules:**
  - batch / expiry / serial by the item's track signature (Opening's helpers);
  - textile columns by `takesBatchDetail`;
  - tax columns hidden for composition / unregistered suppliers;
  - price columns read-only unless `allow_price_update` + right.
- **Row vs F12:**
  - one alloc row edits on the line;
  - more than one → the cell shows "n batches (F12)" / "n serials (F12)", read-only;
  - `purchase_alloc_dialog` owns the rows; both paths write the same alloc array.
- **Calc:**
  - the client calc gives instant figures;
  - before Post the screen calls `/purchase/invoices/validate`;
  - **the server's figures are painted back after save** (server is the authority);
  - golden-vector tests keep the two calcs identical.

## 3. Screens (build order = backend order)

| # | Screen | Key behaviour | Live test |
|---|---|---|---|
| F1 | Supplier form + item card additions; Supplier–Item map list + popup; bill-from GSTIN list on the supplier | MST-01 / 02 / 03 fields; deterministic ids handled server side | tests/purchase/masters |
| F2 | Purchase Settings view | 14 settings, per company | — |
| F3 | Purchase Order | pending / received derived; short close dialog; clone / merge; reorder suggestions; approval state; PDF / WhatsApp via the print dialog | tests/purchase/po |
| F4 | GRN | PO pick (Ctrl+L); challan / received / damaged / free; excess action; F12 alloc + serial scan; shelf-life warn; crew (dd 72); off-bill charges + payee split; labels after post | tests/purchase/grn |
| F5 | Purchase Invoice | source picker (direct / GRN / PO / sister bill); GRN-sourced lines locked; supplier context chips + notes popup; item context strip; ITC amounts; check column; printed-amount strip; payment hold + release dialog; cash mode tender strip; TCS; price setting (setting-gated) + "price kept" note; attachments; hold F9; Ctrl+F11 / F12 return from bill; Ctrl+F8 pay | tests/purchase/invoice |
| F6 | Shortage Receipt + Claims (one view, two tabs or two sections as the mockup) | tick open rows → receive (alloc); claims settle dialog (credit / short-settled / rejected; received as) | tests/purchase/dues |
| F7 | Purchase Return | source rows from bill / GRN; outcome per row; reasons (purchase set); settlement; supplier CN; e-way / vehicle | tests/purchase/return |
| F8 | Textile pieces: ratio-pack dialog, textile columns, label printing (print designer template), sale-bill scan check | tests/purchase/textile |
| F9 | Payment 100: payment-hold indicator + refusal; loadman payables; return / claim set-off | tests/money (existing) + new cases |
| F10 | Lists (TxnMainView) + reports RPT-01…11 (grids) | tests/purchase/lists |

## 4. Rules the screens must keep (from the plan / addenda)

- **Numbers:** paint the server's refno (no client-side shortening; `dev_code` makes them short).
- **Edit:** F2 only on drafts; posted docs follow §3.8 (paid → money fields locked, with a message naming the payment).
- **Price note** (D16): show the server's `price_update_result` per line ("Price not updated — changed later on … by …").
- **GRN-sourced invoice lines:** Recd / Dmg / Short / batches / serials read-only; no off-bill charge rows.
- **Branch:** an invoice from a GRN only offers that branch's GRNs (R2).
- **Offline:** the screen works the same; duplicate supplier bills are refused online and reported otherwise. Scanning an unknown barcode → "save as EAN" only when online.
- **Times** shown AM / PM; plain-English errors with the server line in grey.
- **Popups:** `EnterNavigation::install()`; never `setCellWidget`; NexDialog heap-allocated (WA_DeleteOnClose).

## 5. Tests

- `tests/purchase/run.sh`: offscreen, links ninja objects (the Qt test harness memory). Live against :3012 as VKPOS. ZT- data, cancels what it posts. `SNAP=<dir>` saves PNGs to compare with the mockups.
- Per screen (as the table above), plus the acceptance list (plan §9 + addenda) end-to-end through the screens.
- Calc parity: run the server's golden vectors through `purchase_calc` (offline test).

## 6. Order with the backend

| Phase | Backend | Frontend |
|---|---|---|
| 1 | B0 | — |
| 2 | B1 | F1, F2 |
| 3 | B2 | calc mirror |
| 4 | B3 / B4 | F3 |
| 5 | B5 | F4 |
| 6 | B6 | F5 |
| 7 | B7 / B8 | F6 / F7 |
| 8 | B9 / B10 | F8 / F10 |
| 9 | B11 | F9 |
| 10 | B12 | full acceptance |

Each pair is finished and tested live before the next starts.
