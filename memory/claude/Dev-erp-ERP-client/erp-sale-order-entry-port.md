---
name: erp-sale-order-entry-port
description: "Sale Order Entry screen (2026-08-11) — reuses quotation engine/components; tender contract traps, server-owned-by-omission rule, grid ids, backend gaps"
metadata: 
  node_type: memory
  type: project
  originSessionId: e714d21c-acd4-48f2-bbeb-aa43dce86853
  modified: 2026-08-11T14:44:37.382Z
---

Sale Order Entry shipped 2026-08-11: `features/sales/sale-order/` + route `/sales/sale-order`, modeled on [[erp-quotation-entry-port]] and reusing its pricing engine, grids (ItemGrid/ChargeGrid/GridCell), pickers, fields, and parameterised quotationApi endpoints. New: `tender/` domain (arithmetic, instruments, rows), CreditPanel, TenderDialog (`purpose="advance" | "settlement"`), quotation import (Ctrl+F3 → source trail + derived customer lock).

**Why:** the server contract has traps the plan document got wrong, verified against the live server + DB.

**How to apply:**
- `advances[]` does NOT exist on `POST /sale-orders/create` despite the server README — `forbidNonWhitelisted` 400s it. Server-owned fields (soAdvance*, soFulfilStatus, soBilled/Pending/CancelledAmt, per-line soiDelivered/Cancelled/PendingQty + soiLineStatus) are enforced untouched **by omission from the DTO type**, not by echoing.
- GET and DELETE `/sale-orders/*` need all FOUR of soId+soCompanyId+soBranchId+soAccYear (uuid v7 pipes).
- There is no `soi_bill_qty`: `soiOrderQty` = total units promised (fulfilment equation runs on it); the keyed loose qty is reconstructed on load from net/case/factor (`soiToBaseFactor` IS persisted, unlike the quotation).
- Tenders: `tdTotalAmt` must equal round(amount+surcharge,2); `tdCardLast4` exactly 4 digits or null (cheques null); type 5 = CHEQUE needs tdRefNo+tdInstrumentDate ≥ order date; surcharge NEVER settles the document — `tender/arithmetic.ts` computeTenders is the single source (soPayStatus from the NET; the Qt gross-ceiling bug deliberately not ported).
- `GET /tender-masters/list`: no tenant scope, returns INACTIVE rows — client filters tndIsActive + effective window vs the DOCUMENT date. Nullable tndNeedsRef/AllowChange/AllowInReturn = inherit from type; type-default table lives in `tender/rows.ts` (11 rows incl. VOUCHER; no REST list for tender types).
- `GET /master-lookups/party-credit?partyId=` (camelCase; partyId = cus_id; 404 for unknown): real fields are pendingAmount/overdueAmount/maxOverdueDays/creditAmtLimit/availableCreditAmount(null when check off)/isAmtLimitExceeded/isCreditCheckEnabled — no status text, no available_balance fallback anywhere.
- Grids: item ui_table **24** (96 cols exactly, fulfilment quartet at 92–95, widths are Qt-percents not px), charges = shared table 21, list = grid **87** (binds only icompany_id/ibranch_id/ifrom_date/ito_date; projects neither so_src_doc_type nor so_is_deleted), item picker = shared grid 71 (**no iallow_so filter exists server-side** — backend gap).
- Wire mix on one GET payload: header/item decimals = strings, tender/charge money = numbers, tender date-only = yyyy-mm-dd.
- Cross-feature exports added for reuse: quotation.payload `chargeDto`/`chargeFromPayload`, quotation.utils `resolveItemColumnsWith`.

Nav (2026-08-12): the header menu is server-driven (`/menu-masters/usermenu`); hrefs attach by normalized-label match against `DEFAULT_PRIMARY_MENU`. The DB names it **"Sales Order" (singular, menu 11, direct child of "&1 Sales")** — the static entry must use that exact label or the item renders dead. Guarded by `sale-order.menu.test.ts`.

Legacy UI pass (2026-08-12), both screens matched to supplied mockups:
- **List** = `CrudMasterPage` shell (like the quotation's), grid 87 columns, From/To/Period window (90-day default), status pills, cancelled rows struck through (`so_status='CANCELLED'` is the only deleted signal grid 87 exposes). Three shell props added for it: `searchPlaceholder`, `buildDeleteRequest` (compound-key DELETE — an order needs all four scope fields), `rowClassName` (forwarded to ReusableTable, which already supported it).
- **Entry** = uppercase title + icon strip + FOUR header columns (customer · order · sales/delivery · credit-as-read-only-fields) + Advance/Refund/Balance Due inside the shared TotalsStrip via new optional `extraColumnOne`/`extraColumnTwo` props.
- Footer/shortcuts follow the legacy set: **F5 opens Tender (does NOT save), F6 Save & Print is the only save path**, F8 list, Alt+Y copy, F11 last order, F2 edit, F3 delete, F7 clear. No Hold on this screen.
- "Beat" renders read-only from the customer: `sale_order` has **no area column** (unlike `sq_cust_area_id`) — wiring it needs a migration. "Term" is `soOrderType` (CASH/CREDIT). Packed By = `soPackedId`, Loyalty = `soHasLoyalty` (both real DTO fields; the uuid[] pair must go as [] not null).

- Tender/advance opens on a NEW entry too (no items yet): `computeTenders` only computes change when the document amount is > 0, so a counter deposit settles in full instead of being handed back as change, and `payStatusOf` never reads PAID against an unpriced document. Save re-judges tenders against the finished total.

Tender dialog full port (2026-08-12), `features/sales/sale-order/tender/` — arithmetic + rows + validate are pure and tested (44 cases), the dialog computes and judges nothing:
- **base vs amount**: cashier keys the base; `amount = base + surcharge` is what is charged. Surcharge is the bank's cut — it nets out of the balance and EVERY comparison against the document uses the base (a card with a fee must settle a full bill). Flat fee only applies once `base > 0`.
- **Deviation from the plan, deliberate**: the plan keeps change as a separate `refundAmt` on top of the keyed base; this backend seeds `acc_bill_balance`'s ADVANCE from `Σ td_amount` and never sees the change, so change is taken OUT of the row's base instead (`td_amount` = kept, `td_change_amt` = returned, `td_received_amt` = handed over). Document net identical; only where the subtraction happens moves.
- Nullable `tndNeedsRef`/`tndAllowChange`/`tndAllowInReturn` = inherit the TYPE default (never `Boolean(x)`); **CASH always gives change** whatever the master says.
- Bank field = **dropdown 46 "INDIAN BANKS LIST"** (`fixed.bank_master`, 166 rows) — SQL selects `bnk_name` twice, so there is NO id: value and label are both the name, which is what `td_bank_name` stores.
- **`sales.allow_excess_tender` does NOT exist in `public.app_setting_def`** on this deployment (checked) — gate 3's exemption is a passed-in flag defaulting to false.
- Daily limit is deliberately NOT checked client-side (server-only; a wrong warning is worse than none). RRN/LOYALTY rows render read-only until their panels land (plan phases 5–6); they are excluded from advance mode anyway.
