---
name: sale-bill-party-rules
description: "User's 2026-09-24 rules for the sale bill party area — beat is a customer FILTER (dropdown 58), bill-to typed only for the seeded default party, term follows the party and is locked unless sales.allow_payment_term_change (SQL 38, pending)"
metadata:
  node_type: memory
  type: project
  originSessionId: 2a57b2fc-d57a-4c6a-8daf-436af25c2f7f
  modified: 2026-09-24T05:14:14.334Z
---

Rules from the user's 2026-09-24 ten-point review of the sale bill. All built the same day:

- **Beat is a FILTER, never painted from the customer.** The customer combo uses
  dropdown **58 CUSTOMERS BY BEAT**, registered via `/dropdown-details/create`
  (payload in share/grids/dropdown_58_customers_by_beat.json). It is dropdown 39
  plus `('icus_area_id' = '' OR cus_area_id::text = 'icus_area_id')`, and the
  param is ALWAYS sent ('' = every beat). 39 is still used by the DC,
  quotation and order.
- **Bill-to text is editable ONLY for the default party** that the config
  seeds (`sales.pop_default_customer` on + `sales.default_customer_id`). For
  any other customer the fields are disabled and take no focus. The "Edit for
  this bill" toggle was removed. `applyBillToLock()` is the one writer and
  already includes the import lock.
- **Term = the party's own**: Credit if `cus_credit_allowed`, else Cash, applied
  on every customer lookup. The combo is locked unless
  **`sales.allow_payment_term_change`** (BRANCH, default false) is on. Its
  app_setting_def row is `share/38_setting_allow_payment_term_change.sql`, NOT
  yet applied on .106 (the claude role is read-only and there is no API to
  create a definition). Until someone applies it, the client default (locked)
  holds. An import keeps its source's term when switching is allowed.
- **Due Days** is an editable box that defaults to the customer's credit days.
  It shows only on Credit. A bill-date move keeps the typed days
  (`onTermChanged(false)`).
- The credit panel lost **Oldest Open** and **Last Bill**. An overdue bill is
  still reported in lblCrStatus. **Loyalty** is its own group (points + card).
  The facts lines no longer repeat CREDIT or LOYALTY.
- lblStockInfo and lblPartyFacts now sit above Remarks. lblPartyFacts is never
  written (paintPartyFacts clears it); asked whether to delete it.

Related: [[sale-bill-upgrade]], [[grid-runner-binding-rules]], [[shipping-dialog]]

**Follow-up the same day:** lblPartyFacts was DELETED at the user's request.
NexWarningStrip is now a ONE-LINE footer summary with a View button plus a
"Validation" popup that holds the rows. Every answer that has notes opens the
popup; `setNotes(notes, false)` paints only the summary. Post goes through
`confirmProceed(verb)` (Back / Post, and Back only when something is refused).
A 422 that carries notes no longer also shows handleError's message box. The
bill and DC both use it. The snap harness's modal-closer must skip the
"Validation" title.

**Header compaction, later that day:** grpCustomer and grpDelivery have 0
vertical spacing. The Packed By, Agent, Their Ref and Ref Date rows are HIDDEN
via `QFormLayout::setRowVisible`; they are not removed, so the user can show
them later. grpCredit has two merged rows (Outstanding | credit days, Limit |
avail), then Adv / CN (valAdvances | valCreditNotes) and Cash / Temp (valCashToday | valTempCredit) as read-only fields — lblFacts2 is GONE — then Loyalty. lblCrStatus is GONE (hybrid, 2026-09-24): Avail goes red on a limit breach and Outstanding on overdue, with the reason in the tooltip (only while the check is on). A one-time warning shows on an OPERATOR pick (m_creditAlertPending) only when the party is creditAllowed, the mode is REFUSE and it is already breached. Every other reason waits for the Validation popup. Loyalty was folded into the
panel as one row (points | card) and grpLoyalty no longer exists.

**Default customer (fixed 2026-09-24):** clearing `sales.default_customer_id`
in App Settings still seeded MADHAVAN. /effective returns a cleared override as
`value: null, source: OVERRIDE`. applySettings skipped every null, so the value
fell back to a customer uuid hard-coded as the compiled default in
app_session.h. Now a null OVERRIDE on a TEXT/UUID setting applies ''; a null
on a bool or number still keeps the compiled default. The compiled default
customer is empty. The catalog row's asd_default_value still carries the
test-box uuid (a DB matter, not ours).

**Item grid columns (2026-09-24):** ui_table **22** is used by the bill alone
(quotation 18, order 24, DC 38). It was trimmed via PUT
/ui-table-masters/layout-settings to 14 visible columns: Barcode (the scanner
writes there), Description, BatchNo, GodownName, Uom, Bill Qty, Sch, Rate,
DiscPerc, Taxable, Gst %, GstAmt, Total, Src Type. The before-state is in
share/grids/ui_table_22_before_sale_bill_trim_2026-09-24.json. **TRAP:**
/ui-table-masters/get returns uiTblClmNo as a STRING — compare int(), or every
column gets hidden (it happened for ~1 minute).
