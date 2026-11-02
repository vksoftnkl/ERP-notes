# Sales Order Entry — React port plan

Scope: **one screen**, `SaleOrderEntry` (`src/modules/sales/saleorder/`), ported to
React/TypeScript against the unchanged backend.

This is a **delta plan**. `plan-react.md` (Quotation) already specifies the pricing
engine, the two grids, the charge distribution, the draft/reducer shape, the policy
snapshot, the customer snapshot and the place-of-supply rule. **None of that is
re-specified here and none of it is forked** — the order renders the same
`domain/pricing` and the same `<ItemGrid>` / `<ChargeGrid>`. Sections below say
either "identical to plan-react.md §N" or spell out what differs.

Read `plan-react.md` first. This document is what the order adds on top.

---

## 1. What the order is, that the quotation isn't

A quotation is an offer. An order is a **commitment with money against it**, so
three subsystems exist here that have no counterpart in the quote:

1. **Advance collection** — the tender dialog, opened in *Advance* mode (F5). Money
   received before any sale exists. Partial by nature.
2. **Fulfilment** — delivered / cancelled / pending per line, and the document's
   fulfil status. **Entirely server-owned**: painted on load, never computed, never
   sent.
3. **Party credit** — what the customer already owes, read the moment they are
   picked, because that is when the decision to accept the commitment is made.

Plus two smaller ones: a **source trail** (a converted quotation) with the customer
locked to it, and an **advance ledger** roll-up that the screen only echoes.

Everything else — pricing, charges, taxes, totals, the item grid — is the quotation
screen with different field names.

---

## 2. Module layout

```
src/features/saleOrder/
  SaleOrderEntry.tsx
  components/
    CustomerBlock.tsx        // reused from quotation, + the source chip (§6)
    OrderInfoBlock.tsx       // refno, order date, delivery date, valid-until,
                             // priority, delivery mode/slot, term
    DeliveryBlock.tsx        // beat, agent, salesman, packed-by, flags
    ItemGrid.tsx             // SHARED with quotation — grid 24, 96 columns (§4)
    ChargeGrid.tsx           // SHARED — grid 21
    TotalsStrip.tsx          // SHARED + advance tendered / balance due / refund
    CreditPanel.tsx          // §7 — replaces the advance panel on screen
    AdvancePanel.tsx         // built, not mounted (§7.3)
  tender/                    // §5 — shared with the bill screen, do not fork
    TenderDialog.tsx
    rows.ts                  // tender master -> row state, type rules
    arithmetic.ts            // pure: surcharge, balance, change
    instruments.ts           // cheque / card / UPI field rules
  state/
    draft.ts  useSaleOrderDraft.ts
  api/
    saleOrder.ts             // get / create / delete
    lookups.ts               // customerDetail, partyCredit, itemPrice, freight,
                             // tenderMasters, quotationForImport
  payload/
    buildSavePayload.ts  buildTenderPayload.ts  parseLoadedDocument.ts
  validate.ts
```

`domain/pricing/` is **imported, not copied**. If the order needs a pricing change,
it is a change to the shared engine with a golden case, or it is not a pricing
change.

---

## 3. Pricing, charges, policy, customer, place of supply

Identical to `plan-react.md` §3, §5, §6, §7. The order's charge context is the same
`{'S', "SALES"}`, the same grid 21, the same `distribute()`.

Two field-name deltas only: the payload prefix is `so`/`soi` instead of `sq`/`sqi`,
and the policy columns are `soFreightCalcType` / `soLoadingCalcType` /
`soDiscAlterBase`.

---

## 4. The item grid

`plan-react.md` §8 applies unchanged, with three notes:

- **Grid id 24**, item picker 71 (`iallow_so` filter), `SalesItemCols` = **96
  columns**. The four reserve columns (`IsReserved`, `ReservedQty`,
  `ReserveExpiresOn`, `LineDeliveryDate`) were removed from the entity — the layout
  config for grid 24 must declare 96 or every column past index 91 is mislabelled.
- **The last four columns are display-only and server-owned**: `DeliveredQty`,
  `CancelledQty`, `PendingQty`, `LineStatus`. Painted from the GET, never editable,
  never in the POST. In React make that a *type*, not a convention: the line's
  fulfilment fields live in a separate `readonly fulfilment?: LineFulfilment`
  branch of the row, and `buildSaveItemPayload` cannot see them.
- **No stock gate.** A back-order is legal on an order (unlike the bill). Stock is
  reported in the hint bar, never blocked.

---

## 5. The tender dialog — one component, two purposes

The Qt screen shares `SaleBillTender` between the bill and the order. Keep that in
React: **one `<TenderDialog>`**, with a `purpose` prop.

```ts
type TenderPurpose = 'settlement' | 'advance';
```

`purpose` is not a preference; the two are different transactions:

| | `settlement` (bill) | `advance` (order) |
|---|---|---|
| must the total be covered? | yes | **no** — a deposit is partial by nature |
| rows offered | all | **minus RRN, LOYALTY, CREDIT, TEMP_CR** |
| credit-note / loyalty panels | yes | never fetched |
| captions | Bill Total / Tendered / Balance | Order Amount / Advance Received / **Balance to Collect** |
| shortfall colour | red (a problem) | amber (the normal case) |

Why those four rows are dropped: a credit note closes out against an invoice and
loyalty is earned on the bill that moves the goods — neither exists yet; CREDIT and
TEMP_CR collect nothing, and an uncollected order simply carries a balance.

### 5.1 Arithmetic — `tender/arithmetic.ts`, pure, tested first

```
rowTotal      = base + surcharge(base, master)
tendered      = Σ rowTotal
settled       = tendered − Σ surcharge − change
balance       = settled − documentAmount
```

**The surcharge is never part of what settles the document.** The Qt code got this
wrong in exactly one place — the per-row ceiling compared the *gross* against the
bill and refused a card settlement of a full bill that carried a fee. Port the rule,
not the bug: every comparison against the document amount uses `base`/`settled`.

The same split reaches the payload: `soTenderAmt` is what crossed the counter
(surcharge included, the customer paid it), `soSurchargeAmt` is the fee, and
`soPayStatus` is decided by the **net**. There is no `so_paid_amt` column, so on
load the net is re-derived as `tender − surcharge − refund`.

### 5.2 Rows come from the tender master

`GET /tender-masters/list`, filtered by `tndIsActive` / `tndIsDeleted` /
effective dates **against the document date, not the wall clock**. Nullable
behaviour flags (`tndNeedsRef`, `tndAllowChange`, `tndAllowInReturn`) are
*overrides* of the tender TYPE's defaults — `{}` means inherit, never false. Keep
the type-default table (`rows.ts`) until the endpoint returns the type's own
columns, and mirror the Qt table exactly.

If the list is empty or the call fails, seed CASH (+ CREDIT in settlement mode) by
hand **and say why in the hint bar**. A silently degraded dialog reads as "this shop
only takes cash".

### 5.3 Instruments — `instruments.ts`

Per tender type, which fields exist and what they mean:

- **CARD** — bank, card no (last 4 only), expiry (`MMM/yyyy`), slip no.
- **CHEQUE** — bank, **cheque no is the reference** (`td_ref_no`), **cheque date**
  (`dd-MM-yyyy`), and `isPdc = date > documentDate`. A cheque has no card number;
  `td_card_last4` is CHECK'd as exactly four digits, so anything else must go as
  null. Bank and number are required for a cheque whatever the master says.
- **UPI / WALLET / BANK** — reference only, labelled by the type.

The cheque panel must not be gated on `needsRef`: a master row with the flag off
would leave no way to key a cheque at all.

### 5.4 Credit override

A cash-only customer's CREDIT row stays **keyable**; using it asks once
("…not allowed to buy on credit. Put this bill on credit anyway?") and remembers the
answer for that settle. In React the answer is draft state, not a member flag.

---

## 6. Source trail and the customer lock

A quotation imported with Ctrl+F3 stamps `srcDocType/Id/Refno/Date` on the draft
and per line (`soiSrcDocId`). Two things follow, both **derived, never toggled**:

```tsx
const source = draft.srcDocId ? { type: draft.srcDocType, ref: draft.srcDocRefno, date: draft.srcDocDate } : null;
const customerLocked = !!draft.srcDocId && !session.allowCustomerChangeOnImport;
```

- A chip beside the status badge: `FROM QUOTATION · SQ-000123 · 04-08-2026`. It
  reads the raw `srcDocType`, so a future `MOBILE_ORDER` needs no code change.
- `customerLocked` disables the customer picker and its snapshot fields with a
  tooltip saying why. The converted order is the same promise to the same party;
  repointing it would leave `so_src_doc_id` naming a document raised for someone
  else, and the carried-over prices were the source customer's.

**Copy as New clears both**, plus every line's `soiSrcDocId` — otherwise one
quotation looks converted twice and the copy opens with a locked customer.

---

## 7. The credit panel

### 7.1 What and when

`GET /master-lookups/party-credit?partyId&companyId&branchId&accYear` (partyId is
the **customer** id). Fetched on: customer picked, quotation imported, order loaded,
copy-as-new. Cleared when the customer is cleared. A failed call clears the panel
and blocks nothing.

Shown: Outstanding (bills), Overdue (bills), Overdue By (days; oldest due date in
the tooltip), Credit Limit (bill limit), Available (bills), and a status line.

### 7.2 Colour and gate

Red only for the two facts that change a decision: overdue money, and a breached
limit (`isAmtLimitExceeded` / `isBillLimitExceeded`). `isCreditCheckEnabled === false`
greys the panel, prints "credit check is off", **and takes it out of the save gate
entirely** — off is an answer, not missing data. The save-time credit confirmation
judges the same object the panel rendered; only a summary that never arrived falls
back to the customer lookup's older `available_balance` / `available_bills`.

### 7.3 The advance panel

Built (`AdvancePanel.tsx`, seven server-owned figures: required, received,
adjusted, refunded, forfeited, balance, status) but **not mounted**: on a new order
every one of them is zero, and the space is worth more as credit. Mounting it later
— on a saved order, beside the credit panel — is a layout decision, not new data:
the load already parses the fields and the save already echoes them back untouched.

Advance *tendered* and *balance due* are not part of that panel; they belong to the
totals strip and stay visible always.

---

## 8. Validation (`validate.ts`, pure)

Everything in `plan-react.md` §10, plus the order's own gates:

| gate | rule |
|---|---|
| customer | required |
| salesman | required when `session.salesmanMandatory` |
| delivery date | ≥ order date |
| valid until | ≥ order date |
| **priority URGENT** | remarks are then mandatory — an order that jumps the queue must say why |
| credit | §7.2 — confirm, never block |
| line qty | ≠ 0 |
| line rate | ≠ 0 unless free-issue (or `freeItemTax`) |
| min price / MRP | rate ≥ min, rate ≤ MRP unless `skipMrp` |
| stock | **no gate** — back-orders are legal |
| at least one item, total > 0, every charge row has a posting ledger | |

---

## 9. Save payload

`plan-react.md` §11's three rules carry over verbatim (nothing repriced on load,
tenant context is the voucher's, server-owned fields handed back untouched). Order
specifics:

- **Server-owned, echoed or omitted**: `soAdvance*` (echoed exactly as loaded — the
  server turns tenders into `soAdvanceRecdAmt`, and recomputing would double-count),
  `soFulfilStatus`, `soBilled/Pending/CancelledAmt`, and per line
  `soiDelivered/Cancelled/PendingQty`, `soiLineStatus` (never sent at all).
- **Settlement roll-ups**: `soTenderAmt` gross, `soSurchargeAmt` fee,
  `soPayStatus` from the net (§5.1), `soRefundAmt`.
- **`tenders[]`** — one `td*` object per settled row: ids, ledger snapshots
  (`tdTenderLedgerId`, `tdSettleLedgerId`, **`tdSurchargeLedgerId`**), the five money
  figures (`tdAmount` base, surcharge, total, received, change), instrument block
  (`tdRefNo`, `tdBankName`, `tdCardLast4` — four digits or null,
  `tdInstrumentDate`, `tdIsPdc`), settlement (`tdSettleStatus = 'NA'`,
  `tdExpectedSettleOn = docDate + settlementDays`), device/user.
- **Source**: `soSrcDoc*` and per line `soiSrcDoc*`, null when there is no source.
- **Not on this screen** (sent null, deliberately): advance policy fields, delivery
  crew, invoice category.

---

## 10. Load / edit / delete

`plan-react.md` §11, plus: repaint the source chip and the lock, re-fetch party
credit (today's standing, not the day the order was raised), derive the net settled
figure, and paint fulfilment + the advance ledger into read-only state. The list
(F8) is the shared `TxnMainView` equivalent with the **sales-order** descriptor —
grid 87, `so_*` key fields, 90-day default window (orders outlive bills).

Delete is keyed `?soId&soAccYear` — an order lives in a year partition, unlike the
quotation's plain id key.

---

## 11. Backend gaps this screen inherits

Everything in `plan-react.md` §13 that isn't quotation-specific, plus:

1. **Grid 24 must be reconfigured to 96 columns** (the four reserve columns are
   gone). Until then the client warns and mislabels.
2. `td_surcharge_ledger_id` was only just added to `acc_tender_detail` — confirm the
   API accepts `tdSurchargeLedgerId` before relying on it.
3. `tdSrcReturnId` (RRN → which credit note) is specified but commented out in the
   Qt payload. Settlement mode can't close a note out until it is live.
4. `tdPartyLedgerId` goes null — the customer lookup returns no ledger id.
5. **Order → bill**: the bill screen sends `sbAdvanceAmt` and expects the server to
   write the `ADVANCE_ADJUST` pair. Until that exists the advance shows on the bill
   and reduces what is collected, but stays open in accounts.
6. `ux_avh_src` is unique on (module, doc id, year) — a **second advance instalment
   on one order has nowhere to post**. Raise before the advance ledger goes live.
7. The sale-order list SELECT has no `so_src_doc_type`, so the list can't badge
   which orders came from quotations.
8. No print pipeline; both print routes stay stubbed in one place.
9. `party-credit`'s `partyId` is assumed to be the customer id — confirm.

---

## 12. Tests

`plan-react.md` §12 in full (the shared engine is the same engine), plus:

- **`tender/arithmetic.ts` golden cases**, written before the dialog renders:
  - bill 6,300 = cash 100 + card 1,000 + UPI 700 + cheque 4,500 → balance 0, one
    PDC row, `tdCardLast4` null on the cheque;
  - order 1,500 = cash 100 + UPI 1,400 with a 14 fee → tendered 1,514, settled
    1,500, balance 0, `soPayStatus` PAID, and **not** PAID if the fee is the only
    thing closing the gap;
  - advance mode: order 50,000, advance 5,000 → saves, `PARTIAL`, no
    full-settlement gate;
  - a card settling a full bill with a fee → **not** refused (the ported bug).
- **Payload round-trip** for `tenders[]` — every `td*` field survives
  `parse(build(draft))`, including the instrument block.
- **Cross-client diff** — save an order in Qt, open it in React, diff totals,
  settlement figures and the tender split.

---

## 13. Phases

| # | Deliverable | Done when |
|---|---|---|
| 0 | Quotation phases 1–2 (`domain/pricing`, draft/reducer) | reused, not rebuilt |
| 1 | Header blocks + customer + source chip + lock | picking a customer fills the snapshot; an imported draft locks the picker |
| 2 | `<CreditPanel>` + party-credit | a customer over their limit paints red the moment they are picked |
| 3 | Item grid (24) + charge grid (21) on the shared engine | totals match the Qt client on one fixture |
| 4 | `tender/arithmetic.ts` + `<TenderDialog purpose="advance">` | every §12 golden case passes; a partial advance closes |
| 5 | `buildSavePayload` + `buildTenderPayload` + `validate` + Save | **a new order saves and the Qt client opens it with identical numbers** |
| 6 | F8 list + load + read-only + Edit + Delete | load → save unchanged → identical stored figures |
| 7 | `purpose="settlement"` for the bill screen | the same dialog settles a bill, no fork |

Phase 4 before phase 5 is deliberate: the settlement figures are payload inputs.

---

## 14. Do not port

`plan-react.md` §15 in full, plus:

- **The gross-vs-base surcharge comparison** (§5.1) — it refuses valid settlements.
- **`m_creditOverridden` as a member flag** — draft state.
- **The advance/credit panel swap by document state** — one panel is mounted, the
  other isn't; visibility is a render decision, not a method that pokes two group
  boxes.
- **Imperative show/hide of the tender instrument fields** — derive them from the
  selected tender type (`instruments.ts`), or the next tender type added will show a
  cheque's date labelled "Expiry".
- **The dialog's silent fallback** to a cash-only master list (§5.2).
- **`Q_UNUSED(printAfter)`** — the print flag is decided in the dialog and returned
  with the result; don't accept a parameter you ignore.
