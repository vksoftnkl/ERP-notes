# Sales transactions — Qt client plan, screen by screen (P5)

**2026-09-20, all defaults.** What the client does for the six documents
once `plan-backend-sales.md` P2–P4 are up. Read against the client as of
commit `08317a1` (`src/modules/sales/{salebill,saleorder,quotation}`,
`src/modules/sales/common/txn_doc_types_sales.cpp`, `src/shared/{charges,
adjustments,promotions,txn}`, `src/modules/accounts/receipt` as the
posting-screen reference).

**House rules that apply** (memory: accounts-screens-own-ui-rule,
widget-signal-wiring-rule, grid-delegate-edit-pattern, nextable-grid-
traps): every transaction screen is its own `.ui` on `NexDialog`; lists
are `TxnMainView` descriptors, never a second list widget; grids edit
through the delegate's `cellEdited`; only user-intent signals are
connected; every new grid is registered in `fixed.grid_details` +
`grid_columns` (memory: ui-tables index-shift trap when adding a column).

**Mockups first** (memory: receipt-design — "how the user likes to run a
design"): the three new screens (DC, DC Return, Sale Return) get a PNG
mockup (`*_ui_mockup.py` beside it) BEFORE any `.ui` is drawn, and the
bill screen's new strips get one PNG showing the four additions. Owed at
the start of P5, not in this file.

---

## 0. Ids to reserve (confirmed by the user 2026-09-20)

Live ceilings 2026-09-20: menu 250, grid 112, dropdown 54, ui_table 37.

| kind | id | name |
|---|---|---|
| menu | 251 | Delivery Challan Return (parent 1, beside 182) |
| menu | 252 | Transporters (parent = masters) |
| menu | 253 | Vehicles |
| menu | 254 | Statutory Limits (accounts setup; read + company override) |
| menu | 257 | Temp credits (sales; the who-owes list) |
| grid | 113 | MAIN LIST - DELIVERY CHALLANS |
| grid | 114 | POPUP - OPEN CHALLANS FOR BILL (line level) |
| grid | 115 | MAIN LIST - DC RETURNS |
| grid | 116 | MAIN LIST - SALE RETURNS |
| grid | 117 | POPUP - BILL LINES FOR RETURN |
| grid | 118 | POPUP - OPEN ORDERS FOR BILL (multi-order) |
| grid | 119 | POPUP - TRANSPORTERS |
| grid | 120 | POPUP - VEHICLES |
| grid | 121 | MAIN LIST - E-INVOICE REGISTER (menu 152) |
| grid | 122 | MAIN LIST - E-WAY BILL REGISTER (menu 153) |
| grid | 123 | POPUP - PARTY CREDIT NOTES |
| grid | 131 | MAIN LIST - TEMP CREDITS · 132 POPUP - TEMP CREDIT BY MOBILE (receipt picker) |
| grid | 133 | POPUP - RECENT BILLS FOR RE-TENDER (§1.7b — this device + session, today) |
| dropdown | 55 | transporters (bare `i` params, read-only role — memory: track-preset-dropdown) |
| dropdown | 56 | vehicles |
| dropdown | 57 | DC purposes (from `comp_dc_purposes`) |
| ui_table | 38 | DC items · 39 DC return items · 40 sale return items · 41 transporter master · 42 vehicle master |

Menus 11 (order), 12 (bill), 13 (return), 14 (quotation), 182 (DC —
"Deliver Note" renamed "Delivery Challan"), 226 (delivery update), 152 /
153 (registers) exist. `user_menus` rows for every user who may post
must carry the four new flags (memory: user-menu-full-replace — the
users screen must send all ten).

---

## 1. Sale Bill (menu 12) — `sale_bill_entry.*`, changes only

The screen exists (5,000 lines, hold, autosave, import quotation / order,
tender dialog, F4 adjustments, promotions binder). What changes:

### 1.1 Post / Cancel / Amend — the receipt's verbs

* **Quick-action strip** (`hboxQuick`, 3.0's tool strip — 19 buttons exist,
  13 of them "coming soon") keeps its place; it gains **Challan import
  (Ctrl+F5)** and **Delivery** (menu 226 status), and e-Inv / e-Way become
  Shift+F2 / Shift+F3 to the registers. Keys F2–F11 and Alt+Y belong to the
  existing verb bar (Hold F9, Pick Held F10), so the new verbs follow the
  receipt screen: **Post & Print Ctrl+Enter, Post Ctrl+Shift+Enter,
  Validate Ctrl+F7** — the same on every posting screen.
* **Save (F5)** = `/bills/create` (draft). **Post (Ctrl+Shift+Enter)** = `/bills/
  validate` → show `warnings[]` in the receipt's warning strip with a
  checkbox per code the user may override (`rights.override`) → `/bills/
  post` with `overrides[]`. **Post & Print** = `printAfter`.
* A loaded POSTED bill opens READ-ONLY (`setScreenReadOnly`, exists)
  with **Cancel** and **Amend** on the identity strip, greyed with the
  reason from `/bills/get`'s `locks` (returns / allocations / IRN / EWB /
  rights). Cancel = reason dialog (mandatory) → `/bills/cancel`. Amend =
  the screen unlocks, `baseRevision` remembered, Save becomes
  "Post amendment" → `/bills/amend` with `editRemark` (dialog).
* Delete (exists) → only on a DRAFT; the button hides otherwise.
* Hold / Held list (exist: `btnHold`, `btnPickHeld`, txn_hold grid 96) stay
  as they are; a held bill is a DRAFT snapshot and is never posted.
* `sales.auto_post` (AppSession) → Save runs create + post in one go
  (POS behaviour).

### 1.2 Mode

`sb_bill_mode` from the launching menu: menu 12 opens WHOLESALE; the POS
screen (later, its own `.ui`) opens POS. Not a combo on the screen.

### 1.3 Bill against a challan / several orders

* **Ctrl+F5 Import challan** → `TxnHoldPicker`-style popup on grid 114
  (`/bills/open-sources?kind=DC`), line-level ticks, quantity editable
  ≤ open; lines land with `sbi_src_doc_type 'DELIVERY_CHALLAN'`,
  `sbiSrcItemId`, lot, rate (re-price → `sales.bill_against_dc_reprice`
  WARN chip on the line). The source chip (`updateSourceChip`, exists)
  lists every DC / order touched.
* **Ctrl+F4 Import order** (exists) → becomes append-capable: grid 118,
  same customer only (`sales.multi_order_bill`), lines from several
  orders; `applyImportedOrder` merges instead of replacing.
* Red flag on a DC line older than `DC_RETURN_WINDOW_DAYS` (server says
  so in the popup rows).

### 1.4 Charges strip (Q37)

`ChargeGridController` (shared) gains a "from order" column: for each
order charge the `/bills/validate` proposal shows source amount, carried
so far, proposed now, basis combo (PRORATA / FULL / MANUAL / NONE);
editing the amount sets MANUAL. Rows carry `cdSrcCdId` / `cdSrcAccYear` /
`cdCarryBasis` on save.

### 1.5 Party panel (Q47, Q50, Q40, Q20, Q25)

`/bills/party-context` on customer pick (replaces `fetchPartyCredit` +
`fetchCustomerDetail` — one call): credit position (limit / used / open
bills / oldest days, red when over), loyalty member + balance + max
redeem, open advances + credit notes count (the F4 panel — shared
`PartyAdjustPanel`, exists — now lists credit notes too, grid 123), PAN /
Form 60 state. **BILL-TO band** (as printed): name, address, place, PIN, phone, GSTIN
= the `sb_cust_*` snapshot. For the walk-in customer
(`sales.default_customer_id`) the band is OPEN — the operator types the
name the customer wants on the bill; it is printed and kept on the bill
only, no customer row is made, the ledger stays the walk-in customer. For
a listed customer the band is filled from the master and locked until
"Edit for this bill" (a one-off correction for THIS bill — the master is
not touched; the customer screen is for that). A typed GSTIN turns the
register row B2B (IRN if the company e-invoices) — the server validates
the shape. **Quick-add customer** (`+` beside the customer combo):
name, phone, GSTIN, area → `/customers/create` (writes the ledger,
memory: customer-creates-ledger) → selected.

### 1.6 Ship-to / dispatch / transport band — SAVED on the bill

**The band is NOT on the bill any more — it is one shared table.** `33`
creates `public.txn_transport_detail`, 1:1 with any goods-moving document, and
`30 §1b2` drops `sbDriverId / sbSupervisorId / sbVehicleId / sbVehicleNo /
sbLoadmanId` off the header (migrating the rows first). The same band serves
the bill, the challan and both returns, so the screen builds it once and the
four screens reuse the widget.

`driverId`, `supervisorId`, `loadmanId[]`, `vehicleId` and `vehicleNo` stay
HEADER fields — they are filled for a home delivery that has no consignment at
all. The rest is a nested `transport` object:
`{ direction, from:{godownId, branchId, addrId, name, addr, place, pin, phone,
stcd, gstin}, to:{…}, mode, transporterId, transporterName, transporterGstin,
lrNo, lrDate, distanceKm }`, and the screen may send it as null. **`direction` is OUTWARD on the bill and the challan, INWARD on
both returns** — on an outward document `from` is our godown and `to` is the
customer; on an inward one they swap. The e-invoice `ShipDtls` / `DispDtls`
and the e-way Part-A/B are built from this one object and never re-typed. `btnQuickShipping` ("coming soon" today) opens / focuses the band.

A collapsible band under the header: ship-to picker (grid 110 addresses,
memory: ship-to-addresses-screen; typed override), dispatch godown
(default the branch's), transporter (dd 55) / vehicle (dd 56) / LR no +
date / distance / mode. Mandatory only when the server's e-way rule
says so (`/validate` returns `SALES_EWAY_TRANSPORT_MISSING` as a WARN
that cannot be overridden → the band opens itself).

### 1.6a Locking — what the screen greys, and when

**Two states, and the second is absolute: once it has been declared, nothing on
this screen is editable.**

| # | state | read-only | still editable |
|---|---|---|---|
| 1 | POSTED | header, item grid, charges, tender | **the transport band** — nothing has been declared yet |
| 2 | **IRN or e-way bill GENERATED** | **everything, band included** | nothing |

At state 2 **Amend is gone**, not greyed-with-a-reason-to-override: a declared
document is corrected by **cancelling it** inside the portal's window and
raising a fresh one, and past that window a bill is corrected by a credit note.
`rights.amend` must not light the button on its own — the right says the USER
may amend, lock 2 says this DOCUMENT may not.

**A later vehicle change is not an edit.** The lorry breaks down mid-route:
that is **Part-B vehicle update** (§1.8) going to the portal via
`/gst/ewaybill/vehicle`, recorded in `gdw_`. It never re-opens the band.

**So once an e-way bill exists, show the vehicle from `gdw_vehicle_no`, not
from the document.** The document keeps the original; `gdw_` has the one
actually carrying the goods. Same for the transport doc no and the mode. This
is the single easiest thing to get wrong on the read side.

**The client NEVER computes any of this.** `/bills/get` returns
`locks.editable { document, transportBand }` and the screen greys from those
two booleans. Deriving it client-side puts the rule in two places and they
disagree the first time a setting or the law changes — and the server refuses
the write anyway, so a client that guessed wrong turns a grey field into a
confusing error.

**Saving the band does NOT go through Save or Amend.** Between state 1 and
state 2 the document is frozen while the band is not, so the band has its own
verb: `PUT /bills/transport`, enabled by `locks.editable.transportBand`, as a
**Save transport** button inside the band. It does not bump the revision.

**The other three screens differ — do not copy this table blindly.** Each has
its own, written where the screen is: §4 the challan (never has an IRN, so only
the EWB trips lock 2 — but it has a *third* editable flag until then,
`purpose`), §5 the DC return (no `/amend` at all, ever, declared or not), §6
the sale return (the IRN is a CREDIT NOTE, plus `SALES_CN_APPLIED` and
`GST_CREDIT_NOTE_CUTOFF` on lock 1).

### 1.7 The tender dialog (Settle, F5) — audit 2026-09-20, nothing to alter

`sale_bill_tender.*` already has: one row per tender master with hotkeys,
change on cash, the CREDIT confirm for a credit customer, the card panel,
the loyalty panel (points at the tender's conversion rate, min points),
the F4 `PartyAdjustPanel` (advances + credit notes → `ADVANCE_ADJUST` /
`NOTE_ADJUST` through `bill-adjustment.helper.ts`, which exists), totals,
Save (F5) / Save & Print (F6). Tables `acc_tender_detail` / `_master` /
`_types` need no ALTER. What it gains (picture:
`sale_bill_tender_additions_ui_mockup.png`):

* **A** cash-limit line under CASH — cash today from this party (from
  `party-context`) + this bill vs `CASH_TXN_LIMIT_269ST`; REFUSE shown
  before the post refuses it.
* **B** PAN / Form 60 box when a cash sale crosses
  `PAN_REQUIRED_CASH_SALE` → `sbCustPan` / `sbForm60Ref`.
* **C** loyalty row: points ↔ amount both ways, capped by the scheme
  (max %, min / max points, min bill, multiple) from `party-context`; earn
  preview. **`m_loyaltyRate` and the min-points come from `party-context`
  (the SCHEME), not from the tender master** — today the panel reads
  `master.conversionRate` / `master.minAmount`; that becomes the fallback
  only. The walk-in shows "no member — add the customer to earn".
* **D** daily-limit hint per tender (`tnd_daily_limit` is stored, never
  enforced — the server will, WARN + override).
* **TEMP_CR row** → the TEMP CREDIT panel (name, mobile, place, pay-in
  days → due date, notes; "copy from bill-to"), mandatory before Save; the
  WARN/REFUSE from `/validate` shown inline (max days, max amount, an open
  temp credit on the same mobile with its balance).
* **CREDIT row** note: the amount is the outstanding — it goes to
  `acc_bill_balance`, nothing else; the receipt collects it.
* `sales.tender_type` (exists: none / cash_bills / all_bills) decides
  whether F5 opens the popup at all — shown on the picture, no new setting.
* RRN row retired: a credit note is an adjustment, never a tender
  (memory: party-adjustment-design). `Purpose::Refund` (CR mode) added for
  the return screen; only masters with `tnd_allow_in_return`.

### 1.7b Re-tender — the counter's most common correction

**The scenario, many times a day:** the bill is rung up, the customer says UPI,
the operator marks UPI and moves to the next customer. The UPI then fails and
the customer pays cash. The SALE was right; only the PAYMENT changed. Today
that forces an edit or a cancel of a posted bill, and both are wrong.

**Ctrl+F6 opens the TENDER FORM ONLY. It never loads the bill.**

That is the whole point of the feature. By the time the customer comes back the
operator is already keying the next bill, so loading the old one would throw
away their half-typed work and put an item grid in front of someone who only
needs to fix a payment. `sale_bill_entry` is not touched, not cleared and not
navigated — whatever is on the screen stays exactly as it is, and the dialog
sits over it.

```
  Ctrl+F6
     │
     ├─ picker (grid 133, POPUP - RECENT BILLS FOR RE-TENDER)
     │     this device + this session, today, newest first:
     │     bill no · time · customer · amount · how it was tendered
     │     …plus a bill-no box for anything older
     │
     └─ the tender form for the picked bill  →  POST /bills/retender
```

The picker is there because the operator says *"the one about two minutes
ago"*, not a bill number — and it may be two or three bills back if others were
served in between. Default the selection to the most recent; in the common case
it is Ctrl+F6, Enter.

**The dialog loads from `GET /bills/tender-context`, not `/bills/get`** —
bill no, date, customer, total, the live tender rows, and whether re-tender is
allowed at all. It does **not** fetch items, charges, party context or
transport. It is a small read on the hot path of a busy counter.

**The form itself** is `sale_bill_tender` in re-tender mode:

* each existing row gets a **Void** tick and a reason combo — UPI failed · Card
  declined · Cheque refused · Keyed wrong · Customer changed;
* below them the normal tender rows, to key what really happened;
* the footer shows **Voided** and **New** totals side by side, and OK stays
  dead until they match. The server refuses a mismatch
  (`SALES_RETENDER_AMOUNT_MISMATCH`) but the operator should see it first;
* OK sends `POST /bills/retender` (HANDOVER §2.13) and closes. No navigation,
  no reload of the entry screen.

**It works on a POSTED bill, and on one that already has an IRN.** The tender
is not on the e-invoice, not in GSTR-1 and not on the e-way bill, so lock 2
(§1.6a) does not reach it. This is the one thing a declared bill still allows,
so the screen must NOT grey Ctrl+F6 from `locks.editable.document`.

**Enabled by its own right, `um_can_retender` — not `amend`.** A till operator
has to fix their own counter in thirty seconds without being able to edit
sales. The users screen must send it with the other flags: granting a menu is a
full replace.

**Two tender types are not simple swaps, and the dialog must show it:**

* voiding a **LOYALTY** row gives the points back (the server reverses the
  REDEEM rows) — show the restored balance, or the customer is silently robbed;
* voiding a **TEMP_CR** row clears the temp-credit entry behind the bill's
  balance (menu 257).

**Refused after day close** — the drawer has already been counted with the
wrong figure, so that is a day-book correction, not this. Say so in the hint
bar rather than leaving the button dead with no reason.

### 1.7a Cash sale identity (Q39, Q40)

The tender dialog (`sale_bill_tender.*`) shows the cash-limit line
(from `/validate`) and, over the PAN limit, a PAN / Form 60 box that
fills `sbCustPan` / `sbForm60Ref`.

### 1.8 IRN / e-way bill — generated FROM the bill (3.0's tBtnEWay / e-invoice on the bill)

On a POSTED bill the identity strip shows the two badges (IRN: NA /
PENDING / GENERATED / FAILED / CANCELLED with ack no + time; EWB: number +
valid-upto) from `/bills/get` `posting`, and the quick strip's **e-Inv**
and **e-Way** buttons act on THIS bill:

* **e-Inv** → `POST /gst/einvoice/generate?sync=true` with the bill keys;
  greyed when GENERATED or not applicable (B2C, company not e-invoicing);
  on FAILED the mapped message shows and the button becomes **Retry**;
  **Fetch** (`/gst/einvoice/get`) when the outcome is unknown. Cancel IRN
  is NOT here — the bill's own Cancel / Amend does it inside the window
  (flow §5.10), the register (152) for anything else.
* No vehicle on the bill → "Generate the IRN without an e-way bill?" (3.0's
  msg 462) and the IRN goes alone; the EWB follows when the band is filled.
  The NIC field pre-checks (`GST_NIC_FIELD_*`, lengths / PIN / GSTIN / HSN /
  UQC) show in the warnings strip at Validate, so the operator fixes the
  master before the call is ever made.
* **e-Way** → `POST /gst/ewaybill/generate?sync=true` (by IRN when one
  exists); if the transport band is incomplete the band opens with the
  missing fields marked (`GST_EWB_TRANSPORT_MISSING`); once generated the
  same button offers **Part-B vehicle update** (`/gst/ewaybill/vehicle`)
  and **Print EWB**; below the value limit it says so and still lets the
  user generate by hand.
* At Post, `sales.einvoice_auto_fire` / `sales.eway_auto_fire` (BRANCH)
  enqueue the same calls — the buttons are for the by-hand and the retry.
* Shift+F2 / Shift+F3 open the registers (152 / 153) on this row for
  cancel, extend, bulk and the log.

### 1.9 Loyalty in the tender dialog

See §1.7 C.

### 1.10 Weight barcodes (Q48)

`SalesItemDelegate` item-code entry: when the scan matches
`sales.weight_barcode` (prefix / lengths), split into item code + value,
value → qty (WEIGHT ÷ divisor) or → rate (PRICE); then the normal item
lookup.

### 1.11 Delivery status (menu 226)

Own small list screen on `TxnMainView` (grid = 108's shape filtered to
`sb_delivery_status NOT IN ('NA','DELIVERED')`) with the action buttons
(Verify when `sales.require_verification_before_dispatch`, Packed,
Dispatched, Delivered) → `/bills/delivery-status`; 3.0's "sales validation
screen" is the Verify column of this list. The bill's read-only view gets
**Edit remarks** (`/bills/update-remarks`, text only). Only when
`sales.delivery_status_tracking`.

---

### 1.12 Temp credits list (menu 257 — `src/modules/sales/temp_credit/`)

`TxnMainView` descriptor over `accounts.acc_temp_credit` (grid 131 MAIN
LIST - TEMP CREDITS): date · bill · name · mobile · place · lent · balance
· due · days overdue (red past due) · status badge (text-derived) · user.
Filter row: status (OPEN + PARTIAL default), overdue only, search on
mobile / name / bill. Row actions (panel dialog, cheques style): **Open
bill**, **Receive** (opens menu 99 on the walk-in ledger with the bill
pre-ticked — the receipt's `?mobile=` path), **Follow-up** (promise date +
remark), **SMS / WhatsApp reminder** (later). No create here — rows are
born in the tender popup. Picture: `temp_credit_list_ui_mockup.png`.

## 2. Sales Order (menu 11) — `sale_order_entry.*`, changes only

Post (F10) → `/sale-orders/post` (CONFIRMED; the WARN list of shortfalls
shown, never blocking). A CONFIRMED order opens read-only with Cancel /
Amend as §1.1 (`locks.delivered` greys Cancel). The order's fulfilment
band (delivered / pending per line, from `/get`) exists as columns —
keep. Charges are entered here as today; they become the SOURCE rows.

---

## 3. Quotation (menu 14) — no change (Q22)

CONVERTED badge already text-derived.

---

## 4. Delivery Challan (menu 182) — NEW `src/modules/sales/dc/`

```
delivery_challan_entry.ui / .h / .cpp     the screen (NexDialog)
dc_item_delegate.h / .cpp                 item grid delegate (ui_table 38) — SalesItemDelegate subset:
                                          item, unit, godown, lot/batch, qty, free, rate, disc, tax (read-only), amount
dc_entity.h                               header + line structs ↔ JSON (save-delivery-challan.dto shape)
txn_doc_types_sales.cpp                   + deliveryChallan() descriptor (grid 113, status badges DRAFT/POSTED/CANCELLED,
                                          fulfilment badge OPEN/PARTIAL/BILLED/RETURNED/CLOSED, red age flag)
form_sales_menu.h                         case 182
```

The `.ui`, top to bottom: identity strip (no / date / purpose dd 57 /
user ref + date / price level) · customer (dd 39 + quick-add) with
bill-to snapshot · ship-to band (§1.6, always open — a DC is transport)
· source order chip (Ctrl+F4 import order lines, line-level) · item
grid · totals footer (qty, weight, taxable, tax, DC value — shown, not
posted) · Save / Post / Post & Print / Cancel / Amend / Hold (txn_hold,
doc type DELIVERY_CHALLAN exists).

**Locking (§1.6a):** a challan never carries an IRN, so only the **e-way bill**
trips lock 2 here — and when it does, everything freezes including the band.
Until then a POSTED challan has **two** editable surfaces where the bill has
one: the transport band (`PUT /delivery-challans/transport`) **and the
purpose**, via Convert. So the screen greys from
`locks.editable { document, transportBand, purpose }` — three flags here.

`purpose` goes false at lock 2 (the purpose became the e-way bill's sub-supply
type and has been declared) and also once any line is billed or returned,
because the purpose decided what those became.

This is the screen where the vehicle really changes en route, so it is the one
where **the read rule matters most**: once the EWB exists show
`gdw_vehicle_no`, not the challan's. Part-B updates go to the portal and never
re-open the band.

Behaviour: Post → `/delivery-challans/validate` → `/post`; EWB badge as
§1.8. Purpose other than SUPPLY greys "bill against" in the bill popup
until converted (a Convert button on the read-only DC: purpose →
SUPPLY, logged). Cancel greyed by `locks.billed / returned / ewbLive`.

---

## 5. DC Return (menu 251) — NEW `src/modules/sales/dc_return/`

```
dc_return_entry.ui / .h / .cpp            small screen: pick the DC (grid 113 in Pick mode), lines from
                                          /dc-returns/open-lines (grid 117's shape: dc qty, billed, open, return qty,
                                          condition, bucket, godown), the shared transport band (33, direction INWARD,
                                          collapsed) — receive godown is required for this document,
                                          reason (stock reason popup 102), remarks · Save / Post / Cancel
dc_return_delegate.h / .cpp               ui_table 39: return qty ≤ open, condition → bucket default
txn_doc_types_sales.cpp                   + dcReturn() (grid 115)
```

No amend (cancel + re-enter). Opens from the DC's read-only view too
("Return un-billed goods" button) with the DC preselected.

**Locking (§1.6a) — the strictest of the four screens.** `locks.editable
.document` is **false the moment it is POSTED and never true again**: there is
no Amend verb on this screen at all, declared or not. No IRN either (a DC
return moves goods and raises no debt, `DocDtls.Typ` CHL), so only the inward
e-way bill trips lock 2. Between POST and that, the band
(`PUT /dc-returns/transport`, `direction` INWARD) is the only editable surface
it has; after it, none.

---

## 6. Sale Return (menu 13) — NEW `src/modules/sales/sale_return/`

```

**Locking (§1.6a) — the bill's two, with the IRN being a CREDIT NOTE.**
Lock 2 fires on `DocDtls.Typ` **CRN**, not INV, or on the inward e-way bill;
once it does Amend is gone and `/amend` answers `SALES_IRN_LIVE` /
`SALES_EWB_LIVE`. A declared credit note is corrected by cancelling it and
raising another. Two more refusals sit on lock 1 and neither is about GST, so
the hint bar has to say which:

* **`SALES_CN_APPLIED`** — the credit note has been set off against a bill or
  refunded. Money has moved; reverse it with a document, not by un-doing this
  one.
* **`GST_CREDIT_NOTE_CUTOFF`** — past the statutory cut-off for the original
  invoice's financial year, so no credit note and no IRN. The date comes from
  `fn_statutory_limit` per company — never show a hard-coded September.

Band verb: `PUT /sale-returns/transport`, `direction` INWARD.

**Transport band — the same widget as the bill's (§1.6), `direction` INWARD.**
It writes `public.txn_transport_detail` (33), not columns on `sale_return`:
`from` is where the customer handed the goods back (defaulting to the customer
snapshot), `to` is our receiving godown. The receive godown here is a **header
default that seeds the lines** — `sri_godown_id` on the line is authoritative,
because a return may split across godowns. The band opens itself when the
server's inward e-way rule asks for it, exactly as on the bill.
sale_return_entry.ui / .h / .cpp          the screen
sale_return_delegate.h / .cpp             ui_table 40: return qty ≤ returnable, free qty, condition, bucket, godown,
                                          rate (locked on a bill return, editable on a free return), lot (read-only)
sale_return_settle_dialog.ui/.h/.cpp      the three modes: CASH → tender rows (sale_bill_tender reused in CR mode);
                                          ADJUST → the receipt's allocation grid (receipt_bill_delegate reused) seeded
                                          original-bill-first; ADVANCE → nothing to fill, a confirmation line
sale_return_entity.h
txn_doc_types_sales.cpp                   + saleReturn() (grid 116; badges status + settle status)
form_sales_menu.h                         case 13
```

The `.ui`: identity strip · **Against bill** toggle (default on; off =
free return, `sales.free_return_allowed`, the header shows a FREE
RETURN chip) · bill picker (grid 108-style list of the customer's POSTED
bills, or scan the bill ref) → lines from `/sale-returns/bill-lines`
(grid 117) with ticks and return-qty edit · customer (locked to the
bill's; free return picks) · reason (popup 102) + remarks · item grid ·
totals (taxable / tax / return amount / COGS reversed shown for the
owner) · loyalty claw-back and promotion claw-back lines (from
`/validate`, read-only) · settlement summary (mode + amounts) with the
dialog button · Save / Post / Post & Print / Cancel / Amend /
**Return & re-bill** (Q28: post, then open menu 12 with the lines and
`sourceReturnId`).

Window WARN (`sales.return_window_days`), credit-note-cutoff WARN and
cross-branch note appear in the warning strip from `/validate`.

---

## 7. Registers 152 / 153 — `src/modules/sales/gst/` (see `../gst/plan-qt-gsp.md`)

Listed here only so the sales menu is complete: both are `TxnMainView`
descriptors over `acc_voucher_doc_register` joined to `_doc_einvoice` /
`_doc_ewaybill` (grids 121 / 122), with the actions Generate / Cancel /
Retry / Print QR (152) and Generate / Update vehicle / Extend / Cancel /
Print (153). Their design is the GST plan's.

---

## 8. Masters — Transporters (252), Vehicles (253), Statutory limits (254)

Three `NexCrudForm` screens (ui_tables 41 / 42 and a small one for
`statutory_limits`): transporter = name, GSTIN / enrolment id, mode,
contact; vehicle = number (upper, no spaces — the CHECK), type REGULAR /
ODC, own / transporter, capacity; statutory limits = the shipped rows
READ-ONLY with an "Override for this company" action that copies the row
with `stl_company_id` set — the accountant never edits a pack row.

---

## 9. Settings the client reads (`AppSession::applySettings` gains them)

`accounts.cogs_mode` (label only), `sales.allow_posted_amend`,
`sales.return_window_days`, `sales.free_return_allowed`,
`sales.return_settle_default`, `sales.backdate_mode`,
`sales.max_line_disc_perc`, `sales.max_bill_disc_perc`,
`sales.rate_below_min_mode`, `sales.credit_limit_mode`,
`sales.multi_order_bill`, `sales.dc_requires_order`,
`sales.bill_against_dc_reprice`, `sales.weight_barcode`,
`sales.delivery_status_tracking`, `sales.quick_add_customer`,
`sales.einvoice_auto_fire`, `sales.eway_auto_fire`. The client uses them
for UX only (which strips open, which buttons show); every rule is
re-checked by the server.

---

## 10. Endpoint table (client → server)

**`PUT /bills/transport`** (and the same on challans and both returns) is the
band's own verb — needed because `/amend` is refused once an IRN exists while
the band is still editable. It does not bump the revision. HANDOVER §2.12.

| screen | calls |
|---|---|
| bill | `/bills/create /validate /post /cancel /amend /delete /get /open-sources /party-context /delivery-status`, `/txn-holds/*`, `/customers/create` |
| order | `/sale-orders/create /post /cancel /amend /delete /get /pending-amount /cancel-lines` |
| DC | `/delivery-challans/create /validate /post /cancel /amend /delete /get`, `/txn-holds/*` |
| DC return | `/dc-returns/create /post /cancel /delete /get /open-lines` |
| return | `/sale-returns/create /validate /post /cancel /amend /delete /get /bill-lines` |
| registers | `../gst/plan-qt-gsp.md` |
| masters | `/transporters/*`, `/vehicles/*`, `/statutory-limits/*` (CRUD, grids) |

---

## 11. Tests (`tests/sales/`, offscreen — memory: qt-offscreen-test-harness)

Extend `run.sh` (links ninja's objects minus `main.cpp.o`):
`test_bill_post.cpp` (validate → warnings strip → post → read-only →
Cancel/Amend greying by `locks`), `test_bill_sources.cpp` (DC lines +
two orders on one bill, charge carry rows), `test_dc_entry.cpp`,
`test_dc_return_entry.cpp`, `test_sale_return_entry.cpp` (three settle
modes, free return chip, re-bill gesture), `test_weight_barcode.cpp`.
Both modal-closers skip the reason / remark dialogs by object name
(memory: cheque-panel-dialog). Live suites run at the END of each screen,
not per edit (memory: dont-test-every-change).

---

## 12. Order of building (P5)

1. mockups (bill additions, DC, DC return, return) — PNG first
2. bill: verbs (§1.1) + party context (§1.5) + IRN/EWB badges (§1.8)
3. bill: challan / multi-order import (§1.3) + charges strip (§1.4)
4. DC screen + descriptor + menu 182
5. sale return screen + settlement dialog + menu 13
6. DC return + menu 251
7. transport band on the bill (§1.6), cash identity (§1.7), loyalty
   tender (§1.9), weight barcodes (§1.10), delivery update (§1.11)
8. masters 252 / 253 / 254
9. registers 152 / 153 (with the GST plan)
