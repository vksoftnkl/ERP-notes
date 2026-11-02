# Sale Bill (Sales Entry, menu 12) — React port plan, REV 2

Scope: **the whole sales-entry surface.** That is the bill screen, its tender
(settlement) dialog, the advance/credit-note set-off panel, shipping, re-tender,
GST actions, hold/autosave, and the four lists the bill owns (bills, delivery,
temp credits, re-tender picker). It is written against the Qt screen **as it
stands on 2026-09-25, including the uncommitted working tree**, and against the
NestJS source in `nexerp-api-ro` read the same day.

**This supersedes** `../react-plans/plan-react-sale-bill.md` (2026-09-12) and,
for the bill, `../react-plans/plan-react-sale-tender.md` (2026-08-12). Both
predate the A3 backend (2026-09-22). They know nothing of validate / post /
amend / cancel / locks / rights / party-context / transport / re-tender /
temp credit / challan import / weight barcodes, and they get the API model wrong
(cancel is not `/bills/delete`; `/create` is not create-or-update of a posted
bill). §31 lists exactly what changed. The sale order still uses the shared
`<TenderDialog purpose="advance">` described in §15.

Qt source (7,470 + 1,753 + 328 lines, plus the shared parts):

    src/modules/sales/salebill/sale_bill_entry.{h,cpp,ui}     the screen
    src/modules/sales/salebill/sale_bill_tender.{h,cpp,ui}    settlement dialog
    src/modules/sales/salebill/sale_tender_delegate.*         tender amount editor
    src/modules/sales/salebill/sale_bill_retender.*           re-tender dialog
    src/modules/sales/temp_credit/*                           follow-up dialog
    src/modules/sales/common/*                                item grid cols/delegate, voucher
                                                              entity, price level, size, weight
                                                              barcode, open-sources, quick-add,
                                                              list descriptors (txn_doc_types_sales)
    src/shared/charges/*  src/shared/promotions/*  src/shared/adjustments/*
    src/shared/txn/tender_master_row.*  src/shared/ui/txn/{txn_shipping_dialog,nex_warning_strip}.*
    src/core/api/api_endpoints.h  (namespace Bills, BillKey)

Companion documents this plan does **not** repeat:

- **`../react-plans/plan-react.md`** owns the pure pricing engine
  (`src/domain/pricing`: `lines.ts`, `charges.ts`, `document.ts`), the
  `VoucherPolicy` snapshot and the `stored | priced` draft union. The bill
  imports it. §10 lists the only places the bill differs.
- **`../react-plans/plan-react-dropdown.md`** covers the searchable picker. §6.4
  lists three gaps the bill needs closed first.
- **`../react-plans/plan-react-printing.md`** is the data-driven print model. §26
  says what is stale in it.
- `plan-qt-sales.md` §1 (this folder) is the Qt spec the screen was built to.
  `HANDOVER_endpoints.md`, `sales_posting_flow.md` and `SIX_TXN_CONTROLS.md` are
  the server side.
- `NOTES_bill_live_test_2026-09-24.md` and `tests/salebill/run-full.sh` in the
  repo form the acceptance list (§29).

> **The whole screen in one line:** the client prices the bill and settles it,
> the server decides whether it may post. Every rule that can refuse a bill
> (credit, cash limit, PAN, stock, sources, loyalty, temp credit, amounts) is a
> `/bills/validate` answer painted on the screen, never a verdict recomputed
> in TypeScript. The client's own checks exist only so the operator hears
> about a zero quantity before a round trip.

---

## 1. What the sale bill is, and what it is not

It is a **document screen with a lifecycle**: DRAFT → POSTED → (amend →
POSTED rev+1) → CANCELLED. The number is given at post and never reused. A
cancelled bill keeps its number and stays on the list. It is also the busiest
screen in the product, keyed at a counter by someone who does not read
dialogs. Every choice below is weighed against that.

Seven facts shape everything:

1. **Post saves first.** `/bills/post` takes only the four keys, so it posts
   what the server holds, not what the screen shows. Every post path is
   therefore `/create` → `/validate` → `/post`, and auto-post validates
   **before** `/create`, so a refused bill leaves no draft and burns no number
   (§17).
2. **A 422 from validate is the answer, not an error.** It is painted in the
   warning strip. It never also opens the generic error box (§16).
3. **Locks and rights are read, never computed.** `rights` and `locks` come from
   `/bills/get`. The client never derives "can I amend" from the status.
   `locks.editable.document` is false on **every** posted bill, and Amend is the
   verb that reopens it (§17.2, §17.8).
4. **An adjustment is never a tender.** A set-off of an advance or credit note
   goes in `adjustments[]` plus `sbAdvanceAmt` / `sbNoteAdjAmt`. The ADJUST row
   in the tender dialog is a read-only mirror of that total. It is never sent
   as a `tenders[]` row, never carries a surcharge and is never counted as
   tendered (§14).
5. **`adjustments` absent ≠ `adjustments: []`.** Omitting the key means
   "leave them". An empty array means "reverse every set-off". The client
   omits the key whenever it could not read them (§14.4). It must also send
   them on `/post`, or the server picks the party's oldest credits instead
   (§17.5).
6. **Money is decimal, rounded once, split from one rounded figure.** CGST and
   SGST come from one rounded local-GST amount, in paise. Summing unrounded
   halves once gave 3999.99 on a 4,000 inclusive line, and the server refused
   it (`SALES_AMOUNT_MISMATCH`) (§10.2).
7. **The bill ports to the web as an online screen.** Posting is online this
   phase (FLOW §10). The payload is offline-ready (client-minted uuidv7, every
   snapshot column filled), but no offline post path is built.

It is **not** the POS screen: `sbBillMode` is `WHOLESALE` from menu 12, and POS
will be its own screen (§3.4). It is not the sale return, and it is not the
place bill numbers, COGS, ledgers or GST registers are decided. The server
owns all four.

---

## 2. Module layout

```
src/domain/pricing/                 ← OWNED BY plan-react.md (quotation). Imported, never forked.
  lines.ts  charges.ts  document.ts  totals.ts  policy.ts

src/sales-shared/                   ← one package, shared by quotation / order / bill / DC
  engines/                          // pure TS: no React, no fetch
    promotions/  rules.ts  match.ts  evaluate.ts        (§11)
    adjust/      adjust.ts          // clamp, fifo, release, validate (§14)
    tender/      masterRow.ts       // TenderMasterRow + type table, SHARED WITH RECEIPT (§15.3)
                 settle.ts          // row gross/base, totals, change routing, F1 (§15.4)
    weightBarcode.ts  size.ts  priceLevel.ts  netQty.ts
  components/
    ItemGrid/         // ui_table 22 layout, cell editors, picker popup (§8)
    ChargeGrid/       // ui_table 21, picker grid 82 (§12)
    AdjustPanel/      // ui_table 25 (§14)
    TenderDialog/     // purpose: 'settlement' | 'advance' (§15)
    ShippingDialog/   // shared with DC / returns (§20)
    WarningStrip/     // + ValidationPopup + confirmProceed (§16)
    OpenSourcesDialog/  CustomerQuickAdd/  AskText/  (AskText = prompt with presets + required)
  api/
    masterLookups.ts  // item-price, item-switch-uom, item-by-barcode, customer-detail, freight
    charges.ts  promotions.ts  tenderMasters.ts  partyBalance.ts  customers.ts  txnHolds.ts

src/features/saleBill/
  SaleBillScreen.tsx               // layout + wiring; no arithmetic
  components/
    IdentityStrip.tsx              // title, source list, status pill, audit, GST badges (§21)
    QuickStrip.tsx                 // icon buttons (§6.2)
    PartyBand.tsx                  // customer + bill-to + options + facts line (§7)
    DeliveryBlock.tsx  BillInfoBlock.tsx  CreditPanel.tsx
    TotalsBlock.tsx  StatsLine.tsx  VerbBar.tsx
    HeldBillsPicker.tsx
  state/
    draft.ts                       // BillDraft + reducer: the ONLY mutable state (§5)
    useBillDraft.ts                // thunks: load, save, validate, post, amend, cancel, …
    verbs.ts                       // verbState(draft) → the verb bar, pure (§17.1)
  domain/                          // no React, no fetch
    rollups.ts                     // settlement header figures (§15.9)
    creditGate.ts                  // the save-time credit confirm (§17.3)
    validate.ts                    // instant client checks, ordered (§17.3)
    loadMap.ts                     // /bills/get → draft (§19)
  payload/
    buildSavePayload.ts            // §18 — explicit whitelist per DTO
    buildTenders.ts  buildAdjustments.ts  buildItems.ts
  api/
    bills.ts                       // the 14 verbs, BillKey typed (§4)
    gst.ts                         // einvoice / ewaybill / vehicle
  retender/ RetenderDialog.tsx  retenderPayload.ts                        (§22)
  lists/    BillList.tsx  DeliveryList.tsx  TempCreditList.tsx  FollowUpDialog.tsx  (§24)
```

`src/domain/pricing`, `sales-shared/engines/*`, `features/saleBill/domain/*`
and `payload/*` never import React or `fetch`. Every number the tests check
comes from there (§29).

---

## 3. Scope, keys and identity

### 3.1 The four keys, two spellings

A bill is `(sbId, sbCompanyId, sbBranchId, sbAccYear)`. The table is
partitioned by `acc_year`, so the year is part of the identity, not a filter.
Model it as **one typed `BillKey`**, so a caller cannot send three keys and
find out about the fourth at runtime:

| Route family | Spelling | Where |
|---|---|---|
| document-addressed: `/get`, `/tender-context` | `sbId, sbCompanyId, sbBranchId, sbAccYear` | query |
| document-addressed: `/post`, `/cancel`, `/amend`, `/delete`, `/transport`, `/retender`, `/delivery-status` | same sb-prefixed names | **body** |
| context reads: `/party-context`, `/open-sources` | **bare** `partyId, companyId, branchId, accYear` (+ `billDate`, `kind`) | query |

- The four GET routes parse ids with **`ParseUUIDPipe({version:'7'})`**, so a
  uuid4 is a 400. Body ids accept v1–v8. Any id the client mints is uuidv7.
- The GETs take individual `@Query()` params, not a DTO. **An unknown or
  missing key is silently ignored, not refused.** A missing `sbAccYear` gives a
  lookup that is not filtered by year. `api_endpoints.h` says the opposite; it
  is wrong. **Always send all four.**
- `partyId` is the customer id, which **is** the ledger id (house rule 1). No
  bridge.

### 3.2 Where each key comes from

- **New bill**: company, branch and year come from the session and are
  captured once on mount. A loaded bill **overwrites them from its own
  record**. An edit always acts in the document's scope, never the session's.
- **Drill-in from a list**: the key comes from the row
  (`sb_company_id / sb_branch_id / sb_acc_year`). If the row lacks one, refuse
  with *"The list query must return sb_company_id, sb_branch_id and
  sb_acc_year on every row."*

### 3.3 Device and session

`sbDeviceId` is required on `/create`. At post, the stock voucher needs a
**registered** device (`fixed.device_master`), or the server refuses with
`SALES_DEVICE_UNREGISTERED`. The Qt client falls back to a hardware
fingerprint, which is not registered. That is why the live suite hard-codes a
registered device. **The browser must not invent a device id.** It sends the
one login returns. If login returns none, the screen says posting is not
possible on this device, and it does not guess. This is an open server item
(§27, DEV). `sbSessionId` = the login session uuid. `sbCounterId` is not sent.

### 3.4 Bill mode

`sbBillMode` comes from the launching menu (menu 12 = `WHOLESALE`) and is never
a combo. A loaded bill keeps its own. POS changes the number series, skips the
e-way warning and sets delivery status to `NA`, all server-side.

---

## 4. The wire contract

### 4.1 Routes (all `/api/v1`, bearer token)

| Verb | Method + path | Body / query | Answers |
|---|---|---|---|
| save draft | POST `/bills/create` | `SaveBillDto` (§18). `sbId` present = update of a **DRAFT** | the full `/get` shape (no second GET needed) |
| validate | POST `/bills/validate` | SaveBillDto + `overrides[]` | `{ok, refusals[], warnings[], rights, proposals}`, or 422 with the refusals |
| post | POST `/bills/post` | keys + `overrides[]?` + `adjustments[]?` + `printAfter?` | `/get` shape; idempotent on POSTED |
| amend | POST `/bills/amend` | SaveBillDto + `sbId` + `baseRevision` + `editRemark` + `overrides[]?` + `adjustments[]?` | `/get` shape |
| cancel | POST `/bills/cancel` | keys + `reason` | `{…keys, sbStatus:'CANCELLED', reversalVoucherRefno, cancelledOn}` |
| delete | POST `/bills/delete` | keys (DRAFT only) | `{sbId, deleted:true}` |
| read | GET `/bills/get` | sb keys | `BillPayload` (§19) |
| party | GET `/bills/party-context` | `partyId, companyId, branchId, accYear, billDate` | §7.3 |
| sources | GET `/bills/open-sources` | `companyId, branchId, partyId, kind=DC\|ORDER, accYear?` | §13.4 |
| transport | PUT `/bills/transport` | keys + `transport{…}` nested | the band row |
| tender read | GET `/bills/tender-context` | sb keys | §22 |
| re-tender | POST `/bills/retender` | keys + `voids[]` + `tenders[]` + `remark` | `/get` shape |
| delivery | PUT `/bills/delivery-status` | keys + `event` + `remarks?` | `{sbDeliveryStatus}` |
| remarks | PUT `/bills/update-remarks` | — | **not used by this screen** (removed 2026-09-25; remarks change through Amend) |

Other routes the screen calls: GET `/master-lookups/{item-price, item-switch-uom,
item-by-barcode, customer-detail, freight-charges/charge}`, GET `/charges/get`,
GET `/promotion-scheme/list`, GET `/tender-masters/list`, GET
`/transactions/party-balance`, GET `/quotations/get`, GET `/sale-orders/get`,
PUT `/sale-orders/cancel-lines`, POST `/customers/create`, GET
`/branch-masters/get`, `/txn-holds/*`, `/temp-credits/follow-up`, and
`/gst/{einvoice/generate, ewaybill/generate, ewaybill/vehicle}`.

### 4.2 Envelopes

- Success: `{success:true, message, data}`.
- Error: `{success:false, message, errors:[{field, message, code?, line?, statutory?}]}`.
  **There is no `statusCode` in the body.** Switch on the HTTP status plus
  `errors[].code`. A class-validator 400 has no code, and its `field` is inferred
  from the message (`items.0.sbiRate …`). A 500 answers `errors: []`.
- Status map: **400** request malformed · **403** a `user_menus` right is false
  (`SALES_RIGHT_*`) · **404** not found · **409** state refuses (posted,
  cancelled, stale revision, lock 2, day closed, PDC moved) · **422** rule
  refusals, **all of them in one body**.

### 4.3 Types on the wire: what arrives vs what to send

| | Arrives on `/get` | Send |
|---|---|---|
| header + item decimals | **strings** (`"1234.5"`, not padded; tiny values can be exponent form). Swagger says numbers; it is wrong | number or numeric string (both pass); the client sends numbers rounded at the column's scale |
| charges, tenders, posting, locks | numbers (`tdTenderTypeId` is a **string** on `/get`, a number on `/tender-context`) | numbers |
| date-only header/item columns | full ISO datetime `"2026-09-25T00:00:00.000Z"`, so slice(0,10) | `YYYY-MM-DD` |
| `sbBillDatetime` | ISO | full ISO |
| blank text | `null` | `null` (a blank string is trimmed to null; you cannot store `''`) |

`wire/parse.ts` reads every number through one lenient `toDecimal(v)` that
accepts number, numeric string and exponent form. The Qt client's `NexJson`
does the same, which is why it never broke.

### 4.4 The echo trap: build the body from state, never from the response

`forbidNonWhitelisted` is on for every BODY DTO, at every nesting level. A
`/get` response echoed into `/create` is a **400**. It carries `posting`,
`locks`, `rights`, `sources`, `tempCredits`, `transport`, about 16 header and
item fields that are not on the DTOs (`sbDcTaxableAmt`, `sbDeliveredOn`,
`sbiTaxId`, `sbiLoyaltyPoints`, `sbiItemName`, …), tender extras
(`tdTenderName`, `tdTenderLedgerName`) and `cdLedgerName`. `payload/*` builds
each body **from an explicit per-DTO whitelist**. The shared charge builder
has a second trap: the quotation DTO 400s on the carry keys, so they are
emitted only on carried rows (§12.4).

---

## 5. State

### 5.1 One draft, one reducer

```ts
type BillDraft = {
  key: BillKey | null;                    // null until the first /create answers
  status: 'NEW' | 'DRAFT' | 'POSTED' | 'CANCELLED';
  refno: string | null;  revisionNo: number;  baseRevision: number | null;
  mode: 'entry' | 'browse';               // browse = read-only, from locks.editable.document
  amending: boolean;
  draftFromAutoPost: boolean;             // a failed auto-post left this draft (§17.4)
  policy: VoucherPolicy;                  // freightCalcType, loadingCalcType, discAlterBase, roundOffStep
  header: BillHeader;                     // §7 fields, crew arrays, terms, flags, source trail
  party: PartyFacts | null;               // /party-context, keyed by the customer it was asked for
  customer: CustomerDetail | null;        // /master-lookups/customer-detail
  lines: BillLine[];                      // stable lineUid per row, never an index
  charges: ChargeRow[];
  adjustments: { rows: Adjustment[]; authoritative: boolean; heldByDocument: Adjustment[] };
  tender: { lines: TenderLine[]; rollups: Rollups };
  transport: TransportBand;
  posting: PostingBlock | null;  locks: Locks | null;  rights: Rights | null;
  notes: ValidationNote[];  overrides: Set<string>;
  sources: SourceSummary[];
};
```

One `useReducer`, with the thunks in `useBillDraft`. This is the same shape as
`plan-react.md` §8.

### 5.2 Priced vs stored: nothing reprices on load

A loaded bill is **history**. Its lines, taxes and totals are painted exactly
as stored. No price lookup and no promotion pass runs against today's
campaigns. Use the quotation plan's `stored | priced` line union. A line
becomes `priced` only when the operator touches it (qty, rate, discount,
unit, level). Totals are the stored header figures until the first touch, and
after that the document pass derives them (§10). This one rule replaces the
Qt `m_loading` guard in about fifteen places.

### 5.3 User intent vs programmatic writes

The Qt screen is careful to recompute only on **user-intent** signals
(`cellEdited`, `activated`, `clicked`, `dateEdited`, `textEdited`), never on
value-changed signals that its own writes also fire. The React form of the
same bug is a `useEffect` watching state that the recompute itself writes.
**Derived values are computed in the reducer or a selector, never in an
effect.** A recompute is a direct call from a user-event handler. A
programmatic fill (load, import, seed, customer lookup) sets state and runs
**one** recompute at the end, or none if it is a load.

### 5.4 Stale responses

Every async read that can land after the operator has moved on is guarded by
**the thing it was asked for**, not just by unmount:

- party-context and customer-detail: drop the reply unless
  `draft.header.customerId` is still the id it was asked for;
- party-balance (adjust panel): per-fetch token, keyed by the party;
- item-price: apply to the row by `lineUid`. The Qt handler captures a **row
  index** across the await, which is a latent bug (a delete in between prices
  the wrong line). Do not port it;
- order-line cancel: re-find the row by source line id after the PUT.

### 5.5 Autosave trigger

Any draft change on a NEW bill with at least one line schedules a debounced
(1 s) autosave (§23.2). It is skipped while a load or restore is painting.

---

## 6. Screen layout and keyboard

### 6.1 Bands, top to bottom (the three-column form the user dictated, not cards)

```
IdentityStrip   SALES BILL ENTRY | sources list | status pill | audit | GST badges (posted only)
QuickStrip      21 icon buttons (§6.2)
PartyBand       col 1 Customer · col 2 Delivery · col 3 Bill info · col 4 Credit (widths 28/24/24/24)
ItemGrid        the only stretching band; min height 180
Footer          ChargeGrid | Totals block (Taxable · Tax · Charges · BILL AMOUNT /
                Tendered · Adjusted[…F4] · Change · Credit / Remarks / stock hint)
BottomBar       Total Profit (x% margin) | Total Saving (y%) | WarningStrip one-liner | verbs (§17.1)
```

- **Col 1, Customer:** Beat (hidden row, still wired), Customer (+ quick-add
  icon), Temp Name (walk-in only), Address, Place + PIN, Phone, option ticks
  (Freight · Load · Unload · Promo · Loyalty · Quick Del), facts line (§7.5).
- **Col 2, Delivery:** Salesman, Loadman, Driver, Vehicle No, Packed By
  (hidden), Supervisor, Agent (hidden).
- **Col 3, Bill info:** Bill No (read-only), Date, Term, Their Ref (hidden),
  Ref Date (hidden), Price Level, Due Days (Credit term only).
- **Col 4, Credit** (all read-only): Outstanding `pending | overdue` ·
  Limit / Avail · Adv / CN · Cash today / Temp · Loyalty `points | card`.
- Hidden rows **still travel on the payload**. In React they are state fields
  with no rendered row.
- GSTIN, POS state and Category **left the form**. They are state, still
  written, and the GSTIN number is shown on the facts line. Consequence: a
  walk-in cannot type a GSTIN, so a B2B invoice to a walk-in is impossible.
  This was flagged by the developer, not decided by the user (§28 Q6).
- The Totals block's Round Off, Items, Bags, Weight, Gross and discount fields
  are hidden but still computed and sent.
- The Qt `.ui` has a `tblFooterExtra` grid and an empty root row 4. Neither
  is used. Do not port them.

### 6.2 Quick strip

Icon-only buttons with the caption as tooltip. Live: Quick add customer ·
Quote import (Ctrl+F3) · Order import (Ctrl+F4) · Challan (Ctrl+F5) ·
Disc % All (Alt+D) · ± Price · Cost (Alt+O) · Shipping · e-Inv · e-Way ·
Copy (Alt+Y). **Disabled "coming soon"** (render disabled, do not hide, same
as Qt): Cust. history, Check price, Last rates, Split stock, Stock fix,
Ledger, Receipt, Returns, Pick list, WhatsApp.

- **Disc % All:** prompt 0–100 (2 dp). For every non-free line: clear
  per-qty and amount, then set Disc %. Then recompute.
- **± Price:** prompt −100…100 (0 = no-op). `rate *= 1 + p/100` on non-free
  lines. MRP and min checks apply at save, not here.
- **Cost:** *"<desc>\nCost: ₹x · Cost (pre-tax): ₹y"* for the current line.

### 6.3 Keyboard, whole screen

| Key | Action |
|---|---|
| F2 | Edit. A posted bill → Amend (§17.8) |
| F3 | Delete (DRAFT only) |
| F4 | Adjust advances / credit notes (§14.3). This is its own key because a credit-term bill never opens the tender |
| F5 | Tender route → open the tender dialog. Plain route → Save |
| F6 | amending → save changes & print · tender route → tender (print) · auto_post → save (= post) & print · else post & print |
| F7 | Clear = "start a new bill", in **every** state (§17.11) |
| F8 | Bill list (as picker) |
| F9 / F10 | Hold / Pick held (keying a new draft only) |
| F11 | Last bill print (§26) |
| Esc | Close. The bottom-bar button reads **"Close - Esc"**; two buttons called Cancel on a money screen is how the wrong one gets pressed |
| Ctrl+F3 / F4 / F5 | Import quotation / sales order / challan lines |
| Ctrl+F6 | Re-tender (§22) |
| Ctrl+F7 | Validate (dry run) |
| Ctrl+Enter | Post & print · **Ctrl+Shift+Enter** post without printing. There is no button for the second |
| Shift+F2 / Shift+F3 | Open menu 152 (e-invoice register) / 153 (e-way register) |
| Alt+D / Alt+O / Alt+Y | Disc % all / line cost / copy as new |
| Ctrl+1…Ctrl+N (N ≤ 4, grid-scoped) | Document price level N (§8.7) |
| grid: bare `+` / `−` | Add line above / remove line |
| grid: ± level hooks | Step this line's price level (§8.7) |
| grid: unit key | Switch unit (§8.5) |

One keymap reads the verb state (§17.1). Do not port the Qt habit of routing
F5 by checking whether a button is visible, and do not port re-setting
shortcuts after `setText()`.

### 6.4 Dropdowns the screen uses, and the gaps to close first

| Field | Dropdown | Params (ALWAYS sent; `''` = unset) |
|---|---|---|
| Customer | **58** (customers by beat) | `icus_area_id` |
| Beat | 13 (area) | — |
| POS state | 21 | — |
| Salesman / Loadman / Driver / Packed / Supervisor | 38 | `iemp_branch_id` = session branch |
| Agent | 45 | `isa_branch_id` |
| Card/cheque bank | 46 | — |
| Transporter / Vehicle (shipping) | 55 / 56 | `itrn_company_id` / `iveh_company_id` |
| Godown (shipping) | 26 | — |
| Quick-add Area / Group / State | 13 / 33 / 21 | — |
| Price level | `priceLevels` combo | `SELECT ipl_id, ipl_name … WHERE ipl_status ORDER BY ipl_id` |

Gaps in `plan-react-dropdown.md` that the bill needs closed:

1. **Every optional param is always sent.** An unsent token is bound as its
   own literal name by the runner (the grid-runner rule applies to dropdowns
   too).
2. **Select by id and resolve the label.** Today `setSelection(id,'')` shows
   a blank. The bill needs "select id, fetch its text" for customer seeding
   and loaded salesman / godown / transporter / vehicle. `/bills/get` carries
   no staff display names (§27).
3. A dropdown exposes a **user pick** event separately from a value set, so
   a programmatic fill never fires the pick handler (§5.3).

---

## 7. The party band

### 7.1 Picking a customer

Operator pick on Customer (dropdown 58):

1. **Confirm the change if set-offs exist:** *"N adjustment(s) totalling X
   are set against <name>.\n\nChanging the customer will remove them.
   Continue?"* No → the customer is put back silently. Yes → clear
   adjustments, then continue.
2. Mark `creditAlertPending` (only an **operator** pick earns the credit
   popup, §7.4).
3. Fire **both** reads in parallel: `customer-detail` and `party-context`.
   Each writes its **own** fields. One writer per field: *"two sources writing
   the same fields is how they end up disagreeing."*
4. Re-run promotions and recompute. The customer affects party scope and
   `allow_promotion`, so lines already keyed are judged again.

Beat is a **filter** on the customer list (`icus_area_id`), never painted from
the customer. Its row is hidden but still wired.

### 7.2 `customer-detail` (keys snake_case: `cus_id, company_id, branch_id, regional`)

It still owns the address, the state and the freight facts, because
party-context carries no address block.

| Response | Goes to |
|---|---|
| `cust_name, cust_ename, cust_phone1, cust_eadd1..3, cust_place, cust_pin, gst_no, ecommerce_gstin, gst_type` | bill-to snapshot |
| `state_code, state_name` | `custStcd` (the customer's registered state) and, **when present**, the POS default |
| `local_sales` | fallback only, when the customer has no state |
| `area_id, distance_km` | freight bands refetch when either changed **and** `freightCalcType != manual` |
| `allow_discount, debit_allowed, debit_days, debit_limit, overdue_billing, available_balance, available_bills` | credit fields (fallback for the save-time gate) |
| `freight_charge, cooly, unloading_charge, allow_promotion, allow_loyalty` | option ticks Freight / Load / Unload / Promo / Loyalty (set, **no confirm** — the confirm is for operator clicks only) |
| `price_level` | a **level id** → document price level |

**Term:** if `!sales.allow_payment_term_change` **or** the bill has no source
document, force the term to the party's: Credit iff `debit_allowed`, else
Cash. An imported bill keeps its source's term only while term change is
allowed. Then re-derive due days (§7.6).

### 7.3 `party-context` (BARE keys, `billDate` drives credit ageing)

```
party   { ledId, name, gstType, gstin, stateCode, isWalkIn, panNo, panVerifiedOn, form60On,
          creditAllowed, defaultPriceLevel, addr, place, pin, phone, areaId, … }
credit  { limitAmount, limitBills, creditDays, used, openBills, oldestOpenDays,
          amtExceeded, billExceeded, daysExceeded, mode: OFF|WARN|REFUSE }
cashToday
advances[] / creditNotes[]  { ablId, ablAccYear, refno, pending, date }
loyalty  null | { memberId, cardNo, balance, redeemable, rate, minPoints, maxPoints,
                  maxRedeemAmount, multiple, schemeId, allowPointRedeem (default true) }
shipTo[] { saaId, name, addr, place, pin, stcd, gstin, phone, distanceKm, isDefault }
tempCredits[] { atcId, billRefno, name, mobile, balance, dueDate }
openSources { dc, orders }
```

- **A failure never blocks billing.** It clears the panel, and `/validate`
  stays the authority.
- **`party.creditAllowed` overrides the customer-detail flag.** A loaded bill
  never runs customer-detail. Without this, every amend of a credit bill was
  refused with "not allowed to buy on credit" (live test, 2026-09-24).
- `loyalty: null` means **not a member**. That is an answer, not missing data.
- `party.isWalkIn` replaces the session's guess once it lands.
- 404 on an unknown `partyId`.

### 7.4 Credit panel and the one-time alert

- `limit 0 = no limit`. `mode OFF` = check disabled. Available =
  `limitAmount − used` (when a limit is set, else "—"). Available bills =
  `limitBills − openBills`.
- Rows: Pending `amount (count)` · Overdue `n credit day(s)` / "no credit
  days" · Limit `amount (bills)` / "n bill(s)" / "no limit" · Avail `avail …`
  / "—".
- **Red only while the check is on:** Avail on amount/bill exceeded (tooltip
  "Over the credit limit" / "Over the open-bill limit"); Pending on
  `daysExceeded` (tooltip "Overdue — oldest open bill N day(s), terms M
  day(s)").
- **No overdue amount** (the endpoint ages only the oldest bill) and **no
  client verdict** (`mode` only repeats what `/validate` will do).
- **One popup per operator pick,** only when `mode === 'REFUSE'` and
  `creditAllowed` and one of the exceeded flags is set: *"<name> is <reasons>
  — a credit bill will be refused. Bill it as cash, or collect first."*
  Reasons: "over the credit limit" / "over the open-bill limit" / "overdue
  (oldest bill N day(s))". Loads and seeds stay silent.

### 7.5 Facts line

`GST <type|—> · <gstin> · <stcd> · intra|inter` · `PAN / FORM 60`: PAN (+✓ if
verified) in green, else "Form 60 · dd-MM-yyyy" in green, else "none" in amber.
It is always shown, because of 269ST and TCS. Four cells, grey when empty:
Advances `amt (n)` green · Credit notes (tooltip "Credit notes open — F4 sets
them off") green · Cash today (tooltip "Cash taken from this party today
(269ST)") amber · Temp credit amber. **Clear resets them.** The Qt screen once
left the last customer's advances showing after Clear.

### 7.6 Term, due, date

- Term `Cash | Credit`. Locked unless `sales.allow_payment_term_change`
  (BRANCH scope; its setting row, SQL 38, is not yet applied on the box, so the
  default "locked" holds). Tooltip when locked: *"The customer's own term —
  changing it is off for this branch (Allow payment term change)."*
- Term change (operator) → re-derive due, **then** re-run promotions and
  recompute. The promotion pass runs on the operator's edit, **not** inside
  "term changed". The same function is reached from the tender dialog (a
  credit line flips the term just before saving) and from order import.
  Re-pricing a bill whose split is already settled would leave tender lines
  adding up to a total that no longer exists.
- **Due days**: editable 0–3650, defaults to the party's credit days, shown
  only on Credit; `dueDate = billDate + dueDays`. A bill-date move keeps the
  typed days.
- **Bill date**: defaults to the session's app date. Operator edit →
  re-derive due, re-run promotions, recompute. It must fall inside the
  session's accounting year (§17.3).

### 7.7 Bill-to lock, walk-in, customer lock, quick add

- The **walk-in** is `sales.default_customer_id` (seeded on a new bill when
  `sales.pop_default_customer`). **Only the walk-in's bill-to text is
  typeable**, and the Temp Name row appears only for it. Every other
  customer's snapshot is filled and disabled. The old "Edit for this bill"
  toggle was removed (2026-09-24).
- A cleared `default_customer_id` override arrives as `value:null,
  source:OVERRIDE`. Apply `''`. Never a compiled fallback uuid.
- **Import lock:** a bill with a source document and
  `!sales.allow_customer_change_on_import` locks Customer and Beat. Tooltip:
  *"Imported from X — the customer belongs to that document and can't be
  changed here."*
- One derived `editable` per bill-to field:
  `walkIn && !importLocked && mode==='entry'`. The Qt screen has two locks
  fighting over one property. Do not port that.
- The server **overwrites** the bill-to snapshot from the master on `/create`
  unless `custOverride` is true, for any listed (non-walk-in) customer. It
  never overwrites `sbPosStcd`. The client does not send `custOverride`.
- **Quick add** (`/customers/create`): Name* · Phone · GSTIN · GST type
  (Unregistered | Regular | Composition; SEZ/Overseas 400 on the ledger) ·
  Area* (dropdown 13; it is also the ledger group) · Group* (33) · State*
  (21, default company). A GSTIN must be 15 chars, valid and start with the
  state code. Unregistered must have none. Body: `cusName, cusPhone1,
  cusGstNo, cusGstType, cusAreaId, cusGroupId, cusStateCode, cusStateName,
  cusPriceLevelId = sales default level, cusCompanyId, cusBranchId,
  cusIsActive:true`. A 409 names the clashing ledger. On success the new
  customer is handled **exactly like an operator pick** (§7.1, including the
  adjustments confirm).

### 7.8 Place of supply

- Two separate facts: **`sbCustStcd`** = the customer's registered state;
  **`sbPosStcd` / `sbStateName`** = place of supply.
- POS defaults to the **company's** state (a fresh bill is a local sale),
  moves to the customer's state on a pick, and the operator may override it
  (the dropdown is hidden today, so in practice the customer's state wins).
- `isLocal = posStcd === companyStateCode`. It is the one authority for the
  CGST/SGST vs IGST split and for the charges' inter-state flag. After any
  POS change: set the charge flag, then one recompute.
- A loaded bill derives local/inter **from its stored taxes**:
  `(cgst+sgst) >= igst` when any tax is present, else from `sbPosStcd`.
- Reset re-applies the charge inter-state flag. The Qt charge controller's
  `clear()` keeps the old flag; do not reproduce that.

### 7.9 Crew

`sbSalesmanId`, `sbLoadmanId` and `sbPackedId` are **`uuid[]`** (element 1 =
primary, credited in outstanding-by-salesman). The screen keeps one picker and
sends a one-element array, or `[]`. It reads element 1 back, and also accepts
a bare string from older rows. `sbAgentId`, `sbDriverId`, `sbSupervisorId` and
`sbVehicleId` are single. `sbAgentCommPerc/Amt` have no input; they are carried
so a re-save keeps them. **Clear keeps the crew** when
`sales.clear_delivery_on_clear` is off (a trip bills many customers with the
same van). The agent is always cleared.

`sales.salesman_mandatory` → client check "Select a salesman.", and the server
refuses too (`SALES_SALESMAN_INVALID`).

---

## 8. The item grid

### 8.1 Layout and columns

ui_table **22**, used by the bill alone. It has 99 columns: the full
`SalesItemCols` set, with 94–98 hidden (SrcDocType, SrcLineId, SrcDocYear,
LotId, Bucket). **14 are visible:** Barcode, Description, BatchNo,
GodownName, Uom, Bill Qty, Sch, Rate, Disc %, Taxable, GST %, GST Amt, Total,
Src Type. `/ui-table-masters/get` returns `uiTblClmNo` as a **string**.
Virtualise the rows (TanStack Virtual, as the quotation grid does).

Row model: typed fields, **booleans for the Y/N flags** (the Qt grid stored
text, and `toBool("N")` is true — IsInclusiveTax was once always true), full
precision numbers, and a `lineUid`. Formatting happens only at render.
`RateBeforeTax` and `NetPriceBeforeTax` are divisors: never round them to 2 dp.

The Qt "trailing blank row is the add-line affordance" convention is **not**
ported. The React grid keeps a data array plus a virtual new-row, so no loop
runs to `rowCount − 1`.

### 8.2 What is editable

| Column | Editable when |
|---|---|
| Barcode | only while empty (the scan lands here; it is the first visible column) |
| Description | opens the item picker (Down arrow = picker) until an item is set, then read-only |
| Size | when size entry is on (the bill turns it on; timber shops) |
| CaseQty / BillQty / LengthQty | item set; decimals = the item's `decimal_count` |
| Rate | item set **and** `session.editPrice` |
| Disc % / per-qty / amount, Spl disc × 3 | item set |
| SchemeFlag | Y/N select |
| Remarks | text |
| everything else | never inline |
| **a free line** (`isFree`) | **nothing** |
| an imported line | ItemId and Description read-only; **OrderQty read-only** ("a cap the operator can raise by hand is no cap at all") |

**Discount tiers are one-of-three:** typing Disc % clears per-qty and amount,
and so on for each tier (Disc, Spl). A keyed amount survives only while
neither % nor per-qty is set.

### 8.3 Picking an item

- Picker = grid **71** popup on Description, with params `iitem_company_id` =
  the document's company and `iitem_branch_id` = the document's branch, **sent
  even though grid 71 does not read them yet**. Grid 71 offers every
  company's items until it does, and a param missing from grid_param fails the
  query.
- The picked row gives `item_id`, `item_uom_id`.
- **Duplicate:** if another non-free line has the item —
  `!sales.allow_duplicate_item` → bump that line's qty by 1, focus it, and
  leave the picker row empty; allowed → confirm *"This item is already on row
  N. Add it again?"*.
- Then the price lookup (§9.1). On a new row the grid grows.

### 8.4 Adding and removing lines

- **Add** (`+`): inserts a line **above** the current one.
- **Remove** (`−`), not on a free line in browse mode:
  - A line whose source is a **sales order** (`srcDocType === 'SALES_ORDER'`;
    a quotation line carries a source line id too, which is why the type is
    checked) gets a three-way choice: *"\"X\" came from a sales order.\n\nCancel
    on Order — the line is cancelled on the order too, and what is still
    pending on it is released.\n\nRemove from Bill — it only comes off this bill
    and stays pending on the order."* **Cancel on Order** → AskText (reason
    required, *"A reason is required to cancel an order line."*) → PUT
    `/sale-orders/cancel-lines?srcModule=<srcDocType>&srcDocId=<soiId>&srcAccYear=<ORDER's
    year>` body `{soiCancelReason}` → on success drop the line (re-found by id).
    On error **the line stays**, because the bill must keep saying what the
    order says.
  - Otherwise confirm *"Remove \"X\" from this bill?"*.
- A drop recomputes once.

### 8.5 Unit switch

GET `/master-lookups/item-switch-uom?item_id&iuc_id` cycles Box → Pc → … →
Box. The answer's `item_id, iuc_id` go to a fresh price lookup.

### 8.6 Quantity, rate and size edits

- **Size** (timber): typed `L*W*T*Pcs` (length ft, width and thickness in,
  pieces an integer, all four > 0). Parse on commit; the validator cannot stop
  a half-typed `12*6`. On a bad size: clear the cell and warn *"Enter the size
  as Length*Width*Thickness*Pieces — all four parts, none of them zero.\n\nExample:
  12*6*2*10 = 12 ft x 6 in x 2 in, 10 pieces = 10 cft."*. On a good one: write
  the canonical text back, then `BillQty = L*W*T*P/144` to **4 decimals**, not
  the item's decimal count (*"rounding a 10.4375 cft board to 10 would quietly
  give the customer three quarters of a foot"*), then run the qty rules.
  A loaded bill's size is painted and **never recomputed**, because the CFT
  recompute would overwrite a hand-adjusted qty.
- **Qty:**
  - Negative stock: `!allowNegative && !service && qty > stock` → warn
    *"Only X in stock — this item does not allow negative stock."*
    **The value is not reverted.** The operator may knowingly accept it, and
    the save gate asks again.
  - Order cap: `orderQty > 0 && qty > orderQty && !sales.allow_bill_over_order_qty`
    → warn *"Only X is pending on this order line — billing more than was
    ordered is not allowed."* and **put the qty back to orderQty**. Lines
    without an order qty are not capped.
  - Then promotions and recompute. If the next line is a free line, focus
    skips it.
- **Rate:** `minPrice > 0 && !free && rate < minPrice` → warn *"Selling rate <
  minimum selling price."* and clear the rate. Then promotions and recompute;
  `rateDiff = actualPrice > 0 ? rate − actualPrice : 0`.
- **Rate variance hint:** a small `±(rate − actualPrice)` annotation in the
  Rate cell, red below the list price, green above. Hidden on free lines,
  when |Δ| < 0.005, or when the cell is too narrow.

### 8.7 Price levels

- **The combo value is the level id, never its index.** Inactive levels leave
  gaps, and an old bug priced at index+1. The cell, the lookup, the payload
  and the defaults all use the id. A level id not in the list (inactive,
  other company) leaves the combo showing what it shows. It never snaps to
  the first level.
- The default is the session's `salesDefaultPriceLevel`. A document loaded
  while the level list was still loading applies **its own** level when the
  list lands, not the company default.
- **Document level** (combo or Ctrl+N): needs `session.editPrice`. Dialog
  *"Change price level to \"X\"?"* with **Apply to Selected** / **Apply to
  All** / Cancel. The affected rows' level is stamped **before** their lookups
  fire; that is what lets "Apply to All" override lines an earlier ± step put
  on another level. Only Apply to All moves the document level.
- **Line level** (± hooks on the grid): one line, one keystroke, no question.
  It steps in list order, clamps at the ends, and reprices that line.
- The grid's PriceLevel cell paints the level **name** ("MRP Sale"), and the
  cell holds the id.

### 8.8 Footer totals

`"n item(s)"` · qty sum · weight (Σ weight × qty) · the order/case/net qty
columns when non-zero · money columns when non-zero (gross, discounts ×4, net
gross, taxable, GST + parts, cess, freight, loading, total).

A stock hint under the totals shows `"Stock: <stock> <uom>"` for the current
line.

---

## 9. The line lookup and scanning

### 9.1 `GET /master-lookups/item-price`, exact keys

`branch_id, company_id, item_id, price_level (int), regional ("true"|"false"),
acccyear` — **three c's; that is the server's spelling** — `godown_id` (empty
until a godown selector exists), `customer_id` (customer-specific discounts),
`loading_type`, `freight_type`, and `unit_id` only when set. The level is the
line's own level if set, else the document's.

| Response key | Line field |
|---|---|
| `item_id, item_name, item_uc_id` | itemId, description, unitId |
| `godown_id, godown_name` | godown |
| `barcode, item_code` | barcode, searchCode |
| **`hsn_code`** (fallback `item_com_code`) | hsn. `item_com_code` is the company code, never the HSN; it was the wrong key on all four sales screens |
| `allow_promo` | schemeFlag |
| `item_group_id, item_category_id, item_brand_id, item_section_id` | promotion keys |
| `unit_name, decimal_count, loyalty_pv` | display / rounding / loyalty |
| `stock, reorder_qty, batch_config` | stock, reorder, batchConfig |
| `service_item, add_freight, allow_negative_stock` | booleans |
| `iuc_uom_weight, loading_charge, freight_charge, base_factor` | weight, loadingPerQty, freightPerQty, toBaseFactor |
| `stock_id, batch_no, batch_date, expiry_date` | single-batch design (user decision) |
| `price_level` (echo; else the requested one) | priceLevel. Otherwise a stepped line falls back to the document's on the next reprice |
| `sales_price` | rate **and** actualPrice |
| `cost_price, cost_wot, max_price, min_price` | cost, costPreTax, mrp, minPrice |
| `disc_perc, disc_qty` | disc %, disc per qty |
| `gst_rate, cgst_perc, sgst_perc, igst_perc, cess_perc, cess_unit, item_incl_tax` | tax |

Then:

- `session.autoPopQty && qty === 0` → qty = 1 (scan → qty 1);
- free line → rate 0, disc 0. `actualPrice` keeps the real price for savings;
- apply any **pending scan value** (§9.2), then recompute.

### 9.2 Barcode and weight labels

- A scan into an empty line's Barcode cell is a **lookup**: GET
  `/master-lookups/item-by-barcode?barcode&company_id`. `company_id` is optional
  on the server but always sent; without it, one scan on the box resolved to
  another company's item. The answer gives `itemId, itemName, unitId` (an iuc
  id).
- **Weight/price labels** (`sales.weight_barcode` JSON: `{prefix, itemLen,
  valueLen, valueKind: WEIGHT|PRICE, divisor}`; numbers may be strings;
  divisor ≤ 0 → 1; enabled iff prefix, itemLen and valueLen are all set):
  - the scan must start with the prefix;
  - its length must be `prefix + itemLen + valueLen`, or one more (EAN-13
    check digit, **not verified**);
  - it must be all digits.

  The item code goes to the lookup. The value **waits** and lands only after
  the price lookup has filled the line: WEIGHT → qty (then the qty rules);
  PRICE → rate (then the rate rules). Applied earlier, the lookup would
  overwrite it.
- Not found: *"No item carries barcode X."*, clear the cell, drop the pending
  value, refocus.
- Found: fill the line, lock item + description, grow the grid, price it,
  and **move focus to the next line's Barcode cell** ("scanning is a rhythm").
- Increment-not-append (the SIX §2 spec for all six documents) and
  `/sales/resolve-item-scan` are **not built** server-side. The client does
  duplicate handling per §8.3 only.

### 9.3 Freight bands

When the freight calculation is not manual and the customer's distance is
known: GET `/master-lookups/freight-charges/charge?distance` gives bands
`{fromWeight, toWeight, freightCharge}`. For every line with `hasFreight`,
the first band with `from ≤ weight ≤ to` (both inclusive) sets
`freightPerQty` and `freightAmt = qty × freight`. Then recompute. Whether the
band is matched on per-unit or total weight is not settled in the Qt code
(§28 Q9).

---

## 10. Pricing: what the bill adds to `plan-react.md`

The engine is the quotation's. The bill runs the **same** two-pass pipeline:
line goods → charge distribution → tax → document totals. These are the
places where the bill's numbers must match the Qt screen exactly.

### 10.1 Line goods

- `netQty = caseQty × toBaseFactor + billQty × (lengthQty > 0 ? lengthQty : 1)`.
- A free line's rate is forced to 0 in **state**, not just in the local
  calculation. Any later lookup writes the real price back, and the grid, the
  totals and the payload must tell one story.
- Standard: `baseRate = incl ? (rate − cessPerUnit) / (1 + (gst% + cess%)/100) : rate`;
  `gross = netQty × baseRate`; `netGross = cascade(gross)`.
- Discount-first (`policy.discAlterBase`): `cascade(netQty × rate)` on the
  **tax-inclusive** value, then de-tax the discounted rate; `gross = netGross`
  (post-discount in this mode).
- Cascade, each tier on the running base: Disc (% > per-qty > kept amount) →
  Spl (same) → Scheme (same; the promotion writes here) → **Bill scheme =
  base × billSch% / 100, always recomputed**. A directly written bill-scheme
  amount is destroyed, which is why bill-level promotions write a %.
- `freightAmt = billQty × freightPerQty`; `loadingAmt = billQty × loadingPerQty`
  — **billQty, not netQty**.

### 10.2 Tax, in paise (the trap that cost a live refusal)

```
p(v)     = round(v × 100 ± 1e-9) / 100          // or decimal HALF_UP to 2 dp
taxable  = p(netGross + chrgBeforeTax)
cess     = p(taxable × cess% / 100 + netQty × cessPerUnit)
local:   localGst = p(taxable × (cgst% + sgst%) / 100)
         cgst = p(localGst × cgst% / (cgst% + sgst%));  sgst = p(localGst − cgst)
inter:   igst = p(taxable × igst% / 100)
gstAmt   = p(cgst + sgst + igst)
total    = taxable + gstAmt + cess + chrgAfterTax
netPrice = total / netQty;  netPricePreTax = taxable / netQty
savings% = mrp > 0 ? (mrp − netPrice) / mrp × 100 : 0
profit   = netPrice − cost;  profitPreTax = netPricePreTax − costPreTax
```

**CGST and SGST are split from one rounded figure.** Summing unrounded halves
once gave 3999.99 on an inclusive 4,000 line at 18%, and the server refused
it (`SALES_AMOUNT_MISMATCH`, user 2026-09-24). If `plan-react.md`'s
`lines.ts` rounds the halves separately, change it there; the quotation has
the same exposure.

### 10.3 Document pass (order is load-bearing)

1. Goods for every line.
2. Basis per line `{netGross, billQty, netQty, weight × billQty}`.
3. Role overrides (host-priced shares):
   - `FREIGHT` ← `freightAmt` when Freight is ticked, unless freight calc is manual;
   - `LOADING` ← `loadingAmt` when Load is ticked;
   - `UNLOADING` ← **`loadingAmt`** when Unload is ticked.

   The last two apply unless loading calc is manual. Unloading reusing the
   loading amount is a suspected defect (§28 Q10). Port it as-is behind one
   named constant, so the fix is one line.
4. `distribute(lines, overrides, netTotalOf)` (§12.3). This is two-pass:
   after-tax DEDUCT rows are measured on each line's net total.
5. Finalise every line with its final charge shares.
6. Header: `taxable = Σ line taxable + charge taxable`, and the same for
   CGST/SGST/IGST/cess. `gstAmount = cgst + sgst + igst` (cess excluded).
   Discount totals. `totalWeight = Σ weight × billQty`. **`totalBags = Σ billQty`.**
7. **Bill amount:** `amount = Σ line total + charges' own GST`. The Qt code also
   subtracts a "free scheme amount", which is always 0 now that free lines
   price at 0. Drop it.
   `bill = step > 0 ? floor((amount + step/2) / step) × step : amount`
   (half-up to `policy.roundOffStep`); `roundOff = bill − amount`.
8. Invariant (assert in dev builds): `bill = header taxable + header tax +
   after-tax flat (tax_apl = 0) charges + roundOff`. The server checks the same
   identity at post, including `cdSepPost` charges.
9. Re-clamp adjustments to the new bill (§14.2), then refresh the settlement
   display and schedule the autosave.

### 10.4 Stats line

"Total Saving" = Σ (mrp − netPrice) × netQty over lines with an MRP, with its
% of the MRP value. "Total Profit" = Σ profitPreTax × netQty, and "% margin" =
that ÷ Σ taxable. The header's `sbMarginPerc` uses a **different** definition,
Σ profit ÷ Σ gross. Keep both exactly as they are; they are different figures.

---

## 11. Promotions

Engine, rules and binder behave as in `promotion-engine-client` (memory) and
are shared with the quotation. The bill-specific parts:

- **Fetch:** GET `/promotion-scheme/list?prm_comp_id=<company>` returns the
  full graph. **Never pass `prm_branch_id`**: it matches the column literally
  and drops company-wide (NULL-branch) schemes. The `api_endpoints.h` comment
  calling it "headers only" is stale. Numerics arrive quoted. It is cached per
  (company, branch, bill date). A failure is silent (no modal), and promotions
  simply do not apply.
- **Context:** `{compId, branchId (the document's), docDate = the BILL date
  (never today), docTime = now, docType 'SALE_BILL', custId, areaId,
  priceLevelId, billType = term or 'ALL', promotionsEnabled = Promo tick}`.
  `customerGroupId` and `cityId` are not available (customer-detail lacks
  them), so CUSTOMER_GROUP and CITY rules never match (§27).
- **Skipped** on a loaded (stored) bill and in browse mode.
- **Order:** the promotion pass runs **before** the document pass at every call
  site: term, bill date, qty, size, rate, scheme flag, any discount, the Promo
  tick, the customer, and a cache refresh.
- **Basis from inputs, never from derived line fields.** The engine runs
  before the recompute, so reading netGross or taxable would lag one edit and
  feed back the previous discount. The Qt binder mirrors only standard
  pricing; with discount-first on, its basis differs slightly. **In React both
  use one `lineGoods()` function**, which removes the drift.
- **Free lines** are keyed `schemeId|itemId|unitId`. Update the qty in place,
  remove the unwanted ones, insert new ones below the last line stamped with
  that scheme. They are never cleared and re-inserted (that causes a lookup
  storm). They have `schemeFlag: false`, so a giveaway cannot trigger a scheme.
  Operator-added free lines (no scheme id) are untouched. Each inserted free
  line is priced through the normal lookup.
- **Hint:** a tooltip on the scheme-discount figure and the Promo tick.
  "Promotions are switched off for this bill." / "Applied:" list / "Not
  applied:" list / "No promotion scheme covers this bill." The caption gets
  " ⓘ" only when promotions are on and there are notes.
- The server warns `SALES_PROMO_NOT_LIVE` per scheme at post. No
  `promotion_usage` API exists, so caps and budgets are advisory.

---

## 12. Charges

### 12.1 Grid and master

Charge grid ui_table **21**, picker grid **82** with param `imodule_name = 'S'`.
One row per `charge_detail`. The only inputs are Rate (not on FIXED rows),
Amount, Qty, Weight, Unit and Remarks. **An Amount edit sets Rate to 0**
(manual-amount mode) and makes a carried row MANUAL. Only Rate and Amount
edits recompute.

Add from master: GET `/charges/get?chgId`. Snapshot `chgName, chgRole,
chgMethod, chgType, chgApplyOn, chgLedgerCode, chgLedgerName, chgLandingCost,
chgCostAlloc, chgBeforeTax, chgTaxApl, chgSepPost, ledHsnSac, chgDefaultRate`.

> **Defect (found in the backend source 2026-09-25, not yet verified live):**
> `/charges/get` stopped returning `ledGstRate` on 2026-09-12 and returns only
> `chgTaxId` / `chgTaxName`. The Qt controller still reads `ledGstRate`, so a
> freshly picked **after-tax taxable** charge gets 0% GST, and `cdTaxCode` is
> never filled. **The React port does not read `ledGstRate`.** It resolves the
> rate from `chgTaxId` (or from a field the server adds) and sends
> `cdTaxCode = chgTaxId`. Report it (§27 CHG-TAX).

Keep the ledger rate in its own field (`ledgerTaxPerc`). The Qt code zeroes
TaxPerc on rows without own GST, and TaxPerc is its only copy of the rate.

### 12.2 Auto-apply (new and cleared bills only)

The seed runs only when **no** row has a charge. It reads GET
`/charges/get?chgModule=S` once per session (it returns module and `B` rows).
Filter: `chgAutoApply` true, `chgIsActive` (default true), not deleted.
Stable sort by `chgDispOrder`, then name. It re-checks "still empty" when the
reply lands, because a load or import may have claimed the grid meanwhile. A
failure is silent. A load or import **replaces** the charges, so the seed never
survives onto a loaded bill. A seeded LOADING/FREIGHT row satisfies the save
role gate (§17.3).

### 12.3 `distribute` (the shared engine, recapped for the tests)

Per row, sign = DEDUCT ? −1 : +1. The basis per method is QTY → billQty,
NET_QTY → netQty, PERCENT → value, KG/QTL/TON → weight. **Qtl and Ton use raw
weight with no ×100 / ×1000 scaling;** port as-is and flag it (§28 Q10). For
FIXED rows the basis is by ApplyOn (FLAT → equal, QTY, WEIGHT, VALUE).

Shares come from the first of these that applies:

1. the role override;
2. manual amount → prorate;
3. PERCENT → basis × rate/100;
4. other rate methods → basis × rate;
5. FIXED → prorate(rate).

Prorate gives any rounding residue to the **last** line. Before-tax shares go
into item taxable. Own GST applies **iff `taxApl && !beforeTax`** (inter → IGST;
local → CGST = SGST = half). Otherwise the tax block, including TaxPerc, is
zeroed.

The passes are ordered:

- pass 1 = every row except after-tax DEDUCTs, measured on netGross;
- pass 2 = after-tax DEDUCTs, measured on each line's **net total**. They are
  not compounding and do not depend on row order.

### 12.4 Payload and carry

`charges[]` per real row, `cdSlno` 0…:

- **Scope:** `cdCompId, cdBranchId, cdAccYear` (null when empty) and
  `cdVoucherNo` (null on a bill).
- **Master snapshot:** `cdChgId, cdChgName, cdRole, cdMethod, cdType,
  cdApplyOn, cdLedgerCode, cdLandingCost, cdCostAlloc, cdBeforeTax, cdTaxApl,
  cdSepPost`.
- **Amounts and tax:** `cdUnit, cdQtyVal, cdWeight, cdRate, cdAmount,
  cdTaxCode, cdHsn`, the % and amount of tax, SGST, CGST, IGST and cess,
  `cdNetAmt`, `cdRemarks`, `cdIsActive:true`.
- **Audit:** `cdCreatedBy` on a new row; `cdId` + `cdModifiedBy` on an
  existing one.

`cdDocType` is **not** sent. Removed rows are simply absent. Numerics on
`/get` may arrive quoted; parse leniently.

**Carry (order → bill):**

- On order import, stamp every row's carry facts `{srcCdId: its cdId,
  srcAccYear: the order's year, basis: PRORATA, orderAmount}` **before**
  clearing its id.
- `/validate` answers `proposals.charges[] {cdSrcCdId, cdSrcAccYear, chgName,
  orderAmount, carriedSoFar, proposed, basis, isFinalBill}`. Apply them to rows
  whose basis is not MANUAL/NONE: if |amount − proposed| > 0.005 → amount =
  proposed, rate = 0.
- Per-row basis menu: PRORATA · FULL (`max(0, orderAmount − carriedSoFar)`) ·
  MANUAL · NONE (0).
- Emit `cdSrcCdId / cdSrcAccYear / cdCarryBasis` **only on carried rows**. The
  quotation DTO 400s on those keys, even as null.

Save-time: every real row needs a ledger. Otherwise: *'Charge "X" has no
posting ledger — fix it in the charge master, or remove the line with CTRL+-.'*

---

## 13. Sources: quotation, order, challan

### 13.1 Quotation import (Ctrl+F3). It **replaces**

1. On a bill with lines, confirm *"This bill already has items. Discard them
   and import a quotation?"*.
2. Picker = the quotation list. GET `/quotations/get?sqId&sqCompanyId&sqBranchId&sqAccYear`.
3. Reset, then:
   - **Trail:** `srcDocType QUOTATION`, `srcDocId sqId`, **`srcDocYear =
     sqAccYear`** (the quotation's own year, because every transaction table is
     partitioned by year), refno, date.
   - **Customer:** the `sqCust*` snapshot. **PIN cleared** (a quotation has
     none; the lookup fills it). POS from `sqCustStcd` → `sqPosStcd`.
   - **Header:** option ticks (no confirm), salesman (element 1), agent, level,
     terms text, policy snapshot. **Term is not taken from a quotation.**
   - **Lines:** every non-deleted line **at its quoted figures**. Size is set
     verbatim **before** the quantities, so nothing recomputes it. Stock and
     batch are not on a quotation; they fill on the next touch.
   - **Charges** replaced, with their ids cleared (the bill writes its own
     charge_detail).
4. One recompute under the bill's context, then customer-detail and
   party-context.

**Defect not to port:** the picker has **no status filter**, so the same
quotation can be billed twice (§28 Q3). The React picker shows open
(non-converted, non-cancelled) quotations only, or the server must refuse a
second conversion.

### 13.2 Sales order, whole-document import (Ctrl+F4 on an empty bill). It **replaces**

- Picker = pending sales orders. GET `/sale-orders/get?soId&soCompanyId&soBranchId&soAccYear`.
- Refuse on `soStatus ~ CANCEL` or `soFulfilStatus ~ COMPLETED`: *"This order
  is cancelled / fully delivered — there is nothing left to bill on it."*
- **Header:**
  - trail `SALES_ORDER` / soId / **soAccYear** / refno / date;
  - the customer snapshot, **including the PIN**, and POS from `soCustStcd`;
  - option ticks, salesman, agent, **packed-by**, level;
  - **Term = CREDIT iff `soOrderType === 'CREDIT'`**, then due;
  - terms text and policy.
- **Lines, at the PENDING qty:**
  - `pending = soiPendingQty > 0 ? soiPendingQty : max(0, orderQty −
    deliveredQty − cancelledQty)`. Skip `pending ≤ 0` lines and count them;
  - `orderQty = billQty = pending`;
  - CaseQty and LengthQty are **not** imported;
  - Size only on a **first whole delivery** (nothing delivered or cancelled
    yet). On a part delivery, re-committing the size would push the qty back
    to the full order and trip the cap;
  - godown and godown name, per-line salesman, source line id **and source
    line number**.
- **Charges:** populate, **stamp carry facts**, clear ids.
- **Advance carry:** if `soAdvanceBalanceAmt > 0`, the adjust panel prefills
  from the source document (by `srcDocId`, in the **order's** year — *"a March
  advance really does settle an April invoice"*). Once the panel has loaded,
  say: *"This order holds an advance of X, which has been set off against this
  bill. Collect only the balance at settlement."*. The server also applies the
  order's own advance first at post.
- Info *"N line(s) were already delivered in full and were not imported."*.
  If nothing is left: *"Every line on that order has already been
  delivered."*.
- **Do not port:** `AllowNegative = true` hard-coded on every imported order
  line (and on challan lines, and on the load path). It bypasses the client
  stock gate. Use the item's flag and today's stock (the lookup refills both)
  (§28 Q4).

### 13.3 Append from orders or challans (Ctrl+F4 on a bill with lines; Ctrl+F5)

- A second order needs `sales.multi_order_bill`. Otherwise: *"This company
  bills one order per bill. Clear the bill first, or turn on
  sales.multi_order_bill."*
- Needs a customer: *"Pick the customer first — only their open documents are
  offered."*
- The Open Sources dialog **appends, never replaces**.

### 13.4 Open Sources dialog

GET `/bills/open-sources?companyId&branchId&partyId&kind=DC|ORDER&accYear?`
(BARE keys). DC lists POSTED challans with open qty. ORDER lists
CONFIRMED/PARTIAL orders with pending qty.

```
[{ docId, accYear, refno, date, ageDays, pastWindow, purpose (DC), convertRequired?,
   lines: [{ lineId, lineNo, itemId, itemName, unitId, unitName, lotId, batchNo, godownId,
             docQty, openQty, freeQty, rate, taxId, taxPerc, hsnCode }] }]
```

- Tree: document (tristate tick; **red + "— past the return window"** when
  `pastWindow`; header "refno · dd-MM-yyyy · N day(s) old · purpose") → lines
  (tick + Take 0…openQty, default openQty, 3 dp; typing > 0 ticks, 0 unticks).
- Summary: "N line(s) ticked · goods value X". An error is shown as status
  text, not a modal.
- **Each picked line:**
  - **same source line already on the bill** → top up to `min(have + take,
    orderQty)`, then the qty rules ("a double-bill waiting to happen");
  - **new line** → `srcDocType` (DELIVERY_CHALLAN | SALES_ORDER), `srcDocId`,
    **`srcItemId = lineId`** (soi_id / sdi_id), **`srcLineNo = lineNo`**,
    `srcDocYear`, `orderQty = openQty` (the cap), `billQty = takeQty`, `rate`,
    `taxPerc`, level = the document's, then the lookup for the rest.
    **Does the lookup overwrite the source rate?** In Qt, yes. Decide (§28 Q5).
- The header trail is set only by the first document. Every document's refno
  is kept for the chip.
- Challan lines **do not move stock again** on the bill (the server skips
  stock for lines from a DC).

### 13.5 The source trail is per line

`sbiSrcDocId` = the **document**. **`sbiSrcItemId`** = the source **line**
(soi_id / sdi_id): the post guards and the open-qty draw-down key on it.
**`sbiSrcDocLineNo` must be sent.** It was always null once, and billed orders
stayed CONFIRMED. The Qt model once held the line id in `SrcDocId`; port only
the corrected model. A line without a trail sends nulls, never the header's.

### 13.6 Source chip and list

The identity strip lists every source document once, in first-seen order:

- a coloured tag per kind: `SO` #1a5fb4, `DC` #7048e8, `QT` #0b7285;
- then the refno and "n line(s) · X taken" (from `/get` `sources[]`);
- empty: "none — a hand-keyed bill";
- an older bill with only a header trail shows the header's.

---

## 14. Adjustments: setting off advances and credit notes

### 14.1 One state, two entry points

There is **one** set of adjusted amounts per bill, held in `draft.adjustments`.
Both **F4** (a modal on the bill) and the tender dialog's **ADJUST row** render
the same `<AdjustPanel>` bound to that state. The Qt screen lends one widget
instance back and forth between them (re-parenting, `destroyed()` hooks).
That was the Qt way to avoid two copies, and the React state already
guarantees it. **Do not port the borrowing.**

Direction is `CR` for a bill (it sets off what we owe the customer). `DR` is
reserved for the sale return.

### 14.2 Fetch, rows, clamps

- GET `/transactions/party-balance?partyId=<customer>&companyId&type=CR`. There
  is no branch and no year, by design. Rows come oldest first: `billId,
  docRefno, docDate, billType (ADVANCE | SALES_RETURN), billAmount,
  pendingAmount, billAccYear, adjType, settlementMode, status, srcDocType,
  srcDocId ({} when null), srcAccYear, narration`.
- **Stale-reply token** keyed by party. A reply for the previous customer is
  money shown against the wrong party.
- `pending = pendingAmount + heldByDocument[billId]`: a **posted bill being
  amended gets its own set-offs back**. The live pending excludes what this
  bill holds, so without this an amend re-sends a short set-off. Credits the
  bill used up entirely are rebuilt from its saved rows. Rows with
  `pending ≤ 0.005` are dropped.
- Grid ui_table **25**. Visible: Credit (refno), Date, Kind ("Advance" /
  "Credit Note", tinted), Pending, **Adjust** (the only input, 2 dp, ≥ 0, no
  exponent), Remarks. Tooltip: "refno · year · value X\nnarration".
- On commit:
  - `adjust > pending` → *"<ref> has only <pending> left on it."*, and the
    cell resets to 0;
  - `adjust > room` (room = bill − the other rows) → confirm *"This document
    can only take X more. Adjust X against <ref>?"* (yes clamps, no → 0).
- `setDocumentTotal(bill)` runs after every recompute. If Σ adjusted > bill,
  **trim FIFO** (keep the oldest, cut later rows). A line delete can never
  leave the bill over-adjusted; the server's `ck_abl_settled` would refuse it.
- **Adjust All** fills FIFO up to the bill.
- `release(amount)` gives back **newest first** (§15.5).
- Empty states:
  - no customer: "Pick a customer to see what they already hold.";
  - none open: "This customer holds nothing to adjust.";
  - fetch error, **never modal**: "Open credits aren't available just now."

### 14.3 F4

It is independent of the tender route and the term: a credit bill never
opens the tender, and its customer is the one most likely to hold an advance.

- Refused in browse mode: "This bill is read-only."
- Refused with no customer: *"Pick the customer first — credits belong to a
  party."*, and focus goes to the customer.
- Refused when the bill total is ≤ 0: *"There is nothing on this bill to
  adjust against yet."*.

Modal "Adjust Advance / Credit Notes": Apply validates; **Cancel restores the
snapshot taken on open**.

### 14.4 Absent ≠ empty (the rule that loses money if missed)

- `authoritative = loadedAdjustments.length > 0 || sbAdvanceAmt ≤ 0.005`.
- When **false** (a bill settled with credit whose adjustments did not come
  back), the save payload **omits the `adjustments` key entirely**. `[]` would
  reverse every set-off. The panel shows *"This bill was settled with X of
  credit that the server does not return yet. It is shown as a total only and
  cannot be changed here."*.
- Any real edit in the panel makes it authoritative again.
- **Draft set-offs are not stored server-side.** `/create` only checks them,
  and a reloaded DRAFT's `/get` has `adjustments: []`. The client keeps them in
  the draft (and in the hold snapshot), and **sends them again on `/post` and
  `/amend`**. Without them on `/post`, the server picks the party's open
  credits oldest-first, so the advance the operator chose is silently swapped
  for another (live test 2026-09-24).
- A posted bill's `/get` adjustments `{againstBillId, againstBillAccYear,
  refno, amount, adjType}` are the live set-offs. They feed `heldByDocument`.
  **Their shape is not the save DTO** (`refno` would 400 on an echo).

### 14.5 Figures and wire shape

- `advanceAdjusted` = Σ ADVANCE rows (the panel's figure once loaded and
  authoritative, else the loaded `sbAdvanceAmt`; it is a fallback and **never
  a sum**, or an import counts twice), capped at the bill.
  `noteAdjusted` = Σ SALES_RETURN rows. `totalAdjusted = min(advance + note, bill)`.
- **`sbAdvanceAmt` = ADVANCE only** (it feeds the order's advance ledger) and
  **`sbNoteAdjAmt` = credit notes only**. The server refuses a single total
  across both, and each must equal its own type's rows ±0.01
  (`SALES_AMOUNT_MISMATCH`).
- Wire row: `{againstBillId, againstBillAccYear, amount, remarks?, billType,
  adjType, settlementMode}`. The last three are echoes the server ignores;
  it re-derives them. Rows ≤ 0.005 are not sent.
- The snapshot form (hold / autosave only) adds `docRefno, docDate,
  pendingAmount, srcDocId` for an offline repaint. They are never sent to
  `/create`.
- Restoring a held bill **reconciles** against today's open credits. Rows no
  longer open are dropped; rows with less left are reduced. **One** notice
  lists them all: *"Some credits changed since this bill was put aside:\n\n…"*.
- The server writes set-offs as bill-wise allocations only. There is no
  ADVANCE_RECEIVED leg any more (`sales_posting_flow.md` §5.2 rows 9–10 are
  stale).

---

## 15. The tender dialog (settlement)

### 15.1 When F5 opens it

`sales.tender_type` ∈ `none | cash_bills | all_bills` (default `all_bills`):
`tenderRoute = all_bills || (cash_bills && term !== 'Credit')`. On the tender
route the bar shows **Tender (F5)** only; the dialog has its own Save (F5) and
Save & Print (F6). `sales.tender_print_only` disables (never hides) the
dialog's Save.

Opening sequence:

1. Browse → "This bill is read-only."
2. Client checks (§17.3) with `openingTender = true`. That **skips** the
   "settled + adjusted > total" check, because re-opening the tender is how
   that mismatch gets fixed (user 2026-09-24).
3. **POST `/bills/validate` with the whole payload.**
   - OK → paint the notes in the strip (no popup) and open with them.
   - Error → if any note is a refusal, paint it (with popup) and **do not
     open**. Counting money for a bill that cannot post is wasted.
   - Any other failure (server away) → **open anyway**. A sale must not stop.

Context handed in:

- **Amounts:** `grossAmount` (the bill as raised) · `billAmount = max(0,
  gross − totalAdjusted)` (only the balance is settled at the counter).
- **Customer:** `creditAllowed` · customer / company / branch / year ·
  `isWalkIn` · bill-to name, phone, place and address (for "copy from
  bill-to" and the cheque drawer default).
- **Loyalty:** `loyaltyPoints` · `loyaltyAllowed = customer allowLoyalty &&
  Loyalty tick` · the `loyalty` block.
- **Checks:** `cashToday` · the validate notes · `openTempCredits` · `custPan`
  / `form60Ref`.

### 15.2 Layout

Title "Settle Bill" ("Receive Advance" for the order).

- **Left column** (min 280):
  - heading "Tender";
  - a one-column amount table. The row label is "K. Name" with a colour dot
    (hotkey letter + master name);
  - the cash-limit line (§15.8 A) and the identity box (B);
  - the chips: Bill Total #1E88E5 · Service Charge #FB8C00 · Tender #43A047 ·
    Balance/Refund (coloured by state) · Adjusted #8E24AA (only when > 0).
- **Right column, fixed 560 px** (user 2026-09-25: the window never resizes as
  rows change). Exactly one of: blank · instrument panel · loyalty panel ·
  temp-credit panel · the adjust panel. On a narrow screen stack the columns;
  never resize on row change.
- The hint line starts as "F1 - Auto fill Balance". A per-row hint replaces
  it.

Bill Total shows `grossAmount` (the invoice as raised). The purple Adjusted
chip explains the gap.

### 15.3 Rows from the tender master (`masterRow.ts`, shared with the receipt)

GET `/tender-masters/list` (no params). Keys read: `tndId, tndTypeId,
tndTypeName, tndName, tndShortName, tndLedgerId, tndSettlementLedgerId,
tndSettlementDays, tndMinAmount, tndMaxAmount (0 = unlimited), tndDailyLimit,
tndSurchargePerc, tndSurchargeAmount (flat), tndSurchargeLedgerId,
tndEditSurcharge, tndUpiVpa, tndMerchantId, tndTerminalId, tndConversionRate
(≤ 0 → 1), tndNeedsRef, tndAllowChange, tndAllowInReturn` (nullable overrides:
the API sends `{}` for NULL, meaning inherit the type default), `tndIsDefault,
tndDisplayPosition, tndHotkey, tndColour, tndEffectiveFrom/To, tndIsDeleted,
tndIsActive`.

**Behaviour comes from the master, never the type name.** Type defaults:

| type id | type | needsRef | refLabel | gives change |
|---|---|---|---|---|
| 1 | CASH | n | – | **always** |
| 2 | CARD | Y | "Slip No" on this screen | n |
| 3 | UPI | n | UTR No (**optional**, user 2026-09-25) | n |
| 4 | WALLET | Y | Txn Ref | n |
| 5 | CHEQUE | Y (panel regardless) | Cheque No | n |
| 6 | BANK | Y | UTR / Ref No | n |
| 7 | ADJUST (legacy name RRN; accept both) | n | – | n — **mirror only** |
| 8 | TEMP_CR | own panel (see D1) | – | n |
| 9 | CREDIT | n | – | n |
| 10 | LOYALTY | n | – | n |
| 11 | VOUCHER | Y | Voucher No | n |

- **Offerable:** not deleted, active, and inside the effective dates **as of
  the bill date**. (Qt uses the session's app date, although its comment says
  the voucher's date. Use the bill date, and flag the divergence.) Sort by
  display position, then name.
- **Fallbacks, when nothing is offerable:** seed CASH (+ CREDIT) with no
  ledger, and replace the hint:
  - "No tender is configured — add one in the Tender Master.";
  - "No configured tender can be used here — check the Tender Master's active
    dates.";
  - "Tender master unavailable (…) — only cash can be taken.".
- ADJUST and LOYALTY cells are **not editable**; they mirror their panels.
  CREDIT for a non-credit customer stays editable with a tooltip ("…putting
  the bill on credit will ask for confirmation.").
- **Start row:** the first default-and-usable row, else the first usable one.
  CREDIT is unusable when credit is not allowed.
- **Hotkeys:** the master's `tndHotkey` wins. On a collision the first
  configured row keeps it. Others get the next free letter of A–L, and more
  than 12 unkeyed rows get none. The keys work while focus is in the tender
  list, not while typing in the right panel.
- **Re-open:** match saved lines to rows **by tender id** (type name as the
  fallback). Loaded lines carry no type name.

### 15.4 Arithmetic (`settle.ts`), in paise

```
surcharge(m, base, perc) = base ≤ 0 ? 0 : base × perc/100 + m.surchargeAmount   // flat fee once used
row.amount   = base + surcharge                                                  // the cell keys BASE, shows GROSS
tendered     = Σ amount       over rows except ADJUST
surchargeSum = Σ surchargeAmt over rows except ADJUST
balance      = tendered − surchargeSum − billAmount     // billAmount re-nets live as set-offs change
```

The surcharge is the bank's cut. It settles nothing, so it is netted out of
the balance, and **every comparison against the bill uses the base.**

- **While typing**, the chips update live and the cell text is left alone.
- **On commit** of an amount:
  1. ADJUST / LOYALTY → open their panel, stop.
  2. CREDIT with amount > 0 → the credit confirm (§15.7). Declining resets the
     row.
  3. `maxAmount > 0 && base > max` → *"<name> cannot take more than <max>."*,
     reset the row **and recompute** (Qt forgets the recompute; G5).
  4. Set the row, recompute, show its panel.
  5. **Release set-off for excess** (§15.5).
  6. amount > 0 → move focus to the first empty field of the row's right panel.
- **F1** fills the current row with the shortfall:
  `base = amount − surcharge − balance`. On the ADJUST row it opens the panel
  instead. On CREDIT it asks the credit confirm. **Run the release rule after
  F1 as well**; Qt runs it only on a grid commit (G1).
- **Balance chip:**
  - 0 → grey "Refund";
  - < 0 → red "Balance" ("Balance to Collect" amber for an advance);
  - > 0 → green "Refund".

  Compare in **paise**; Qt compares floats exactly (G7).

### 15.5 Set-off is given back before change (user decision, 2026-09-25)

After any amount commit on a settlement: `excess = balance; adjusted = total
set-off`. If both are > 0.005 → `release(min(excess, adjusted))` newest first,
re-net, and hint *"The money tendered covers the bill, so X of the credit
set-off was given back to the customer's advance."*.

A credit is never spent **and** handed back as change; only the excess beyond
the set-off becomes change. This was learned the hard way: an advance of 300
was set off, cash of 400 was typed, 300 went out as change, and the advance
was gone (bil00702, bil00715).

### 15.6 Change routing on Save

`change = balance`. Walk the rows in display order:

- the **first change-capable row that is at least the change** takes
  `refundAmt = change`;
- CREDIT rows add to `creditAmt`;
- **TEMP_CR rows also count as credit** (D5, below);
- the ADJUST row is skipped: **no tender line, ever**;
- a line is emitted iff `amount > 0 || refundAmt > 0`.

If change is still left over: *"No tender on this bill can give change back.
Reduce the over-tendered amount, or add a cash tender."*, and the dialog stays
open.

(Qt gives change to the first change-capable row **even at amount 0**. With
excess tender allowed, that sends a CASH line with a negative total (D3). Do
not port it.)

Output: `{lines, tender = Σ amount (excl. ADJUST, incl. surcharge), credit,
refund = max(0, balance), surcharge, identity {pan, form60}, print}`. For the
advance purpose, `refund` is clamped at 0 too. Qt emits a negative refund on a
short advance, which inflates the order's paid figure (D6).

### 15.7 Per-type panels and checks

**Instrument panel** (CARD, CHEQUE, any row that needs a reference or has a
surcharge):

- **Amount block:** Amount (editable, = base) · Service % (editable iff
  `editSurcharge`) + amount · Total.
- **Bank + date** for CARD and CHEQUE:
  - bank = dropdown 46;
  - CARD date = "Expiry Date" MMM/yyyy;
  - CHEQUE date = "Cheque Date" dd-MM-yyyy, **defaulted to today in state, not
    just in the input**. An undated cheque cannot be judged post-dated.
- **CARD:** Card No, and **only the last 4 digits go on the wire**
  (`tdCardLast4`, exactly 4 digits else null; the full number never leaves the
  screen).
- **Reference:** "Slip No" for CARD, else the type's label. Visible iff
  needsRef or CHEQUE. UPI placeholder "optional".
- **CHEQUE extras:** Drawer (defaults to the bill-to name, ≤ 150) · Bank
  Branch (≤ 100) · IFSC (upper, `^[A-Z]{4}0[A-Z0-9]{6}$`, optional) · MICR
  (`^[0-9]{9}$`, optional) → `cheque{}` (notes 48, live).
- **Hint:** "VPA x · MID x · TID x · settles T+n · daily limit x".

**CREDIT:** no panel. If credit is not allowed: *"This customer is not allowed
to buy on credit. Put this bill on credit anyway?"*, remembered for the
dialog. Any credit > 0 flips the bill's term to Credit on accept.

> **D2:** the bill's own save check then refuses it ("This customer is not
> allowed to buy on credit."), so "credit anyway" is a dead end today. Decide
> (§28 Q1): either carry the override into the save check and let `/validate`
> rule, or do not offer it.

**TEMP_CR** ("Temp Credit — who is borrowing"):

- **Fields:** Name* (≤ 100) · Mobile* (≤ 15, `[0-9 +]`) · Place · Address ·
  Pay in N days (1…`sales.temp_credit_max_days` or 365; default min(7, max);
  due = app date + N) · Notes · **Copy from bill-to** · a warning line (amber).
- **Open-credit warning:** for each of the party's open temp credits on the
  same mobile, show *"This mobile already has an OPEN temp credit: <bill> ·
  <balance> · due <date>"* (+ " — Post will refuse" on REFUSE).
- Read **`sales.temp_credit_block_open` as an enum** `OFF | WARN | REFUSE`:
  block only on REFUSE, warn on WARN.
  - **D7:** Qt reads it as a bool, so it never blocks.
  - **D8:** match mobiles the way the server does (exact, after normalising)
    and **skip this bill's own temp credit**. Qt compares the last 10 digits and
    self-blocks on an amend.
- **D1:** exempt TEMP_CR from the generic reference check. The type default
  says `needsRef`, and the panel has no reference field. The server overwrites
  the reference with the mobile anyway.
- The walk-in cannot owe money without a TEMP_CR row
  (`SALES_TEMP_CREDIT_DETAILS_MISSING`).

**LOYALTY:**

- **Rate:** `rate = member && scheme rate > 0 ? scheme rate : master rate || 1`.
- **Points:** `points = member ? balance : ctx points`;
  `redeemable = member ? min(redeemable, points) : points`;
  `worth = redeemable × rate`;
  `minPoints = member ? scheme min : master min / rate`.
- **Usable** iff loyalty is allowed, redemption is allowed, `worth > 0` and
  `redeemable ≥ minPoints`. Otherwise the inputs are read-only.
- **Hints:**
  - walk-in → "Walk-in — no member. Add the customer to earn points.";
  - loyalty off → "Loyalty is switched off for this customer (customer master)
    or unticked on this bill — N points can't be redeemed here." / "This
    customer is not on the loyalty scheme.";
  - redemption off → "The scheme does not allow redemption on this bill.";
  - below the minimum → "Needs at least N points to redeem.";
  - otherwise "card X · max Y on this bill · in multiples of N pts".
- **Redeem amount:** capped at `worth`, and at `maxRedeemAmount` for a member.
  Over the cap: *"The scheme allows at most X on this bill."* / *"Only X of
  points is available."*, reset to 0.
- **Points:** `points = amount / rate`, floored to `multiple`, and the amount
  is rewritten.
- **Payload:** `tdConversionRate` = **the rate used (the scheme's)** and
  `tdUnitsUsed` = **the points**.
  - **D4:** Qt sends the master rate and base / master-rate. The server checks
    points × scheme rate = amount and refuses `SALES_LOYALTY_CAP` whenever the
    two rates differ.
- **Header:** `sbLoyaltyMemberId = loyalty.memberId`, or the server refuses
  any LOYALTY tender.
- **Points input:** the Qt points field is unreachable (NoFocus). Decide
  whether points are keyable (§28 Q7).
- **Server-side:** earn happens at post (`posting.loyaltyEarned`). A walk-in
  never earns.

### 15.8 Additions A–E

- **A. Cash-limit line (269ST):**
  - text: "Cash today from this party <cashToday> + this bill <cash> = <sum>";
  - with a `SALES_CASH_LIMIT` note, append " — limit <value> (<code>, from
    <date>)" in red bold, else grey;
  - hidden when both figures are 0;
  - **never hard-code the figure** (law params are per company).
- **B. Identity box:**
  - visible when a `SALES_PAN_REQUIRED` note exists or the bill already
    carries PAN / Form 60;
  - PAN `^[A-Z]{5}[0-9]{4}[A-Z]$` (upper-cased) or a Form 60 reference;
  - → `sbCustPan` / `sbForm60Ref`.
- **D. Daily limit:** shown in the hint, **not** checked client-side (the server
  warns `SALES_TENDER_DAILY_LIMIT`).
- **C and E** are the loyalty pair and TEMP_CR above.

**Checks on Save, in this order** (each focuses its field):

1. Settlement and balance < 0 → "The bill is not fully tendered."
2. Adjust panel invalid → its reason.
3. Identity visible and both empty → *"A cash sale above the limit needs the
   customer's PAN or a Form 60 reference."*; a bad PAN → "That is not a valid
   PAN (AAAAA9999A)."
4. Per used row (skipping ADJUST):
   - min amount → "<name> needs at least <min>.";
   - no change and `base > billAmount` without `sales.allow_excess_tender` →
     "<name> cannot exceed the bill amount — it can't give change back.";
   - reference required (`needsRef && !UPI && !TEMP_CR`, or CHEQUE) → "<name>
     needs its <label>.";
   - CHEQUE bank → "<name> needs the bank the cheque is drawn on.";
   - TEMP_CR name + mobile → "Who is taking the goods on credit? Name and
     mobile are mandatory."; `temp_credit_max_amount` → "Temp credit is capped
     at X per bill."; open-credit block (REFUSE);
   - LOYALTY over worth.

Esc closes without settling. Set-offs made in the dialog **persist**, because
they are the bill's state.

### 15.9 After Save: settlement roll-ups (`rollups.ts`)

```
onSettled(lines, tender, credit, refund, surcharge):
  counterPaid = tender − surcharge − refund − credit          // credit includes TEMP_CR
  payMode     = the type of the line with the largest amount
  if credit > 0 and term is Cash → term = Credit, re-derive due (no promotion re-run)
  then save (the dialog's Save) or save & print (F6)
```

Header figures on the payload:

| Key | Value |
|---|---|
| `sbTenderAmt` | tender |
| `sbSurchargeAmt` | surcharge |
| `sbRefundAmt` | refund |
| `sbCreditAmt` | credit (**CREDIT + TEMP_CR**, D5) |
| `sbAdvanceAmt` / `sbNoteAdjAmt` | §14.5 |
| `sbPaidAmt` | `counterPaid + totalAdjusted` |
| `sbBalanceAmt` | `bill − sbPaidAmt` |
| `sbPayStatus` | PAID (`paid ≥ bill > 0`) · PARTIAL (`paid > 0`) · UNPAID |
| `sbPayMode` | above |

**No tender lines** (plain route):

- Credit term → `credit = bill − adjusted`, tender = paid-at-counter = 0,
  payMode CREDIT.
- Cash term → `tender = counterPaid = bill − adjusted`, payMode = `ADVANCE`
  when the set-offs cover the bill, else `CASH`.

**On load, the counter part is `sbPaidAmt − sbAdvanceAmt − sbNoteAdjAmt`.** The
stored figure includes the set-offs. Before this fix an amend sent an advance
twice as "paid" and 500'd.

The server **recomputes** paid / balance / pay status / advance / note at post,
and again on re-tender. What the client sends is its proposal.

What `sbTenderAmt` should equal (base, received or total; change and
surcharge in or out) has **never been defined** (C4, §27). Send `Σ amount incl.
surcharge` as Qt does, until it is.

### 15.10 `tenders[]` (per line, `tdRowNo` from 1)

```
received = line.amount (incl. surcharge and over-tender); change = line.refundAmt
total = received − change;  base = total − surchargeAmt
{ tdRowNo, tdCompanyId, tdBranchId, tdAccYear, tdDocDate (bill date),
  tdTenderId, tdTenderTypeId (int, null if 0), tdTenderLedgerId, tdPartyLedgerId: <customer id>,
  tdDrCr:'DR', tdAmount: base, tdSurchargePerc, tdSurchargeAmt, tdSurchargeLedgerId,
  tdTotalAmt: total, tdReceivedAmt: received, tdChangeAmt: change,
  tdConversionRate, tdUnitsUsed,                      // loyalty: scheme rate + points (D4)
  tdRefNo, tdBankName, tdCardLast4, tdInstrumentDate (card expiry / cheque date),
  tdIsPdc: typeId === 5 && instrumentDate > billDate,  // by TYPE ID, not name (D9)
  tdSettleStatus:'NA', tdSettleLedgerId, tdExpectedSettleOn: billDate + settlementDays | null,
  tdDeviceId, tdUserId,
  tempCredit?: {name, mobile, place, addr, idRef, days, notes},   // TEMP_CR only
  cheque?:     {drawerName, bankBranch, ifsc, micr} }             // CHEQUE, when any is set
```

- **Never sent:** `tdSrcModule / tdSrcDocType / tdSrcDocId / tdVoucherId`. The
  parent implies them, and the server stamps them.
- The ADJUST row never appears. A redeemed credit names its document in
  `adjustments[]`.
- `tdPartyLedgerId`: Qt sends null ("customer lookup returns no ledger id
  yet"). But the customer id **is** the ledger id (house rule 1), and the
  server defaults it to `sbCustId`. Send the customer id, or omit it.
- On a DRAFT the server packs `tempCredit` into the row's spare columns
  (`tdBankName` = name, `tdRefNo` = mobile, …), so anything the client puts
  there on a TEMP_CR row is overwritten. `/get` decodes it back.
- Loaded tender lines carry **no type name**; key every rule on
  `tdTenderTypeId`. Qt tests the name, so an amend that does not reopen the
  tender sends `tdIsPdc: false` for a post-dated cheque (D9).
- Copy-as-new and Clear drop the tender lines.

---

## 16. The warning strip and the Validation popup

Shared by all six posting screens (`sales-shared/components/WarningStrip`).

### 16.1 Note shape and parsing

`{code, level: REFUSE|WARN|INFO, message, field ("tenders[0].tdAmount"),
line (1-based), overridable, isRefusal, statutory: {code, value, effectiveFrom,
isCompanyOverride}}`.

**Read every holder**, refusals first:

- `data.{refusals, warnings}` (the success envelope);
- `details.{…}`;
- `error.details.{…}`;
- a bare `{refusals, warnings}`;
- the 422 error envelope's `errors[]` (each with a `code` is a refusal).

Missing one of them was the bug this parser exists to prevent.

A successful validate carries `data.rights.override`. That is the **only**
source of the override right before a new bill's first `/get`.

### 16.2 Four rules

1. A refusal is never tickable, and confirmProceed never passes with one.
2. The Override tick shows only when `overridable && rights.override`.
   Overridable without the right shows a grey "needs override right".
3. The strip hides when empty.
4. **Ticks reset on every new answer.** Losing the right clears them.

The server re-checks all of it; this is an honesty boundary, not a security
one.

### 16.3 What the server does with a WARN

The same guard run serves `/validate`, `/post` and `/amend`. On `/validate` an
overridable WARN stays a WARN. **On `/post` and `/amend` it becomes a refusal,
unless its code is in `overrides[]` and the user holds `rights.override`.** So
`/validate` `ok: true` with WARNs means "post will fail unless you override
these". The confirm popup says so.

### 16.4 UI

- **Footer one-liner:** a "REFUSED" or "WARN" pill, "N refused · M warning(s)"
  in the worst colour, and a **View** button.
- **"Validation" popup:**
  - banner, red when refused: *"This document cannot be posted as it stands —
    a refusal is not something that can be overridden; the document has to
    change."*; amber otherwise: *"The server has something to say about this
    document. Read it; tick an override where you may and mean to."*;
  - one row per note: level pill · code (monospace) · "line N — message" or
    "field — message", plus " · 269ST · 2,00,000.00 · from 2017-04-01 (company
    override)" · the Override tick or its hint.
- **`confirmProceed(verb)`:** the same popup with **Back** and the verb ("Post"
  / "Post & Print"). The verb is hidden on a refusal, and focus goes to Back.
  With no notes, it returns true without showing anything.
- `overrides: [code…]` (de-duplicated) is sent on `/validate`, `/post` and
  `/amend`. **Ticking an override does not re-validate**; it rides on the next
  post.
- **A 422 with notes never also opens the generic error box.**

### 16.5 Notes that act on the screen

`SALES_EWAY_TRANSPORT_MISSING` / `GST_EWB_TRANSPORT_MISSING` → mark the
shipping dialog "required" with the message and **open it** (a WARN that cannot
be overridden). The server reads the band **from the database**, not from the
validate body, so an unsaved band warns falsely until the draft is saved (§27).

The server's "no customer" refusal uses the code `SALES_PAN_REQUIRED` with
`field: 'sbCustId'`. Show it by the field, not the code.

---

## 17. Verbs and the lifecycle

### 17.1 The verb bar is a pure function

```ts
verbState({ status, isNew, editable, amending, autoPost, tenderRoute,
            canPost, canAmend, canCancel, canEdit, amendBlockedReason, cancelBlockedReason })
```

`keying = status==='DRAFT'|'NEW' && editable && !amending`;
`saveVerbs = keying || amending`.

| Button | Visible when | Label | Enabled / tooltip |
|---|---|---|---|
| Tender | saveVerbs && tenderRoute | "Tender - F5" | |
| Save | saveVerbs && (!tenderRoute \|\| amending) | amending: "Save changes" (+" - F5" on the plain route) · autoPost: "Save - F5" · else "Save draft - F5" | |
| Post/Save & Print | saveVerbs && !tenderRoute | amending "Save changes & Print - F6" · autoPost "Save & Print - F6" · else "Post & Print - F6" | amending \|\| canPost \|\| isNew. Tooltip (not amending, not autoPost): "Post without printing: Ctrl+Shift+Enter" |
| Hold, Pick Held | keying | | |
| Clear | **always** | "Clear - F7" | |
| Delete | DRAFT && !isNew && !amending | | |
| Edit | (DRAFT && !editable) \|\| (POSTED && !amending) | "Edit - F2" | posted: canAmend, tooltip "Correct this bill. It keeps its number and date." or the reason; draft: canEdit, else "You don't have permission to edit bills." |
| Copy | !isNew && !amending | | |
| Bill List | !amending | | |
| Cancel bill | POSTED && !amending | "Cancel bill" | canCancel, tooltip "Reverse this bill. It keeps its number." or the reason |
| Close | always | "Close - Esc" | |

**Hidden** = never applicable in this state. **Greyed** = applicable but blocked
now, and the tooltip says why. In plain words:

- **Tender route:** Tender (F5) only.
- **Plain route:** Save (F5) · Save & Print (F6).
- **Posted:** Edit · Cancel bill · Copy · Clear.
- **Amending:** Tender (tender route) or Save changes / Save changes & Print ·
  Clear.

### 17.2 Rights and locks (`/bills/get`)

- `rights {post, cancel, amend, override, retender}`: these are the
  `user_menus` flags for **menu 12** for the calling user. They are not the
  generic screen permissions, which know nothing of posting.
- **A new bill has no rights yet.** Do not refuse Post on `!canPost` for a new
  bill; the server checks. A validate answer's `rights.override` feeds the
  strip.
- `locks {returns, allocations, dayClosed, irnLive, ewbLive,
  irnCancelWindowUntil, ewbValidUpto, editable: {document, transportBand}}`.
  All are derived server-side at read time.
- **Cancel's blocked reason**, in the order the operator must act:
  1. no right → "You don't have permission to cancel bills.";
  2. IRN live → *"An e-invoice (IRN) is live on this bill. Cancel the IRN first
     — the GST document always goes before the bill. Past the portal's window
     it cannot be cancelled at all, and the correction is a credit note."*;
  3. EWB live → *"An e-way bill is live on this bill. Cancel it first, or wait
     for it to expire."*;
  4. returns → "N sale return(s) point at this bill. Cancel those first.";
  5. allocations → "N receipt allocation(s) are set against this bill. Release
     them first.";
  6. day closed → *"The day is closed. A drawer already counted with this bill
     in it is a day-book correction, not a cancel."*.
- **Amend** = POSTED && !irnLive && !ewbLive && rights.amend. The reasons: IRN
  → *"This bill has an IRN, so it cannot be edited. Cancel it and make a new
  bill."*, likewise for EWB, and no right → "You don't have permission to edit
  posted bills." **Lock 2 (IRN or EWB) removes Amend entirely; no right brings
  it back.**
- **Never gate Amend on `locks.editable.document`.** The server sets it to
  `status === 'DRAFT'`, so it is false on every posted bill. It means "the
  fields are open right now" and drives read-only only.
- Read-only on load = `!locks.editable.document || !openForEdit`. It is never
  derived from the status.

### 17.3 Client checks before any server call (`validate.ts`, in order)

1. Browse → "This bill is read-only."
2. No customer → "Select a customer."
3. Invalid date → "Enter a valid bill date."; outside the year → "Bill date X
   is outside the accounting year (A – B)."
4. `salesman_mandatory` → "Select a salesman."
5. Quick Delivery ticked and remarks empty → "Quick delivery needs a remark
   (who takes it, where)."
6. Adjust panel invalid → its reason. Adjusted > 0 with total ≤ 0 → "There is
   nothing on this bill for the adjusted credits to settle."
7. **Not opening the tender**, tenders exist, and `!allow_excess_tender`:
   `tender − surcharge − refund + adjusted > total` → *"This bill is settled
   for X and adjusted by Y, which is more than its Z.\n\nRe-open settlement
   (F5) so the adjustment comes off what the customer pays."*
8. **Credit term:**
   - `!creditAllowed` → "This customer is not allowed to buy on credit."
     (see D2);
   - credit exhausted → **confirm**, not a block: *"The customer's credit
     limit or bill count is exhausted. Bill on credit anyway?"*. `/validate`
     is the authority.
   - "Exhausted" uses the party-context flags when loaded (limit 0 = none,
     check off = no gate), else the customer-detail figures.
9. **Per line with an item:**
   - qty 0 → "Quantity cannot be zero.";
   - over the order qty without permission → "Line N bills X against an order
     quantity of Y." (refuse, do not clamp);
   - rate 0 on a non-free line (unless free-item tax) → "Rate cannot be zero.";
   - below the minimum → "Rate is below the minimum selling price.";
   - above the MRP (unless MRP is skipped) → "Rate cannot exceed MRP.";
   - negative stock not allowed → "Row N: only X in stock and negative stock is
     not allowed.".

   Each focuses the cell.
10. No item → "Add at least one item."; total ≤ 0 → "Bill amount must be
    greater than zero."
11. Charges: a row without a ledger (§12.4).
12. **Role gates:**
    - FREIGHT when Freight is ticked, freight calc is not manual and Σ
      freightAmt > 0;
    - LOADING / UNLOADING likewise with loadingAmt.

    If the charges grid has no row of that role → *"X of Y is calculated on
    the items, but the charges grid has no X line — the amount would not be
    billed. Add the X charge, or untick X."*

### 17.4 Save, plain route (F5)

```
if amending            → amend(print)                                    §17.8
checks (§17.3)
if autoPost and isNew  → postNewBill(print)                              §17.5
if !autoPost           → validate(payload); refusal → paint + popup, stop; warnings → paint (no popup)
create(payload)        → key = data.sbId, refno = data.sbBillRefno
   autoPost (a loaded draft re-saved) → adopt the draft, then validateThenPost(print)
   else → "Bill X saved successfully.", remove the autosave, reset to a new bill
```

### 17.5 Post

- **Auto-post, new bill (`postNewBill`):**
  1. `validate(payload + overrides)`, painted without a popup.
  2. **`confirmProceed(verb)`**; a refusal offers only Back.
  3. `create`. No id → *"The server saved the bill but returned no id — reopen
     it from the list (F8) to post it."*.
  4. **Adopt the draft**: key, DRAFT, refno, `draftFromAutoPost = true`, remove
     the autosave. A retry then sends `sbId` and updates instead of duplicating.
     The missing id was the duplicate-bill cause.
  5. `postNow` (no second validate).
- **Post from the keyboard (Ctrl+Enter / Ctrl+Shift+Enter / F6, auto-post
  off):** checks → **`create` first, always** (the posted document must match
  the screen) → `validate + overrides` → apply the charge carry proposals →
  `confirmProceed` → `postNow`. A create with no id: *"The draft was saved but
  the server did not return its id, so there is nothing to post against.
  Reopen the bill from the list (F8) and post it there."*
- **`postNow`:** POST `/bills/post {key, overrides?, printAfter?,
  adjustments?}`. **Send `adjustments[]` whenever the payload has them**
  (§14.4).
  - OK → clear the strip, remove the autosave, say "Bill X posted." / "Bill X
    posted — voucher V." (`posting.voucherRefno`), print (§26), reset.
  - Error → paint the strip. If `draftFromAutoPost`, discard the draft (§17.6)
    and stop. Otherwise show the generic error only if the strip stayed empty.
- `/post` is idempotent on an already-POSTED bill. The response is the full
  `/get` shape. `posting.irn.status` is PENDING right after a post, because
  e-invoice and e-way run async after commit.

### 17.6 A failed auto-post leaves nothing behind (user, 2026-09-24)

POST `/bills/delete` with the key.

- OK → the screen becomes an unsaved bill again: *"The bill could NOT be
  posted:\n\n<reason>\n\nNothing was saved — correct it and press Save
  again."*
- The delete fails too → *"Bill X is saved as a DRAFT but was NOT
  posted:\n\n<reason>\n\nCorrect it and press Save again — it posts this same
  bill, it does not create another."*

Show **one** dialog, not the strip popup plus this one.

### 17.7 Validate (Ctrl+F7), the operator's dry run

Checks → `validate(payload + overrides)` (so a second dry run reflects what is
already ticked) → paint **with** the popup → apply the carry proposals. No
notes → "Nothing to report — this bill is ready to post." **A 422 is the
answer. Paint it and stop.**

### 17.8 Amend (Edit on a posted bill)

1. Edit (F2) on a POSTED bill → `beginAmend`:
   - not posted → "Only a posted bill is edited this way.";
   - not amendable → the reason;
   - confirm *"Edit bill X?\n\nIt keeps its number and date. When you save,
     you will be asked what you changed."*;
   - then `amending = true`, unlock, clear the strip.
2. Save while amending: checks → AskText *"Bill X keeps its number and date.
   What did you change?"* (required, placeholder "e.g. quantity of line 2
   corrected") → POST `/bills/amend {whole payload, sbId, baseRevision,
   editRemark, overrides?, printAfter?, adjustments?}`.
   - OK → "Bill X updated.", reset.
   - Error → strip. `SALES_REVISION_STALE` means someone else amended; reload.
3. The server re-runs the guards, so a 422 comes back as "Bill cannot be
   amended". Qt does not validate before `/amend`. **The React port validates
   first and runs `confirmProceed`**, the same as a post, because WARNs become
   refusals on `/amend` unless overridden (§16.3).
4. Server preconditions:
   - POSTED;
   - `rights.amend`;
   - `sales.allow_posted_amend` (else 409 `SALES_AMEND_OFF`);
   - `baseRevision === sbRevisionNo`;
   - no live IRN/EWB;
   - no return, no allocation, cheques still HELD, day open.

   The voucher is restated **in place** under the same number, and the
   revision goes up by 1.
5. **Open server defect N45:** amend returns a 500 on a bill that **earned
   loyalty points**. Show the server's message; there is no client workaround.

### 17.9 Edit on a draft; Cancel; Delete

- **Edit (F2) on a read-only draft:**
  - no edit right → "You don't have permission to edit bills.";
  - CANCELLED → "A cancelled bill can't be edited — raise a new one.";
  - POSTED → amend (§17.8);
  - else confirm "Switch this bill to edit mode?".
- **Cancel bill** (POSTED only):
  - a draft → *"Only a POSTED bill is cancelled. A draft is deleted (F3) — it
    has no number, no voucher and no stock behind it."*;
  - AskText *"Why is bill X being cancelled?\n\nThe reason is stored with the
    cancellation and cannot be edited afterwards."* (required; presets "Keyed
    wrong", "Wrong customer", "Duplicate bill", "Customer cancelled the
    purchase") → POST `/bills/cancel {key, reason}` → *"Bill cancelled. The
    stock, the ledger and the register have been reversed — the bill keeps its
    number and stays on the list."*
  - The response is **not** the `/get` shape (`reversalVoucherRefno,
    cancelledOn`). Reload with `/get`.
  - The server cancels a live IRN/EWB **at the portal first**, synchronously,
    and refuses past the window.
  - **Open defect D3:** the cancel's mirror voucher takes the next sale-bill
    number and leaves gaps in the GST series (§27).
- **Delete (F3, DRAFT only):**
  - no id → "Nothing to delete — this is a new bill. Open a saved one from the
    list (F8) to delete it.";
  - no right → "You don't have permission to delete bills.";
  - confirm "Delete this bill? This cannot be undone." → POST `/bills/delete`
    keys (a body, not a query DELETE) → "Bill deleted.", reset.

### 17.10 Copy as new (Alt+Y)

- A new bill → *"Nothing to copy — this is already a new bill. Open a saved one
  from the list (F8) to copy it."*.
- Confirm *"Start a new bill from this one? The bill you copied from is not
  changed."*.
- **Keeps:** the lines and the doc type (a bill of supply copies as a bill of
  supply).
- **Clears:**
  - id, number, status (→ DRAFT), audit; date = today;
  - tenders and roll-ups;
  - **adjustments** and the held-by-document set ("carrying them would set the
    same money off twice");
  - **the whole source trail**: header, and per line src ids, type, year, lot
    and bucket. One source would otherwise look converted twice, and the
    customer lock would stay on;
  - charge ids.
- Re-read party-context, unlock.

### 17.11 Clear (F7) in every state = start a new bill

- **Amending:** confirm *"Discard your changes to bill X and start a new
  bill?\n\nBill X stays as it was."*.
- **A loaded read-only bill:** no prompt.
- **Otherwise:** "Clear all entered data on this screen?".

Then remove the autosave and reset. The reset:

- document state: status NEW, refno, revision, `amending = false`, rights,
  locks and strip cleared;
- adjustments cleared silently;
- customer and bill-to cleared; the beat filter reset; **POS = the company's
  state**;
- party facts cleared; date = app date; term Cash; remarks cleared;
- crew kept unless `clear_delivery_on_clear` (§7.9);
- ticks off, grids emptied, freight bands cleared, a fresh policy snapshot;
- charge inter-state flag re-set, **then auto-charges seeded**;
- walk-in seeded if configured;
- focus on the first line.

"Discard changes (reload)" is gone.

### 17.12 Close

With unsaved lines in entry mode: "This bill hasn't been saved. Close anyway?".
A confirmed close removes the autosave, because it is an intentional discard.

---

## 18. The save payload (`/create`, `/validate`, `/amend`)

Built from draft state with an explicit whitelist (§4.4). Empty strings go out
as `null` for uuid, date and optional text columns.

### 18.1 Header

**Identity and scope**

- `sbId` (only once saved), `sbCompanyId`, `sbBranchId`, `sbAccYear`,
  `sbSessionId`, `sbDeviceType`, `sbDeviceId`.
- `sbCounterId` is **not** sent.

**Document**

- `sbBillMode`, `sbDocType` (default `TAX_INVOICE`), `sbBillType`
  (`CASH|CREDIT`, from the term), `sbCategoryId: null`, `sbPriceLevel` (level
  id).
- `sbBillRefno` and `sbBillSlno`: the server assigns both and ignores what it
  is sent.
- `sbUsrRefno`, `sbUsrRefdate`.
- `sbBillDate` (`YYYY-MM-DD`), `sbBillDatetime` (bill date + current time, ISO).
- `sbDueDays` and `sbDueDate` on Credit only, else null.

**Source trail**

- `sbSrcDocType, sbSrcDocId, sbSrcDocYear` (**the source's own year**),
  `sbSrcDocRefno, sbSrcDocDate`.
- The server ignores the reference silently unless both the id and the year
  are sent.

**Party**

- `sbCustId`, `sbLoyaltyMemberId`, `sbCustName` (required, non-empty),
  `sbCustAddr, sbCustPlace, sbCustPin, sbCustPhone, sbCustGstin,
  sbCustGstType, sbCustPan, sbForm60Ref`.
- `sbCustStcd` (the registered state), and `sbPosStcd` + `sbStateName` (place
  of supply). These are two separate facts.
- `sbPosStcd` is a foreign key to `fixed.state_codes`; an unknown code is a 400.

**Transport, flat** (§20.3)

- `sbShipAddrId, sbShipName, sbShipAddr, sbShipPlace, sbShipPin, sbShipPhone,
  sbShipStcd, sbShipGstin`.
- `sbDispatchGodownId, sbDispatchBranchId`.
- `sbTransportMode` (`ROAD|RAIL|AIR|SHIP`, upper-cased by the server),
  `sbTransporterId, sbTransporterName, sbTransporterGstin`.
- `sbLrNo, sbLrDate, sbDistanceKm` (an integer ≥ 0).

**Flags**

- `sbHasLoad, sbHasUnload, sbHasFreight, sbHasPromo, sbHasLoyalty`.
- `sbHasComm` = agent commission > 0.

**People**

- `sbUserId`.
- **`sbSalesmanId`, `sbLoadmanId`, `sbPackedId` as arrays.**
- `sbAgentId, sbAgentCommPerc, sbAgentCommAmt, sbDriverId, sbSupervisorId`.
- `sbVehicleId`: set only from the vehicle picker; cleared when the number is
  typed by hand ("no master row behind it").
- `sbVehicleNo`.

**Totals** (the client computes them; the server stores them as sent and
checks the arithmetic at post)

- `sbTotItems` = **the number of item lines**. Qt sends the row count − 1,
  which counts blank rows too. Send the real line count.
- `sbTotWeight`, `sbTotBags`.
- `sbGrossAmt, sbItemDisc, sbSplDisc, sbSchDisc, sbBillSchDisc`,
  `sbAddlDisc1/2: 0`.
- `sbCashDisc: 0` (a cash discount is a before-tax DEDUCT charge row).
- `sbTaxableAmt` (after charges), `sbCgstAmt, sbSgstAmt, sbIgstAmt, sbCessAmt,
  sbTaxAmt`.
- `sbFreightAmt` = Σ freightAmt when Freight is ticked; `sbLoadAmt` likewise.
- `sbUnloadAmt` = **Σ loadingAmt** when Unload is ticked (§10.3, flagged).
- `sbOtherAmt1/2: 0`, `sbRoundOff`, `sbBillAmt`.
- `sbTotalCost, sbMarginAmt`, `sbMarginAmtWot: null`, `sbMarginPerc,
  sbMrpSavings, sbMrpSavingsPerc`.

**Settlement** (§15.9)

- `sbPayMode, sbCreditAmt, sbSurchargeAmt, sbTenderAmt, sbRefundAmt`.
- `sbAdvanceAmt, sbNoteAdjAmt`.
- `sbPaidAmt, sbBalanceAmt, sbPayStatus`.

**Text**

- `sbPaymentTerms, sbDeliveryTerms, sbTermsConditions, sbRemarks`.

**Policy snapshot**

- `sbFreightCalcType, sbLoadingCalcType, sbRoundOffStep`.
- `sbDiscAlterBase`: send a bool or omit it. **`null` is a 400.**

**Status and audit**

- `sbStatus`: the server ignores it (a save is always DRAFT). Omit it. Qt
  sends the status label's text, or "POSTED" under auto-post; do not port
  that.
- `sbCreatedBy` on a new bill, `sbModifiedBy` on a loaded one (the user's
  name).

Arrays: `items[]`, `charges[]`, `tenders[]`, and `adjustments[]` **only when
authoritative** (§14.4). An omitted array is left untouched on an update. A
present array fully reconciles: rows with an id are updated, rows without one
are created, and missing rows are soft-deleted.

### 18.2 Items (`sbi*`), one per line

**Scope and position**

- `sbiCompanyId, sbiBranchId, sbiAccYear`.
- `sbiLineNo` (1-based, unique within the payload, else 409); `sbiSplitNo: 1`.

**Source trail**

- `sbiSrcDocType, sbiSrcDocId, sbiSrcDocRefno, sbiSrcDocYear`.
- **`sbiSrcItemId`** (the source line) and **`sbiSrcDocLineNo`**.
- `sbiSrcItemQty` (= the order qty), `sbiSrcFreeQty: null`, `sbiLotId`.

**Traps**

- **`sbiBucket`: never null.** Send the value upper-cased, or `'SALEABLE'`. An
  explicit null beats the NOT NULL default and gives a bare 500.
- **`sbiSize` / `sbiSizeUom`: null when blank** (`ck_sbi_size` rejects `""`),
  else the text plus `'CFT'`.

**Item and stock**

- `sbiItemId`, `sbiItemUnitId` (an **iuc id**, not a unit id),
  `sbiGodownId` (**required** on the DTO).
- `sbiToBaseFactor, sbiHsnCode, sbiPriceLevel, sbiEanCode`.
- `sbiBatchNo, sbiBatchDate, sbiExpiryDate, sbiSerialNo`.

**Flags**

- `sbiIsTaxIncl, sbiIsPromo, sbiIsFree`, `sbiFreeType: null`, `sbiIsService,
  sbiHasFreight`.

**Quantities**

- `sbiCaseQty, sbiBillQty, sbiLengthQty`.
- **`sbiNetQty`**: it is what draws an order down. Default 0 delivers
  nothing, so send both qtys.
- `sbiWeightQty, sbiAvailableStock`.

**Rates and discounts**

- `sbiRate, sbiRatePreTax, sbiRateDiff`.
- The % / qty / amount of each tier: `sbiItemDisc*`, `sbiSplDisc*`,
  `sbiSchDisc*`, `sbiBillSchPerc` / `sbiBillSchAmt` (`sbiBillSchQty: null`),
  `sbiCashDisc*`.
- `sbiAddlDisc1/2*: null`.

**Amounts and tax**

- `sbiGrossAmt, sbiNetGross, sbiChrgBeforeTax, sbiChrgAfterTax,
  sbiTaxableAmt`.
- `sbiTaxPerc/Amt`, `sbiCgst*`, `sbiSgst*`, `sbiIgst*`, `sbiCessPerc,
  sbiCessPerUnit, sbiCessAmt`; `sbiAcess*: null`.
- `sbiFreightQty/Amt, sbiLoadQty/Amt`; `sbiUnloadQty/Amt: null`.
- `sbiRoundOff: null` (rounding is document-level only).
- `sbiNetAmt` (the line total).

**Cost and outcome**

- `sbiCostPrice, sbiCostPreTax, sbiMaxPrice, sbiMinPrice, sbiActPrice`.
- `sbiSoldPrice, sbiSoldPreTax, sbiItemProfit, sbiProfitPreTax`.
- `sbiMrpSavings, sbiMrpSavingsPerc`.

**People and scheme**

- `sbiSalesmanId` (single, per line), `sbiSchemeId, sbiSchemeName`.
- `sbiRemarks, sbiBatchConfig`.
- `sbiCreatedBy | sbiModifiedBy`.

**Not sent**

- `sbiTaxId`: the server stamps it from the item master on every save and
  resolves the output ledgers from it.
- `sbiStockId`: optional on the DTO.

---

## 19. Loading a bill (`/bills/get` → draft)

1. Mark the screen as opened for a document **first**, so no walk-in seed
   runs.
2. GET `/bills/get` with all four keys. An empty response → "That bill could
   not be found."
3. Map. The header is **flat** (no `header` object; verified live 2026-09-22).
4. Fire party-context for the customer. It gives the party's standing **as of
   today**, which is what matters when deciding to amend or cancel.
5. Apply rights and locks, the transport locks and the GST badges.
6. Read-only = `!locks.editable.document || !openForEdit`.

**Header map.** Decimals are strings, and date-only fields are full ISO:

- **Identity:** `sbId`, `sbDocType`, `sbBillMode`. Company, branch, year and
  device override the session's.
- **Policy snapshot:** `sbFreightCalcType, sbLoadingCalcType, sbDiscAlterBase,
  sbRoundOffStep`. The bill keeps the policy it was billed with; the session
  value is only the fallback.
- **Customer snapshot:** `sbCust*`, `sbCustPan, sbForm60Ref`. `sbCustStcd`
  falls back to `sbPosStcd` for older bills. Local/inter comes **from the
  stored taxes** (§7.8).
- **Transport:** the flat `sbShip* / sbDispatch* / sbTransport* / sbLr* /
  sbDistanceKm` (also nested under `transport`).
- **Document:** `sbBillRefno` → the title "SALES BILL — X". `sbBillDate`,
  `sbDueDays/Date`, `sbUsrRefno/refdate`. `sbBillType` → the term.
- **Crew:** element 1 of the three arrays; the other crew fields and the
  vehicle as they are. **No staff display names come back** (§27), so the
  dropdowns must resolve labels by id (§6.4).
- **Source trail:** `sbSrc*`; `sources[]` for the chip.
- **Flags and text:** the ticks, `sbPriceLevel`, the terms text and remarks.
- **Status:** `sbStatus`, `sbRevisionNo` (→ `baseRevision`), and the audit
  line "created … · modified …".

**Items** (skip `sbiIsDeleted`). Every stored figure is painted as it was
billed (a `stored` line, §5.2). `sbiSize` is painted and never recomputed.
The GET-only display fields are `sbiItemName, sbiUnitName, sbiGodownName,
sbiDecimalCount, sbiGroupId/BrandId/SectionId/CategoryId`, and
**`sbiAllowNegativeStock`** (the effective flag). Use that flag. Qt hard-codes
`"Y"` on the load path; do not port that.

**Charges:** populate from `charges[]` (numbers). `cdLedgerName` is read-only.

**Totals:** the stored header figures. **Counter paid = `sbPaidAmt −
sbAdvanceAmt − sbNoteAdjAmt`** (§15.9). The profit stats are recomputed from
the lines.

**Tenders:** `tenders[]` → tender lines (`tdReceivedAmt` → amount,
`tdChangeAmt` → refund, `tempCredit`, `cheque`). The type name is not stored;
match to the master by `tdTenderId`.

> **Suspected defect (read in the source, not yet verified live):** `/get`
> does not filter **voided** tender rows and carries no `isVoided` flag. After
> a re-tender, `/get` shows voided and live rows the same way, and an amend
> would count them again. Only `/tender-context` exposes `isVoided`. **Verify
> on the box before relying either way** (§27 T-VOID). Until then, a
> re-tendered posted bill should open read-only, with its tenders shown from
> `/tender-context`.

**Adjustments:** §14.4. `heldByDocument` is set **only for a POSTED bill**.

**Posting block:** `{voucherId, voucherRefno, postedOn, registerId, cogsAmt,
loyaltyEarned, loyaltyRedeemed, irn{status, number, ackNo, ackOn, message},
ewb{status, number, generatedOn, validUpto, message, vehicleNo}}` → badges
(§21).

---

## 20. Shipping and transport (the shared modal)

### 20.1 Why a modal

The shipping detail is a **popup** in the 3.0 ShippingDetail layout: Transport
on top, then Dispatch From beside Ship To, then Clear (F7), Save transport (F5)
and Close (Esc). It is not an inline band. The bill's **Terms** group
(Payment / Delivery / Other terms) lives **inside** this popup, in its extra
slot. Crew and vehicle are **header** fields, because a home delivery has a
van and no consignment. The vehicle is mirrored in the popup.

### 20.2 Fields

- **Transport:**
  - Vehicle: dropdown 56 plus a typed number (≤ 20, upper-cased). A typed
    number clears the vehicle id.
  - Transporter: dropdown 55. It fills "Name as declared" only when that is
    blank.
  - Mode: — / Road / Rail / Air / Ship, codes `'' ROAD RAIL AIR SHIP`.
  - Distance (km, 0–99999). Transporter ID (GSTIN / TRANSIN, ≤ 15).
  - LR / Doc No (≤ 50) + date.
- **Dispatch From** (read-only, from the branch):
  - GET `/branch-masters/get?brId`: address = `brAddr1..3` joined; name =
    `brMailingName || brName`; state "stcd · name"; the PIN may arrive as a
    number or a string;
  - Godown (dropdown 26) is the only input: *"From the branch master — change
    the godown, not the address."*;
  - the branch id is cleared when no godown is set, because an empty dialog
    must stay empty.
- **Ship To:**
  - Site combo: item 0 = "Bill-to address (as printed)", then party-context
    `shipTo[]` ("name · place (default)");
  - "+" opens the customer's ship-to list (a separate screen). On close it
    re-reads **only** `shipTo[]`;
  - Name*, Address*, Place*, PIN* (6 digits), State* (dropdown 21), GSTIN,
    Phone.
- **"Empty Ship To means 'as the bill-to'."** The bill-to shows as grey
  placeholders and is **never written**, so opening the popup does not stamp a
  ship-to snapshot on every document.
- Picking a site fills the fields. Its distance is copied only when it is > 0
  and the distance box is empty. A loaded document keeps what it was saved
  with, and the default site is applied only while the dialog is empty.
- State names come from the fixed GST state table (codes 01–38, 97, 99),
  because dropdown 21 shows only the text it is handed.

### 20.3 Two save modes

| Bill state | Mode | How it saves |
|---|---|---|
| DRAFT / new | with the document | flat `sbShip*`… on `/create` |
| POSTED, `locks.editable.transportBand` | its own verb | **PUT `/bills/transport {key, transport: {direction:'OUTWARD', from, to, mode, transporterId, transporterName, transporterGstin, lrNo, lrDate, distanceKm}}`**. The document is not re-sent and its revision does not move. "Transport details saved on bill X." |
| IRN or EWB generated (declared) | locked | *"Declared — locked. A vehicle change is a Part-B update on the e-way bill."* (+ "Vehicle on the e-way bill: X" from `posting.ewb.vehicleNo`) |

- An `End` is `{godownId, branchId, addrId, name, addr, place, pin, phone,
  stcd, gstin}`.
- Mood strip, in priority order:
  1. the required message from validate (red);
  2. declared (blue);
  3. read-only (grey);
  4. posted, own verb (amber): *"The document is posted — these details save
     on their own (Save transport, F5). Locked once the e-way bill is
     generated."*.
- **Server traps:**
  - `/create` **cannot clear** a band: all-blank = no write. It does a full
    replace when any field is set, and it always nulls `remarks`. To clear a
    posted band use the PUT.
  - A `GST_DECLARED_LOCKED` answer on the PUT means an IRN or EWB landed while
    the popup was open. **Reload the bill.** Qt describes this in a comment
    but does not do it.
- Clear (F7) empties the form but keeps the vehicle ("the header's, not ours
  to clear").
- A **summary line** sits beside the Shipping button: "to NAME, PLACE ·
  transporter · VEHICLE · LR n · N km · driver X", or "Not filled — opens
  itself when the e-way rule needs it". Qt meant to show this, but its label
  is never created. Build it.

---

## 21. GST badges and actions

**Badges** (POSTED or CANCELLED only):

- IRN · EWB · "COGS X · voucher V".
- Colours: GENERATED green · FAILED/REJECTED red · PENDING amber ·
  CANCELLED/EXPIRED grey · empty "NA".
- IRN detail: "ack N · dd/MM HH:mm", plus the message when not generated.
- EWB detail: number and "valid to …".

**e-Inv** (enabled when posted and no IRN yet):

- Label: "Retry e-Inv" if failed, "Fetch e-Inv" if pending, else "e-Inv".
- With no transport and no vehicle, ask *"No transport details on this bill.
  Generate the IRN without an e-way bill?"*.
- POST `/gst/einvoice/generate {key, sync:true}` → "IRN generated: X" or
  "e-Invoice <status>: <message>", then **reload** read-only.
- When an IRN exists, the tooltip says: "IRN already generated. Cancel it from
  the bill's Cancel (inside the window) or the register (Shift+F2)."

**e-Way** (enabled when posted):

- Label: "Part-B" if generated, "Retry e-Way" if failed, else "e-Way".
- With an empty band: mark the shipping dialog required ("An e-way bill needs
  the transport details — fill them and save first.") and open it.
- POST `/gst/ewaybill/generate {key, sync:true}` → "E-way bill X generated,
  valid to Y.", then reload.
- An error goes **to the warning strip first** (`GST_EWB_TRANSPORT_MISSING`
  with `fields[]`). The generic error shows only if the strip stayed empty.

**Part-B:**

- AskText vehicle number (required, upper-cased, default = the header
  vehicle).
- AskText remark (optional; presets "Breakdown", "Transhipment").
- POST `/gst/ewaybill/vehicle {gdrId: posting.registerId, vehicleNo, mode,
  reasonCode: 1, remark, fromPlace, fromState: company state}` → "Vehicle
  updated to X on e-way bill Y.", then reload.
- It is a portal operation recorded on `gdw_`, never an edit of the band.

A B2C bill answers NA and stays NA.

**The GST routes are not on the test box** (404). The screen calls them and
shows the server's answer. **Shift+F2 / Shift+F3** open the e-invoice /
e-way registers (menus 152 / 153).

---

## 22. Re-tender (Ctrl+F6): change how a posted bill was paid

It is its own verb, not an amend. It works **even with an IRN**, because the
tender is not on the e-invoice, GSTR-1 or the e-way bill. It **opens the
re-tender form only**. It never loads or clears the bill on screen, and it
reloads that bill only if the re-tendered bill is the one showing.

1. **Rights:** `rights.retender` from `/get` (it is its own flag, not amend's).
   With a bill loaded and the key false: "You don't have the re-tender right on
   this menu." `/tender-context` does not check rights; the post does (403
   `SALES_RETENDER_RIGHT`).
2. **Picker:** grid **115**, "Recent bills — re-tender". It lists this
   device's bills from today, newest first (`idevice_id` = the session device;
   the default is today). The hint says "Type a bill number for anything
   older." The pick label is "Re-tender".
3. **GET `/bills/tender-context`** (sb keys) returns:

   ```
   {sbBillRefno, sbBillDate, sbCustName, sbBillAmt, sbPaidAmt, sbBalanceAmt, sbStatus,
    canRetender, reason: null|'bill cancelled'|'day closed',
    tenders: [{tdId, tenderName, tdTenderId, tdTenderTypeId, tdAmount, tdIsPdc,
               pdcMoved, isLoyalty, isTempCredit, isVoided, replacesId}]}
   ```

   `!canRetender` → "day closed" gives *"The day is closed — the drawer was
   counted with this figure. That is a day-book correction, not a
   re-tender."*; anything else gives "This bill cannot be re-tendered:
   <reason>".
4. **Dialog.** The head reads "Bill X · date · customer · total · paid".
   - **"As it was tendered — tick what did NOT happen":** columns Void ✓ ·
     Tender · Amount · Reason · Note. Voided rows are skipped.
   - **Void reasons:** `UPI_FAILED` "UPI failed", `CARD_DECLINED`,
     `CHEQUE_REFUSED`, `KEYED_WRONG`, `CUSTOMER_CHANGED`, **`OTHER`** (the
     server accepts it; Qt leaves it out).
   - **Row notes:**
     - `pdcMoved` → the tick is disabled, with the red note "cheque already
       deposited / cleared — a bounce, not a re-tender";
     - `isLoyalty` → "voiding restores the points to the member";
     - `isTempCredit` → "voiding clears the temp-credit entry behind the
       balance";
     - `tdIsPdc` → "post-dated cheque, not yet deposited".
   - **"What really happened":** Tender · Amount · Reference, over the
     offerable masters **excluding** ADJUST, LOYALTY, CREDIT and TEMP_CR.
   - **Footer:** "Voided X · New Y", green when they match, else red.
   - **Remark** (required, ≤ 250, placeholder "UPI failed at counter, paid
     cash").
   - **OK** only when ≥ 1 void is ticked, |voided − new| < 0.005, and
     `canRetender`.
5. **POST `/bills/retender {key, voids:[{tdId, reason}], tenders:[…], remark}`.**
   - **New rows are numbered after ALL existing rows, including voided ones:**
     `tdRowNo = existing count + 1…`. Numbering from 1 collides and gives a
     500.
   - **Build each new row with the full §15.10 builder.** Qt sends bare rows:
     surcharge 0, no bank, no cheque date or details, `tdIsPdc` always false,
     no device or user. The React port offers the instrument panel for a
     CHEQUE / CARD replacement (G2).
   - OK → "Bill X re-tendered.", and reload the bill if it is on screen.
   - Refusals: `SALES_RETENDER_AMOUNT_MISMATCH` (422), `SALES_DAY_CLOSED`
     (409), `SALES_RETENDER_PDC_MOVED` (409), `SALES_RETENDER_RIGHT` (403).
6. **Server effect:**
   - the voided rows stay (`td_is_voided`, the reason);
   - the new rows carry `td_replaces_id`;
   - on a POSTED bill a **contra voucher** (type 22 `TndC`, refno
     `<bill>/RT<n>`, dated today) goes DR the new tender ledgers and CR the
     voided ones. The party balance never moves;
   - the revision is **not** bumped;
   - tender, paid, balance and pay status are recomputed from the live rows;
   - voiding LOYALTY restores the points; voiding TEMP_CR cancels the temp
     credit.

There is still **no button** for it; Ctrl+F6 only (the user has not picked
one).

---

## 23. Hold and autosave

### 23.1 Hold (F9) and Pick Held (F10): move to the server

The Qt bill keeps holds in a **local file per device** (`holds_<deviceId>.json`),
with the comment "to be swapped for a /holds endpoint when one exists". That
endpoint now exists, and the quotation and the DC already use it. **The React
bill uses `/txn-holds`** (decision to confirm, §28 Q2):

- **Hold:**
  - refused in browse mode ("A read-only bill can't be held."), on an empty
    bill ("Nothing to hold — the bill is empty.") and on a saved bill ("A
    saved bill can't be held — use Copy or Edit instead.");
  - confirm "Suspend this bill and start the next one?";
  - POST `/txn-holds/create {txhKind:'HOLD', txhStatus:'HELD',
    txhSrcModule:'SALES', txhDocType:'SALE_BILL', company/branch/year,
    txhDeviceId (registered), txhPartyId + txhPartyType:'CUSTOMER', party name
    and mobile, item count, qty, net amount, txhPayload: <the snapshot>,
    txhPayloadVersion: 1}`. Never send a hold number; the server counts per
    device. Optional text is omitted, not `''`;
  - then reset.
- **Pick** (grid 96, `idoc_type`, 7 days back):
  1. An unsaved bill is confirmed away first.
  2. POST `/txn-holds/{id}/resume` (the lease; `X-Device-Id` header).
  3. GET `/txn-holds/get`.
  4. **Restore, THEN mark the hold active.** A restore resets the screen, and a
     reset releases any active lease.

  Delete in the picker = DELETE `/txn-holds/delete` after a confirm. A row
  leased by another terminal is not restorable.
- **After a successful save or post:** POST `/txn-holds/{id}/convert`
  (converted doc id, year and refno). The bill post does not convert a hold by
  itself. Abandoning the bill → `/release`.
- **The snapshot** = the save payload **plus display fields per line**:
  `sbiItemName, sbiUnitName, sbiGodownName, sbiGroupId, sbiBrandId,
  sbiSectionId, sbiCategoryId, sbiLoyaltyPv`. It also carries the adjustments'
  snapshot form (§14.5). Without the display fields a restored bill has blank
  names, and promotions (keyed on group) mis-match (user, 2026-09-25).
- **The restore** runs the load path on the snapshot:
  - it strips the identity (no id, NEW);
  - it reconciles the adjustments (§14.5);
  - it **re-reads party-context**. Qt forgets to, and its restore can race the
    walk-in seed.

### 23.2 Autosave (power-failure recovery)

- **When:** `sales.auto_save_temp_bill` on, a NEW bill with ≥ 1 line,
  debounced 1 s.
- **Where:** **IndexedDB**, keyed by device + user + document type. It uses
  the same snapshot shape as a hold. (The server's `txhKind: 'AUTOSAVE'`
  upsert is the alternative; IndexedDB survives a dead network, which is the
  case autosave exists for.)
- **Offered on open:** *"An unsaved bill for X was found (power failure or
  crash). Restore it?"*. No → delete it.
- **Deleted on:** save, post, clear, hold and a confirmed close.
- Every read and write is in `try/catch`. Billing must never stall on it.

---

## 24. The lists the bill owns

The lists run on the shared `TxnMainView` pattern: descriptors, not
subclasses. The key comes **from the row**, never the session. Badges are
derived from text. Filters are data. From/To are always sent. Keys: F1 add,
F2 edit, F3 delete, F5 refresh, F6 print, F7 print in format, Enter edit/pick,
Ctrl+Enter view. **Every optional grid param is always sent (`''` = unset);
never cast a token.**

| List | Menu / grid | Rows | Rules |
|---|---|---|---|
| **Sale bills** | 12 / grid **86** (a TODO dedicated id: it shares the quotation's layout, and its SQL still casts tokens; re-register in the runner's quoted-token style) | `sb_id, sb_company_id, sb_branch_id, sb_acc_year, sb_bill_refno, sb_status`; badges `sb_status`, `sb_pay_status`; 30 days; search "party name, bill no, amount or mobile" | deleted → read-only · CANCELLED → no edit/delete ("raise a new bill") · `sb_return_status` set → no edit/delete ("A sales return exists against this bill.") · POSTED → no delete (edit → Amend). Delete = POST `/bills/delete` with the four keys |
| **Delivery** | 226 / grid **113** | `sb_delivery_status` badge; filter `idelivery_status` ('' = pending set, PENDING…ALL) | PUT `/bills/delivery-status {key, event, remarks?}`: Verify F4 (only with `sales.require_verification_before_dispatch`, from NA/PENDING) → Packed F5 → Dispatched F6 → Delivered F7; the server enforces the order (`SALES_DELIVERY_ORDER`). Only meaningful with `sales.delivery_status_tracking`; POS bills are NA |
| **Temp credits** | 257 / grid **114** | `atc_*`; filters `istatus` ('' = Open + Partial … ALL) and `ioverdue_only`; 365 days; search "mobile, name or bill no" | Open = the **bill** behind it (`atc_src_doc_id`), read-only. **Receive F5** (balance > 0): copy the mobile to the clipboard, say "Opening the receipt. Bill X · mobile M (copied) — pick the walk-in ledger and tick the bill.", open menu 99. **Follow-up F6** (balance > 0): "Promised for" date + "Remark *" → PUT `/temp-credits/follow-up {atcId, atcAccYear, remarks, promiseDate}`, where promiseDate is **three-way: omitted = keep, null = clear, date = set** |
| **Re-tender picker** | grid **115** | §22 | pick only |

- Grid ids: the Qt handover gives 131 / 133 / 134. The box assigned **114 /
  115 / 113**. The code is right.
- The bill list has **no** delivery-status, IRN or mode columns and no status
  filter. Add them when re-registering the grid (§28 Q8).

---

## 25. Settings the screen reads

All are `sales.*` unless named. **Read a TEXT/UUID override of `null` as
`''`**; only bools and numbers fall back to their compiled defaults.

| Setting | Effect |
|---|---|
| `tender_type` (none / cash_bills / all_bills) | tender route (§15.1) |
| `tender_print_only` | disable the dialog's Save |
| `auto_post` | Save = check → create → post (§17.5) |
| `allow_excess_tender` | client save check 7; tender check "cannot exceed"; server `SALES_TENDER_MIN_MAX` |
| `allow_payment_term_change` (BRANCH) | term combo lock |
| `pop_default_customer`, `default_customer_id` | walk-in seed and bill-to rule |
| `allow_customer_change_on_import` | import lock |
| `allow_duplicate_item` | merge vs confirm |
| `allow_bill_over_order_qty` | order cap |
| `salesman_mandatory` | client + server |
| `multi_order_bill` | append a second order |
| `weight_barcode` (JSON) | §9.2 |
| `auto_pop_qty`, `edit_price` (session) | qty 1 on scan; rate editable |
| `auto_save_temp_bill` | autosave |
| `clear_delivery_on_clear` | Clear keeps the crew |
| `temp_credit_max_days` / `_max_amount` / **`_block_open` (ENUM)** | §15.7 |
| `default price level`, `round_off_step`, freight / loading calc types, `discount_alter_base_rate` | the **policy snapshot**, taken once per document and replayed from a loaded bill (never re-read) |
| `require_verification_before_dispatch`, `delivery_status_tracking` | delivery list |
| `accounts.reconcile_on_post` | any post may be refused `ACC_PARTY_OUT_OF_BALANCE` / `ACC_CHEQUES_OUT_OF_BALANCE` (trial mode). Show it like any refusal; no client logic |

Server-only settings the screen never reads, but whose refusals it shows:
`backdate_mode`, `credit_limit_mode` (also echoed as `party.credit.mode`),
`rate_below_min_mode`, `allow_posted_amend`, `max_line_disc_perc`,
`max_bill_disc_perc`, `post_scheme_disc_separately`, `loyalty_*`,
`pos_series_per_device`. Law thresholds (269ST, PAN, TCS) are **per company**
through `fn_statutory_limit`, never a constant.

---

## 26. Printing

**Nothing prints today.** The client stubs F11, the list's F6/F7 and
print-after-save with "Printing isn't available yet". The server **accepts
`printAfter` on `/post` and `/amend` and throws it away.**

What changed since `plan-react-printing.md` (2026-08-27), which says nothing
renders: a **server renderer exists**.

- POST `/print-render/print` resolves the assignment ladder, renders and logs
  one `print_log` row per copy. It returns bytes: a PDF, or ESC/POS / ESC/P as
  an octet-stream, with `X-Print-*` headers.
- POST `/print-render/preview` renders a named revision and does not log it.
- GET `/print-render/providers`.
- Body: `purposeId, docId`, with optional `companyId, accYear, srcModule,
  srcDocType, branchId, deviceId, params, outputMode, copies, isReprint,
  inspect, filename`.
- Engines: JSON_BANDS → PDF and ESCPOS_TEXT → ESC/POS.

Still missing for a bill: a `print_purpose` row, a published template and an
assignment for the sale bill.

Rules that bind:

- **No client-side rendering** (no jsPDF, no `window.print()` of a div).
- **Every counter print goes through `/print-render/print`.** `print_log` is
  the anti-theft control: no row means no print was attempted.
- A reprint (F11, list F6) sends `isReprint: true`, which adds the duplicate
  watermark.
- `sb_print_count` is display only.

Until the bill's purpose and template exist, the print verbs are **visible
and disabled**, with the tooltip "Printing arrives with the bill's print
template". Post & Print posts, then says it did not print. It never pretends.

---

## 27. Backend gaps this screen inherits

| id | What | Effect on the screen |
|---|---|---|
| **D3** | A cancel's mirror voucher draws the **next sale-bill number** and leaves gaps in the GST invoice series (214 phantom vouchers on the box) | nothing to do client-side; blocking for GST |
| **D1 (old rows)** | two parties' old bill-balance rows omit their counter tender; with reconcile on, every post for them is refused | shown as a refusal |
| **C4** | the header-vs-rows check is unconfirmed, and **what `sbTenderAmt` means has never been defined** | send the Qt figure (§15.9) until it is |
| **N45** | `/amend` returns a 500 on a bill that earned loyalty | show the message |
| **DEV** | `SALES_DEVICE_UNREGISTERED`: login does not bind a registered device, so the token carries none. `/validate` should refuse it (so no draft is orphaned) | **blocks the web client**: it cannot post until login returns a registered device (§3.3) |
| **GST-404** | `/gst/einvoice/*` and `/gst/ewaybill/*` are not on the box | e-Inv / e-Way / Part-B answer 404 |
| **CHG-TAX** | `/charges/get` returns no numeric tax rate for a charge (`ledGstRate` was dropped, `chgTaxId` has no rate) | after-tax taxable charges price at 0% (§12.1). Verify live, then report |
| **T-VOID** | `/get` does not filter or flag voided tender rows; post/amend may count them | re-tendered bills: see §19. Verify live |
| **E-WAY-DB** | the validate e-way check reads the **stored** band, not the body | a false "transport missing" on an unsaved draft |
| **NO-CUST-CODE** | the no-customer refusal uses the code `SALES_PAN_REQUIRED` | show it by `field` |
| **CREATE-SCOPE** | the `/create` update finds the bill by `sbId` only (company/branch/year ignored) | none; server hardening |
| **BAND-CLEAR** | `/create` cannot clear a transport band and always nulls `remarks` | use the PUT to clear |
| **NAMES** | `/get` has no display names for the crew | dropdowns resolve labels by id (§6.4) |
| **PROMO-KEYS** | customer-detail lacks the customer group and city ids | CUSTOMER_GROUP / CITY promotion rules never match |
| **PRINT** | `printAfter` is dropped; no bill print purpose or template | §26 |
| **SCAN** | `/sales/resolve-scan` and `/sales/resolve-item-scan` (increment-not-append, scan-to-load) not built | §9.2 |
| **SET-38** | the `allow_payment_term_change` definition row is not applied on the box | the default "locked" holds |
| **RT-500** | a re-tender with a duplicate `tdRowNo` is a bare 500 on a full row | the numbering rule (§22) avoids it |
| **NULL-500** | an explicit null on a NOT NULL DEFAULT column is a bare 500, not a 400 | `sbiBucket` rule (§18.2) |
| **ORD-DRAFT** | the whole-order import can post lines from a DRAFT order | none |

**Closed and verified; do not re-raise:** GST 0% from the lookup; the HSN key;
set-off on bill cancel; the credit-note `sbNoteAdjAmt`; amend 500 (non-loyalty);
the second re-tender; order post / advance tender; an overridable WARN also
being a refusal; the credit-note cancel 500; the order header advance figures;
`rights.retender`; the party-context address; `posting.ewb.vehicleNo`; the
menu 257 row; the grid-runner search cache; cheques → register (notes 46);
cheque details (notes 48); reconcile functions (notes 47); D1 for new bills; the
opening sign.

---

## 28. Decisions for the user (the plan assumes the first option)

| # | Question | Assumed |
|---|---|---|
| Q1 | "Put on credit anyway" for a customer not allowed credit (D2): carry the override into the save check and let `/validate` rule, or do not offer it | **do not offer it**; say "not allowed to buy on credit" at the CREDIT row |
| Q2 | Holds: server `/txn-holds` (like the quotation and DC) or local (like Qt) | **server** for holds; IndexedDB for autosave |
| Q3 | Quotation import with no status filter lets a quotation be billed twice | picker shows **open quotations only** + ask the server to refuse a second conversion |
| Q4 | `AllowNegative` hard-coded true on imported and loaded lines | use the item's effective flag (`sbiAllowNegativeStock` / the lookup) everywhere |
| Q5 | Does the price lookup overwrite an imported challan/order line's rate? | **keep the source rate**; the lookup fills everything else |
| Q6 | A walk-in cannot type a GSTIN (the field left the form), so there is no B2B bill for a walk-in | restore GSTIN to the walk-in's bill-to only |
| Q7 | Loyalty: is the points field keyable (the Qt field is unreachable)? | amount only, points shown |
| Q8 | Bill list: dedicated grid with status / pay / delivery / IRN / mode columns + a status filter | yes, re-register |
| Q9 | Freight band match: per-unit weight or line weight | per-unit (as Qt reads the cell); confirm |
| Q10 | Unloading reuses the loading amount; Qtl/Ton charge methods use raw kg | port as-is behind named constants, raise with the user |
| Q11 | Re-tender gets a button, or stays Ctrl+F6 only | Ctrl+F6 only |
| Q12 | TEMP_CR: does it flip the term to Credit (a walk-in probably stays Cash)? | it counts as credit in the roll-ups; it does **not** flip the term |

---

## 29. Tests

- **Vitest, `domain/pricing` extensions, written first:**
  - the paise GST split (4,000 inclusive at 18% → parts sum exactly);
  - discount-first vs standard;
  - billQty-based freight/loading;
  - round-off half-up to the step;
  - the bill invariant.
- **Vitest, `settle.ts`:**
  - surcharge netted from the balance;
  - F1; change to the first change-capable row that can hold it;
  - TEMP_CR counted as credit;
  - release before change: bill 1,000 with an advance of 300 set off, cash
    1,000 typed → 300 set-off released, change 0; cash 1,100 → 300 released,
    change 100 (the bil00702 case);
  - an advance's refund clamped at 0;
  - LOYALTY scheme rate + points (D4).
- **Vitest, `rollups.ts`:**
  - every no-tender and tender combination;
  - the load round-trip: `counterPaid(load(save(x))) === counterPaid(x)`
    with advances and notes.
- **Vitest, `buildSavePayload`:**
  - a whitelist test per DTO: no key outside SaveBillDto, SaveBillItemDto,
    SaveChargeDetailDto and SaveTenderDetailDto;
  - `adjustments` omitted when not authoritative;
  - carry keys only on carried rows;
  - `sbiBucket` never null; `sbiSize` null when blank;
  - salesman arrays;
  - `sbDiscAlterBase` never null.
- **Vitest, `loadMap`:** decimals as strings, ISO dates sliced, stored lines
  not repriced, local/inter derived from the stored taxes.
- **Vitest, `verbState`:** every row of the §17.1 table.
- **Vitest, `WarningStrip.parse`:** all five holders, the 422 envelope,
  ticks reset on a new answer.
- **Component tests (MSW):**
  - a validate refusal blocks the tender from opening;
  - server away → the tender opens;
  - a failed auto-post deletes its draft and shows one dialog;
  - stale party-context and party-balance replies are dropped.
- **The live acceptance list** mirrors `tests/salebill/run-full.sh` (156 / 0 /
  3 on 2026-09-25). The modes are `chequecheck, chequebounce, advamend,
  partial, holdcheck, tendershot, amendmatrix, itemclear, custlimit,
  chargeform, amend <billno>`. Run them against the React screen through the
  same `ZT-` data, with a registered device.

---

## 30. Phases

| Phase | Builds | Done when |
|---|---|---|
| 0 | Close the dropdown gaps (§6.4); the device-id decision (§3.3); verify CHG-TAX and T-VOID on the box | the three gaps are merged; both defects confirmed or dismissed |
| 1 | `domain/pricing` extensions, `sales-shared/engines/*`, `rollups`, `payload`, `loadMap`, `verbState`, WarningStrip parse | the §29 Vitest suites pass |
| 2 | Screen: party band, item grid, lookup / barcode / size / levels, charges, promotions, totals; save draft, validate, load, clear, copy | a draft round-trips `/create` → `/get` byte-equal on the whitelisted keys |
| 3 | Post, auto-post, amend, cancel, delete, the verb bar, rights/locks, the strip + popup + overrides | `amendmatrix`, `custlimit` and `itemclear` pass |
| 4 | Tender dialog (all types, A–E), the adjust panel + F4, roll-ups | `tendershot`, `partial`, `advamend`, `chequecheck` pass |
| 5 | Imports (quotation, order whole + append, challan), the charge carry, shipping + transport PUT, GST badges/actions | `holdcheck` + the import cases pass |
| 6 | Re-tender, holds on `/txn-holds`, autosave, the four lists + follow-up | `chequebounce` + re-tender cases pass |
| later | Printing (§26); POS mode screen; TCS; the deferred quick-strip buttons | the print purpose exists |

---

## 31. What changed from the two old plans

**`plan-react-sale-bill.md` (2026-09-12) was wrong about:**

- The API: cancel is `/bills/cancel`; `/delete` is DRAFT only; `/create`
  cannot touch a posted bill (409 → `/amend`).
- "Validation is 100% client-side": `/validate` is the gate.
- `partyCredit`: replaced by `/bills/party-context`.
- The grid "89 visible": it is 14.
- Holds "on the server already": Qt is still file-backed.
- The set-off "posts DR ADVANCE_RECEIVED": it posts no leg now.
- "No loyalty accrual": the server earns at post.
- "No IRN / no cancel in Qt": both are built.

**It was missing:**

- validate, post, amend, cancel;
- the verb table, rights, locks, the strip and overrides;
- party-context facts;
- the beat filter, the walk-in bill-to rule, the term lock, due days, quick add;
- shipping + transport PUT;
- re-tender, TEMP_CR, cheque details, PAN / Form 60, 269ST;
- the challan import + multi-order append + charge carry;
- weight barcodes;
- bill mode, copy-as-new, Clear-in-every-state;
- GST actions;
- the delivery and temp-credit lists;
- device identity, reconcile refusals.

**`plan-react-sale-tender.md` (2026-08-12) was wrong about:**

- Credit notes as a `RedemptionPanel` with per-note tender lines: they are
  now adjustments, and the ADJUST row is a mirror that is never sent.
- The RRN type needing a reference: it does not.
- Loyalty rate and minimum from the tender master: the scheme wins.
- The UPI reference being required: it is optional.
- The reference panel collapsing: the right column is fixed at 560.
- Enter behaviour: Enter jumps to the panel.
- `refund` never negative: it was negative on an advance.

**It was missing:** validate-first, the gross vs bill split with the purple
chip, releasing the set-off before change, the cash-limit line, the identity
box, TEMP_CR, cheque details, the exact `tenders[]` keys and roll-ups, the
tender route settings, re-tender, and defects D1–D9.

---

## 32. Do not port

- **The borrowed adjust panel** (re-parenting between the screen, F4 and the
  tender). It is one state slice rendered twice.
- **`m_loading`, `QSignalBlocker`s, silent `setSelection`, `clicked` vs
  `setChecked` tricks.** Use user-event handlers plus the `stored | priced`
  union (§5.2–5.3). Never recompute in an effect.
- **The trailing blank row** and every `rowCount − 1` loop; row indexes
  captured across an await.
- **Y/N text flags**; numbers re-parsed from formatted cell text.
- **`AllowNegative = "Y"` hard-coded** on import and load (Q4).
- **`ledGstRate`** in the charge snapshot (CHG-TAX).
- **`ChargeGridController::clear()` keeping the inter-state flag.**
- **The free-scheme-amount subtraction** in the bill total (always 0).
- **Sending the ADJUST row as a tender**; sending `adjustments: []` when
  unread; leaving `adjustments` off `/post`.
- **TEMP_CR outside the credit roll-up (D5); the bool read of
  `temp_credit_block_open` (D7); last-10-digit mobile matching without
  excluding the own bill (D8); `tdIsPdc` keyed on the type name (D9); change on
  a zero-amount CASH line (D3); a negative advance refund (D6); the master rate
  on LOYALTY rows (D4); TEMP_CR under the generic reference gate (D1).**
- **"Put on credit anyway"** that the save check then refuses (D2, Q1).
- **Offerable tenders judged on the session date** instead of the bill date.
- **Re-tender replacement rows without instrument details** (G2); a re-tender
  without the `OTHER` reason.
- **Local-file holds.** Also: restoring a snapshot without re-reading
  party-context.
- **`sbStatus` from a label's text**, and `"POSTED"` sent on `/create`.
- **`sbTotItems` counting blank rows.**
- **`tdPartyLedgerId: null` "because there is no ledger id"**: the customer id
  is the ledger id.
- **A hardware-fingerprint device id.**
- **Print stubs that pretend**; any client-side renderer.
- **The Qt shortcut plumbing** (F5 routed by button visibility, shortcuts
  re-set after `setText`, one QShortcut per tender letter, synthetic key
  events forwarded into the panel).
- **The dead `paintShipSummary`** (build it properly, §20.3), `tblFooterExtra`,
  and the empty root row 4.
- **`kServerHas*` compile-time flags.** The contract is live; code to it.
- **The stale comments** in `api_endpoints.h`: unknown GET keys are not a 400;
  only GET ids need v7; `printAfter` is not honoured.
