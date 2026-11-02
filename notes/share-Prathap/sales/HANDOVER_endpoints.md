# Sales transactions — the endpoint contract (HANDOVER for Prathap)

**2026-09-20.** Every route the six documents need, with the exact JSON both
sides build to. The reasoning is in `plan-backend-sales.md`; the rows and
legs each verb writes are in `sales_posting_flow.md`; this file is only the
wire. Where a shape already exists (`/bills/create`, `/bills/get`) it is
EXTENDED, never changed. Read `README.md` for the run order (SQL first:
11 → 13 → 14 → 15 → 29 → 30 → 31).

> **STATUS 2026-09-22 — §2 to §7 are BUILT AND DEPLOYED.** 373 routes on the
> box. Bills carry all fourteen verbs; sale-orders have post/cancel/amend;
> delivery-challans, dc-returns and sale-returns are complete. §11 and §12 are
> the only parts of this file that are still specification.

House conventions (`api-house-conventions`): four keys on every verb, one
`partyId` (= `cus_id` = `led_id`), no `/list` (grids), DRAFT → POSTED →
CANCELLED, cancel = reversal never delete, dates `YYYY-MM-DD`, ids uuidv7
minted by the client for documents.

**Money and quantity on the wire — read this before "fixing" it to strings.**
JSON numbers are IEEE-754 float64, so the question is whether a value
round-trips exactly:

| column | digits | as a JSON number |
|---|---|---|
| money `numeric(15,2)` | 15 | **safe.** float64 holds ~15.95 significant digits; 2470.00 and 999999999999.99 both round-trip exactly |
| qty / rate `numeric(18,6)` | 18 | **NOT always safe** — 18 significant digits exceeds float64. Real supermarket quantities are far inside the range, but a rate or a value near the column's ceiling is not guaranteed |

So the wire is **not** the leak, and rewriting 373 routes to send strings buys
little. **The leak is arithmetic**, and it is real — see the rule below, which
is not optional.

> **Decimal end to end.** Money and quantity must never become a JavaScript
> `number` for *computation*. `sale-order.service.ts` does exactly that today:
> `asNumber()` → float64, `roundAmount()` = `Math.round(v*100)/100` (which turns
> 1.005 into 1.00, because float64 stores it as 1.00499999…), and line 186 is
> `const AMOUNT_EPSILON = 0.005`. **An epsilon on money is the system admitting
> its arithmetic drifts.** 79 files use `Prisma.Decimal` correctly; that service
> is the exception, and `temp-credit.service.ts` repeats it with `num()`.
> Rules: keep `Prisma.Decimal` through every add/subtract/multiply/divide and
> compare; **no epsilon comparisons on money**; round ONCE at the column with a
> stated mode (`ROUND_HALF_UP`); and **distribute by largest-remainder** — round
> each share down, hand the leftover paise to the largest remainders — so parts
> sum to the whole exactly. `SIX_TXN_CONTROLS.md` §6 is the long version.

Every success is `{ "data": … }`; every error is the existing filter shape
`{ "statusCode", "error", "message", "code", "details"? }` with the codes of
§9. A validation-style refusal (422) ALSO carries `warnings[]` / `refusals[]`
in `details` in the §2.2 shape, so the client paints them instead of popping.

---

## 1. Common blocks

### 1.1 Keys
```jsonc
{ "sbId": "…", "sbCompanyId": "…", "sbBranchId": "…", "sbAccYear": "2026-2027" }
```
(`soId`, `sdcId`, `sdrId`, `srId` on the other documents, same three tenant
keys with their prefixes.)

### 1.2 `posting` — on `/get`, `/post`, `/amend` of every document
```jsonc
"posting": {
  "voucherId": "…", "voucherRefno": "bil00042", "postedOn": "2026-09-20T11:42:10+05:30",
  "registerId": "…",                       // acc_voucher_doc_register.gdr_id, null when none
  "cogsAmt": 620.00,                       // what THIS document posted as COGS (0 under PERIODIC / bill-against-DC)
  "loyaltyEarned": 24, "loyaltyRedeemed": 0,
  "irn": { "status": "GENERATED", "number": "a1b2…", "ackNo": "112210043391",
           "ackOn": "2026-09-20T11:42:30+05:30", "message": null },
  "ewb": { "status": "GENERATED", "number": "391000123456",
           "generatedOn": "…", "validUpto": "2026-09-22T23:59:00+05:30", "message": null }
}
```
`status` ∈ NA · PENDING · GENERATED · FAILED · CANCELLED · EXPIRED · REJECTED.

### 1.3 `locks` — why cancel / amend would be refused (computed, never assumed)
```jsonc
"locks": {
  "returns": 0, "allocations": 0, "dayClosed": false,
  "irnLive": true, "ewbLive": true,
  "irnCancelWindowUntil": "2026-09-21T11:42:30+05:30",
  "ewbValidUpto":         "2026-09-23T23:59:00+05:30",

  // What may still be EDITED. The client greys from these and never
  // recomputes the rule itself — see 1.3a.
  "editable": {
    "document":      false,   // items, values, party
    "transportBand": false    // the whole txn_transport_detail row
  }
}
```

### 1.3a The two lock points — DERIVED, never stored

| # | when | frozen |
|---|---|---|
| 1 | **POSTED** | the document's content — items, values, party. The transport band is still open, because nothing has been declared yet |
| 2 | **IRN or e-way bill GENERATED** | **everything, the band included — and AMEND IS GONE. Cancel is the only verb left** |

**Lock 2 is: once it has been declared, it is not ours to edit.** Either
generation trips it — an IRN with no e-way bill freezes the band just as an
e-way bill does, because the IRN payload carries `ShipDtls` / `DispDtls`. There
is no third, partial state.

**No amendment after either generation, for any document.** `/amend` answers
`SALES_IRN_LIVE` (IRN) or `SALES_EWB_LIVE` (e-way bill without one) and there
is no override and no right that unlocks it — not `rights.amend`, not
`sales.allow_posted_amend`. A declared document is corrected by **cancelling
it**, inside the portal's own window, and raising a fresh one; past that window
a sale bill is corrected by a credit note, never by an amendment.

That is also why `rights.amend` must not light the Amend button on its own: the
right says the USER may amend, lock 2 says this DOCUMENT may not.

**A later vehicle change is a PORTAL operation, not an edit.** The lorry breaks
down, the goods trans-ship — that goes to NIC through
`POST /gst/ewaybill/vehicle` and is recorded in `acc_voucher_doc_ewaybill`
(`gdw_vehicle_no`, `gdw_transport_doc_no`, `gdw_transport_doc_date`,
`gdw_transport_mode`). It does **not** write `txn_transport_detail` and does
not touch the document.

That gives each table one job and no overlap:

| | holds | after lock 2 |
|---|---|---|
| document + band | what we PREPARED to declare | frozen |
| `gdw_` | what we DECLARED, and every update since | the only thing that moves |

**Consequence for any screen or report: once an e-way bill exists, the CURRENT
vehicle is `gdw_vehicle_no`, not the document's.** The document keeps the
original. Reading the vehicle off the document after a Part-B update shows a
lorry that is no longer carrying the goods.

**Compute both locks, never store them.** Lock 1 is `*_status = 'POSTED'`;
lock 2 is `gde_status = 'GENERATED' OR gdw_status = 'GENERATED'`. A
`ttd_is_locked` column would be a second source of truth that drifts — the same
trap the transition stamps were.

**The server enforces it regardless of what the client sent:**

| attempted | refused with |
|---|---|
| `/amend` on a document with an IRN | `SALES_IRN_LIVE` |
| `/amend` on a document with an e-way bill and no IRN | `SALES_EWB_LIVE` |
| `/create` on a declared document | `SALES_IRN_LIVE` / `SALES_EWB_LIVE` |
| any write to the band once an IRN or EWB is GENERATED | `GST_DECLARED_LOCKED` |
| any document write while `dayClosed` | `SALES_DAY_CLOSED` |

**The band still needs its own verb** — `PUT /bills/transport` and the same on
challans and returns — because between lock 1 and lock 2 the document is frozen
while the band is not, and `/amend` is the wrong door for it. See §2.12.

### 1.4 `rights` — the four `user_menus` flags for THIS user on THIS menu
```jsonc
"rights": { "post": true, "cancel": true, "amend": true, "override": false }
```

### 1.5 `warnings[]` / `refusals[]` — from `/validate`, and inside a 422
```jsonc
"refusals": [ { "code": "SALES_CASH_LIMIT", "message": "…", "field": "tenders[0].tdAmount",
                "statutory": { "code": "CASH_TXN_LIMIT_269ST", "value": 200000, "effectiveFrom": "2017-04-01",
                               "isCompanyOverride": false } } ],
"warnings": [ { "code": "SALES_RATE_BELOW_MIN", "level": "WARN", "message": "…", "line": 2,
                "overridable": true } ,
              { "code": "HSN_DIGITS", "level": "INFO", "message": "…", "overridable": false } ]
```
A WARN with `overridable: true` passes when its code is in the request's
`overrides[]` AND `rights.override` is true (the server re-checks both).

---

## 2. Bills — `/api/v1/bills`

### 2.1 `POST /bills/create` — the DRAFT (exists; extended)
Body as today (`save-bill.dto`: header + `items[]` + `charges[]` + `tenders[]`
+ `adjustments[]`), plus the new header fields (30 §1):
```jsonc
"sbBillMode": "WHOLESALE",            // POS | WHOLESALE — from the launching screen
"sbUsrRefdate": "2026-09-18",
"custOverride": false,                // true = the BILL-TO band was edited for a listed customer
"sbShipAddrId": null, "sbShipName": "…", "sbShipAddr": "…", "sbShipPlace": "…", "sbShipPin": "641001",
"sbShipPhone": "…", "sbShipStcd": "33", "sbShipGstin": null,
"sbDispatchGodownId": "…", "sbDispatchBranchId": "…",
"sbTransportMode": "ROAD", "sbTransporterId": "…", "sbTransporterName": "…", "sbTransporterGstin": "…",
"sbLrNo": "VRL-88213", "sbLrDate": "2026-09-20", "sbDistanceKm": 112,
"sbCustPan": null, "sbForm60Ref": null,
"sbLoyaltyMemberId": null
```
Items gain `sbiSrcItemId` (the source LINE: `soi_id` / `sdi_id`), `sbiBucket`,
`sbiLotId` (a PROPOSAL — the engine decides), `sbiPromoUsageId`. Charges
gain `cdSrcCdId`, `cdSrcAccYear`, `cdCarryBasis` (PRORATA | FULL | MANUAL |
NONE). Tenders gain, on a TEMP_CR row only:
```jsonc
"tempCredit": { "name": "Karthik", "mobile": "9094442679", "place": "Namakkal",
                "addr": null, "days": 7, "notes": "will pay on Saturday" }
```
Rules: `sbStatus` in the body is IGNORED (server-owned; a draft is DRAFT);
a POSTED `sbId` → **409 `SALES_BILL_POSTED`** ("use /bills/amend"); the
walk-in customer (`sales.default_customer_id`) keeps whatever `sbCust*` the
client typed; a listed customer's `sbCust*` are copied from the master
unless `custOverride`. Response: the `/get` shape (§2.7) with `sbStatus
"DRAFT"`.

### 2.2 `POST /bills/validate` — the dry run
Body = §2.1's body + `"overrides": ["SALES_RATE_BELOW_MIN"]`. Writes
nothing. Response:
```jsonc
{ "data": { "ok": false,
  "refusals": [...], "warnings": [...],            // §1.5
  "rights": {...},                                 // §1.4
  "proposals": {
    "charges": [ { "cdSrcCdId": "…", "cdSrcAccYear": "…", "chgName": "Freight", "orderAmount": 300.00,
                   "carriedSoFar": 0.00, "proposed": 120.00, "basis": "PRORATA", "isFinalBill": false } ],
    "advances": [ { "ablId": "…", "ablAccYear": "…", "refno": "sor00009", "pending": 500.00, "proposed": 500.00 } ],
    "creditNotes": [ { "ablId": "…", "ablAccYear": "…", "refno": "srt00002", "pending": 4000.00 } ],
    "loyalty": { "memberId": "…", "balance": 1240, "redeemable": 1100, "rate": 0.50,
                 "minPoints": 100, "maxPoints": 1240, "maxRedeemAmount": 620.00, "multiple": 1,
                 "earnPreview": 24 },
    "tempCredit": { "openOnMobile": [ { "atcId": "…", "billRefno": "bil00038", "balance": 1200.00, "dueDate": "2026-09-18" } ] }
  } } }
```

### 2.3 `POST /bills/post` — the one-way door
```jsonc
{ ...keys, "overrides": ["…"], "printAfter": true }
```
One transaction, flow §5.1 steps 1–13, status last; after commit the GSP /
print / notify jobs. Idempotent: a POSTED id answers 200 with the same
response. Response: the `/get` shape with `sbStatus "POSTED"` + `posting` +
`locks` + `rights`. Refusals: 422 with §1.5 (any REFUSE, or a WARN not
overridden); 409 `SALES_BILL_CANCELLED`; 403 `SALES_RIGHT_POST`.

### 2.4 `POST /bills/cancel`
```jsonc
{ ...keys, "reason": "customer refused delivery" }
```
Reason mandatory (`ck_tsl_reason_required`). Refusals (409, one code):
`SALES_RETURN_LOCKS_BILL`, `SALES_ALLOCATION_LOCKS_BILL`, `SALES_IRN_WINDOW_PASSED`
(inside the window the server cancels the IRN first, synchronously, and
proceeds only on success — `GST_IRN_CANCEL_FAILED` otherwise), `SALES_EWB_WINDOW_PASSED`,
`SALES_DAY_CLOSED`, `SALES_YEAR_LOCKED`, 403 `SALES_RIGHT_CANCEL`. Response:
`{ "data": { ...keys, "sbStatus": "CANCELLED", "reversalVoucherRefno": "bil00042R", "cancelledOn": "…" } }`.

### 2.5 `POST /bills/amend` — R20 on the bill
```jsonc
{ ...§2.1 body..., ...keys, "baseRevision": 1, "editRemark": "rate keyed wrong on line 2",
  "overrides": ["…"], "printAfter": false }
```
Gate: `sales.allow_posted_amend` (default TRUE) → 409 `SALES_AMEND_OFF`.
`baseRevision <> sb_revision_no` → 409 `SALES_REVISION_STALE`. Same refusal
list as cancel; the IRN case: inside the window the IRN is cancelled and a
new one generated after the post; past it 409 `SALES_IRN_WINDOW_PASSED`.
Response: the `/get` shape with `sbRevisionNo` + 1. Before/after into
`audit.audit_log`.

### 2.6 `POST /bills/delete` — DRAFT only (exists; narrowed)
`{ ...keys }`. A POSTED id → 409 `SALES_BILL_POSTED` ("use /bills/cancel").
The order-cancel side effect this route carried moves to `/sale-orders/cancel`.

### 2.7 `GET /bills/get?sbId&sbCompanyId&sbBranchId&sbAccYear` (exists; extended)
Today's shape (header + `items[]` + `charges[]` + `tenders[]` +
`adjustments[]`) plus:
```jsonc
"sbRevisionNo": 1, "sbBillMode": "WHOLESALE", "sbDeliveryStatus": "PENDING",
"posting": {...}, "locks": {...}, "rights": {...},          // §1.2–1.4
"sources": [ { "kind": "DC", "docId": "…", "accYear": "…", "refno": "dc00007", "date": "2026-09-12",
               "lines": 2, "takenQty": 5, "openQtyAfter": 0 },
             { "kind": "ORDER", "docId": "…", "accYear": "…", "refno": "sor00009", "date": "…", "lines": 1 } ],
"tempCredits": [ { "atcId": "…", "name": "Karthik", "mobile": "…", "balance": 36000.00, "dueDate": "…", "status": "OPEN" } ]
```
`sbPostedVoucherId`, `sbDocRegisterId`, `sbCogsAmt` are read-only here and
never accepted on input.

### 2.8 `GET /bills/open-sources?companyId&branchId&partyId&kind=DC|ORDER&accYear?`
```jsonc
{ "data": [ { "docId": "…", "accYear": "…", "refno": "dc00007", "date": "2026-09-12", "purpose": "SUPPLY",
              "ageDays": 8, "pastWindow": false,                       // DC_RETURN_WINDOW_DAYS per company
              "lines": [ { "lineId": "…", "lineNo": 1, "itemId": "…", "itemName": "…", "unitId": "…", "unitName": "Nos",
                           "lotId": "…", "batchNo": "L-2609-04", "godownId": "…", "docQty": 5, "openQty": 5,
                           "rate": 200.00, "taxId": "…", "taxPerc": 18 } ] } ] }
```
For ORDER: `openQty` = `soi_pending_qty`; a DC of purpose ≠ SUPPLY carries
`"convertRequired": true`.

### 2.9 `GET /bills/party-context?partyId&companyId&branchId&accYear&billDate?`
One call on customer pick:
```jsonc
{ "data": {
  "party": { "ledId": "…", "name": "…", "gstType": "REGULAR", "gstin": "…", "stateCode": "33",
             "isWalkIn": false, "panNo": "AABCS1234F", "panVerifiedOn": "…", "form60On": null,
             "creditAllowed": true, "defaultPriceLevel": 2 },
  "credit": { "limitAmount": 500000, "limitBills": 5, "creditDays": 30, "used": 229100.00, "openBills": 3,
              "oldestOpenDays": 41, "amtExceeded": false, "billExceeded": false, "daysExceeded": true,
              "mode": "REFUSE" },                                          // sales.credit_limit_mode
  "cashToday": 150000.00,                                                   // cash received from this party on billDate (269ST)
  "advances": [ { "ablId": "…", "ablAccYear": "…", "refno": "sor00009", "pending": 500.00, "date": "…" } ],
  "creditNotes": [ { "ablId": "…", "ablAccYear": "…", "refno": "srt00002", "pending": 4000.00, "date": "…" } ],
  "loyalty": { "memberId": "…", "cardNo": "M-00871", "balance": 1240, "redeemable": 1100, "rate": 0.50,
               "minPoints": 100, "maxPoints": 1240, "maxRedeemPerc": 25, "multiple": 1, "schemeId": "…" },
  "shipTo": [ { "saaId": "…", "name": "…", "addr": "…", "place": "…", "pin": "…", "stcd": "33", "gstin": null, "isDefault": true } ],
  "tempCredits": [ { "atcId": "…", "billRefno": "bil00038", "balance": 1200.00, "dueDate": "…" } ],
  "openSources": { "dc": 1, "orders": 2 }
} }
```

### 2.10 `PUT /bills/delivery-status`
```jsonc
{ ...keys, "event": "PACKED", "remarks": "…", "vehicleNo": "…", "lrNo": "…" }
```
`event` ∈ VERIFIED (only when `sales.require_verification_before_dispatch`)
· PACKED · DISPATCHED · DELIVERED; order enforced → 409 `SALES_DELIVERY_ORDER`;
writes `txn_status_log` + `sb_delivery_status`. Response `{ "data": { "sbDeliveryStatus": "PACKED" } }`.

### 2.11 `PUT /bills/update-remarks`
```jsonc
{ ...keys, "sbRemarks": "…", "editRemark": "why" }
```
Text only, on a POSTED bill; `txn_status_log` row `REMARKS_EDITED`.

### 2.12 `PUT /bills/transport` — the band, on its own verb
```jsonc
{ ...keys, "transport": { "direction": "OUTWARD",
    "from": { "godownId": "…", "branchId": "…", "addrId": null, "name": "…",
              "addr": "…", "place": "…", "pin": "…", "phone": "…",
              "stcd": "33", "gstin": "…" },
    "to":   { … same shape … },
    "mode": "ROAD", "transporterId": "…", "transporterName": "…",
    "transporterGstin": "…", "lrNo": "…", "lrDate": "2026-09-20",
    "distanceKm": 112 } }
```
Writes `public.txn_transport_detail` (33). **A separate verb because
`/bills/amend` is refused once an IRN exists while the band is still open**
(§1.3a) — the band outlives the document's own editability.

Does **not** bump `sb_revision_no`: the document did not change. Writes a
`txn_status_log` row `TRANSPORT_EDITED`.

**Refused entirely once an IRN or an e-way bill exists** —
`GST_DECLARED_LOCKED`. This route is for the window between POST and
declaration, which is the only time the document is frozen and the band is not.
A vehicle change after declaration is a PORTAL operation
(`POST /gst/ewaybill/vehicle` → `gdw_`), not a write here.

The same route exists on `/delivery-challans`, `/sale-returns` and
`/dc-returns`, identical body, `direction` INWARD on the two returns.

### 2.12a `GET /bills/tender-context?sbId&sbCompanyId&sbBranchId&sbAccYear`
```jsonc
{ "sbBillRefno": "bil00042", "sbBillDate": "2026-09-20", "sbCustName": "…",
  "sbBillAmt": 2470.00, "sbPaidAmt": 2470.00, "sbBalanceAmt": 0.00,
  "tenders": [ { "tdId": "…", "tenderName": "UPI", "tdTenderId": "…",
                 "tdAmount": 2470.00, "tdIsPdc": false, "pdcMoved": false,
                 "isLoyalty": false, "isTempCredit": false } ],
  "canRetender": true, "reason": null          // e.g. "day closed"
}
```
**The re-tender dialog's ONLY read. Deliberately not `/bills/get`** — no items,
no charges, no party context, no transport. The dialog opens over a half-keyed
next bill at a busy counter (plan-qt §1.7b): it must be small and it must not
give the screen any reason to load the sale.

`isLoyalty` / `isTempCredit` / `pdcMoved` are flags the dialog needs to warn
before the operator commits, not after the server refuses.

The picker in front of it is **grid 133** (POPUP - RECENT BILLS FOR RE-TENDER)
— this device and session, today, newest first — because the operator says
"the one two minutes ago", not a bill number. House rule: a list is a grid, not
a `/list` route.

### 2.13 `POST /bills/retender` — change how it was paid, not what was sold
```jsonc
{ ...keys,
  "voids": [ { "tdId": "…", "reason": "UPI_FAILED" } ],   // the rows that did not happen
  "tenders": [ { "tdTenderId": "…", "tdAmount": 450.00, … } ],  // what really happened
  "remark": "UPI failed at counter, paid cash" }
```
Works on **DRAFT and POSTED**, and on a bill that already carries an **IRN** —
the tender is not on the e-invoice, not in GSTR-1 and not on the e-way bill, so
changing it declares nothing new and §1.3a's lock 2 does not reach it. That is
also why it is its own verb and not `/amend`.

**What it writes.** The voided rows keep their place: `td_is_voided = true`
with `td_void_reason`, `td_voided_on`, `td_voided_by` (35). They are NOT
deleted — the failed UPI happened at the counter and a till review has to see
it, and `td_is_deleted` would say "keyed wrong" when the truth is "payment
failed". New rows carry `td_replaces_id`. Every till and day-book report reads
`NOT td_is_voided`; audit reads both.

**On a POSTED bill it posts a CONTRA, type `TndC`** — `DR` the tender that
really happened, `CR` the one that did not, same amount. **The party balance
never moves and the bill stays settled throughout.** Reversing the original
posting and re-posting would unsettle it in between, and a failure at that
moment would leave it unsettled. On a DRAFT there is no voucher yet, so it is
only the tender rows.

**Guards, and the last two are the ones that bite:**

| refusal | when |
|---|---|
| `SALES_RETENDER_AMOUNT_MISMATCH` | the new tender total ≠ the voided total. A different amount is a part payment or a refund, not a re-tender |
| `SALES_DAY_CLOSED` | the drawer has already been counted with the wrong figure — that is a day-book correction, not this |
| `SALES_RETENDER_PDC_MOVED` | a PDC tender that has been deposited or cleared. Money moved at the bank; that is a bounce (`/cheques`), not a re-tender |
| `SALES_RETENDER_RIGHT` | `um_can_retender` false (35) — its own flag, NOT `um_can_amend` |
| — | voiding a **LOYALTY** tender routes through `LoyaltyLedgerService`: the REDEEM rows are reversed and **the points come back**. Skip this and the customer is silently robbed |
| — | voiding a **TEMP_CR** tender must clear its `accounts.acc_temp_credit` row — it is the WHO on the bill's balance row (31) |

Those last two are why this cannot be a row swap: two tender types have
consequences outside `acc_tender_detail`.

`txn_status_log` row `RETENDERED`. Does **not** bump `sb_revision_no` — the
document did not change. `sb_tender_amt` / `sb_paid_amt` / `sb_balance_amt` are
recomputed from the live rows.

---

## 3. Sales orders — `/api/v1/sale-orders` (exists; extended)

| route | body | notes |
|---|---|---|
| `POST /sale-orders/post` | `{ soId, soCompanyId, soBranchId, soAccYear }` | CONFIRMED + `stock_reservation` rows; response `{ data: { soStatus: "CONFIRMED", warnings: [ { code: "SALES_RESERVE_SHORT", line: 2, short: 4 } ] } }` — never a refusal |
| `POST /sale-orders/cancel` | `{ keys, reason }` | 409 `SALES_ORDER_DELIVERED` when any `soi_delivered_qty > 0`; releases reservations |
| `POST /sale-orders/amend` | full create body + keys + `baseRevision` + `editRemark` | after a partial delivery only undelivered lines may change → 409 `SALES_ORDER_LINE_DELIVERED` |
| `DELETE /sale-orders/delete` | as today | DRAFT only → 409 `SALES_ORDER_CONFIRMED` |

---

## 4. Delivery challans — `/api/v1/delivery-challans` (new)

Body prefix `sdc` / `sdi` (29 §2). `POST /create` (draft upsert),
`POST /validate`, `POST /post`, `POST /cancel {keys, reason}`,
`POST /amend {…, baseRevision, editRemark}`, `POST /delete` (DRAFT),
`GET /get?sdcId&sdcCompanyId&sdcBranchId&sdcAccYear`,
`GET /open-for-bill` (= §2.8 with kind DC).

`/post` response `posting`: `cogsAmt`, `voucherRefno` (type DCh; null under
PERIODIC), `registerId`, `ewb` (never `irn`). Extra refusal codes:
`SALES_DC_PURPOSE_NOT_ALLOWED` (`comp_dc_purposes`), `SALES_DC_REQUIRES_ORDER`,
`SALES_DC_LINE_OVER_ORDER`; cancel: `SALES_DC_BILLED`, `SALES_DC_RETURNED`.
`POST /delivery-challans/convert-purpose {keys, purpose: "SUPPLY", remark}`
on a POSTED DC (logged).

**Locks (§1.3a):**

| # | when | frozen |
|---|---|---|
| 1 | POSTED | lines, quantities, party. The band stays open; so does **purpose**, via convert-purpose |
| 2 | **EWB GENERATED** | everything — band and purpose included. No amend; cancel only |

A challan **never carries an IRN** (Q8), so `gde_*` stays NA for its life and
only the e-way bill trips lock 2 here. Until it does, a POSTED challan has
**two** editable surfaces: the band (`PUT /delivery-challans/transport`) and
`convert-purpose`. Both are single-purpose verbs, neither goes through
`/amend`, neither bumps the revision, and **both are refused after
generation** — purpose with `GST_DECLARED_LOCKED` too, because the purpose
became the e-way bill's sub-supply type and has already been declared.

`locks.editable` carries a third flag here: `"purpose"`. It is false once
lock 2 trips, and also once any line is billed or returned, because the purpose
decided what those became.

Cancel keeps its own refusals on top: `SALES_DC_BILLED`, `SALES_DC_RETURNED`,
and a live e-way bill past `EWAY_CANCEL_HOURS` — cancel the EWB at the portal
first, or let it expire.

---

## 5. DC returns — `/api/v1/dc-returns` (new)

`POST /create` (`sdrDcId` + `sdrDcAccYear`, lines with `sdriDcItemId`),
`POST /post`, `POST /cancel {keys, reason}`, `POST /delete` (DRAFT),
`GET /get`, `GET /open-lines?sdcId&sdcAccYear` →
`[ { dcItemId, lineNo, itemName, unitName, dcQty, billedQty, returnedQty, openQty, lotId, batchNo, godownId, rate, taxPerc } ]`.
Refusals: `SALES_DCR_OVER_OPEN` (qty > `sdi_open_qty`), `SALES_DCR_DC_NOT_POSTED`.

**Locks (§1.3a) — the strictest of the four documents:**

| # | when | frozen |
|---|---|---|
| 1 | POSTED | everything. **There is no `/amend` on this document at all**, declared or not — a mistake is cancel + re-enter. The band stays open |
| 2 | **EWB GENERATED** | the band too. Cancel only |

No IRN ever (a DC return moves goods and raises no debt, `DocDtls.Typ` CHL), so
only the e-way bill trips lock 2. `locks.editable.document` is **false the
moment it is POSTED** and never true again — there is no amend flag to consult
on this document. Between lock 1 and lock 2 the band
(`PUT /dc-returns/transport`, `direction` INWARD) is its only editable
surface; after lock 2 it has none.

---

## 6. Sale returns — `/api/v1/sale-returns` (new)

`POST /create`:
```jsonc
{ "srId": "…", "srCompanyId": "…", "srBranchId": "…", "srAccYear": "…",
  "srIsAgainstBill": true, "srBillId": "…", "srBillAccYear": "…",
  "srSettleMode": "ADJUST",                       // CASH | ADJUST | ADVANCE
  "srReasonId": "…", "srReturnReason": "…", "srCustId": "…", …header (29 §1)…,
  "items": [ { "sriBillItemId": "…", "sriBillAccYear": "…", "sriReturnQty": 2, "sriFreeQty": 0,
               "sriCondition": "RESTOCK", "sriBucket": "SALEABLE", "sriGodownId": "…", … } ],
  "tenders": [ … CR rows when CASH … ] }
```
`POST /validate`, `POST /post`, `POST /cancel {keys, reason}`, `POST /amend`,
`POST /delete` (DRAFT), `GET /get`,
`GET /sale-returns/bill-lines?sbId&sbAccYear` →
`[ { billItemId, lineNo, itemName, unitName, billQty, returnedQty, returnable, isFree, freeType, lotId, batchNo, rate, taxPerc, cost, schemeId } ]`.
`/post` response `posting` adds `settlement: { refunded, adjusted, credited, adjustedBills: [ { ablId, refno, amount } ] }`,
`loyaltyReversed`, `promoClawback`. Refusals: `SALES_RETURN_OVER_QTY`,
`SALES_RETURN_BILL_NOT_POSTED`, `SALES_RETURN_ITEM_NOT_ALLOWED`
(`item_allow_sales_return`), `SALES_FREE_RETURN_OFF`; warnings
`SALES_RETURN_WINDOW`, `GST_CREDIT_NOTE_CUTOFF`; cancel: `SALES_CN_APPLIED`.

**Locks (§1.3a) — the bill's two, with one difference that is easy to miss:
the IRN here is a CREDIT NOTE.**

| # | when | frozen |
|---|---|---|
| 1 | POSTED | lines, quantities, settlement mode. The band stays open |
| 2 | IRN (`DocDtls.Typ` **CRN**, not INV) **or** inward EWB generated | everything, band included. No amend; cancel only |

A credit note that has been declared is corrected by cancelling it and raising
another — there is no amendment path, and no right unlocks one.

Two refusals sit on top of lock 1 and neither is about GST:

* **`SALES_CN_APPLIED`** — once the credit note has been set off against a bill
  or refunded, cancel is refused. Money has moved; reverse it with a document,
  not by un-doing this one.
* **`GST_CREDIT_NOTE_CUTOFF`** — a credit note cannot be raised, and therefore
  its IRN cannot be generated, past the statutory cut-off for the original
  invoice's financial year. That is a date from `fn_statutory_limit`, read per
  company, never a constant.

`PUT /sale-returns/transport`, `direction` INWARD — between lock 1 and lock 2
only.

---

## 7. Temp credits — `/api/v1/temp-credits` (new, `31_temp_credit.sql`)

**BUILT 2026-09-22 and live.** Documented here from the shipped code
(`src/modules/sales/temp-credit/`), not as a spec — this section was missing
while `RUN_ORDER.md` was already telling A3 to build to it, and the DTOs carry
`/** §7 */` for a heading that did not exist.

A TEMP_CR tender is an outstanding on the bill's balance row; `acc_temp_credit`
is the **WHO** — the walk-in who took goods on a promise, keyed by name, mobile
and place because there is no ledger for them. The balance follows
`abl_pending_amount`; nothing here is typed twice.

### 7.1 `GET /temp-credits/open` — the list (grid 131, menu 257)

| query | |
|---|---|
| `companyId` | **required**, uuid |
| `branchId` | optional — omit for every branch |
| `status` | comma list of `OPEN, PARTIAL, SETTLED, WRITTEN_OFF, CANCELLED`. **Default `OPEN,PARTIAL`** |
| `search` | one box over **name, mobile OR bill number** (`ILIKE`) — the counter says "the Ramesh one" or reads the mobile off a slip |
| `overdueOnly` | boolean, default false. True = `due_date < today` **AND** `balance > 0` |

Ordered `atc_due_date, atc_created_on` — oldest promise first. Row:

```jsonc
{ "atcId": "…", "atcAccYear": "2026-2027",
  "partyId": "…", "partyName": "Sri Krishna Traders",   // null for a pure walk-in
  "billId": "…", "billRefno": "bil00042", "billDate": "2026-09-20", "billAmount": 2470.00,
  "name": "Ramesh", "mobile": "9843012345", "place": "Peelamedu",
  "days": 7, "dueDate": "2026-09-27",
  "creditAmount": 2470.00, "balance": 2470.00, "status": "OPEN",
  "daysOverdue": 0,                                     // 0 once balance is 0, never negative
  "promiseDate": null, "followupOn": null, "remarks": null,
  "branchId": "…" }
```

`daysOverdue` is **computed, never stored**, and is 0 whenever the balance is 0
— a settled credit is not overdue however late it was paid.

### 7.2 `PUT /temp-credits/follow-up` — record the chase

```jsonc
{ "atcId": "…", "atcAccYear": "2026-2027",
  "promiseDate": "2026-09-30",     // optional; null CLEARS it, omitted KEEPS it
  "remarks": "called, will pay Monday" }   // required, max 250
```
Stamps `atc_followup_on` / `atc_followup_by` and writes an audit row carrying
the before and after of `promiseDate` + `remarks`. Returns
`{ atcId, promiseDate, followupOn, remarks }`.

**Note the three-way `promiseDate`:** absent = unchanged, `null` = cleared, a
date = set. A client that always sends the field will wipe a promise it did not
mean to.

### 7.3 What is NOT a route here

* **The balance is never written by this module.** `BillBalanceRecomputeService`
  refreshes `atc_balance_amount` / `atc_status` whenever it recomputes the `abl`
  row — so a receipt against the bill settles the temp credit as a side effect,
  which is the point. `atc_balance_amount` is on the §10 ignore list.
* **The receipt hook:** `/receipts/open-items` carries
  `tempCredit { name, mobile, dueDate }` per bill and accepts `?mobile=` so the
  collector finds the bill from the phone number; `/receipts/post` stamps
  `atc_last_receipt_id`.
* There is no create route. A temp credit is born with its bill, from the
  TEMP_CR tender.

---

## 8. GST — see `../gst/HANDOVER_endpoints.md`

The bill calls `POST /gst/einvoice/generate?sync=true`, `POST /gst/ewaybill/generate?sync=true`,
`POST /gst/ewaybill/vehicle`, `GET /gst/einvoice/get`, `GET /gst/einvoice/preview` with
`{ gdrId }` or the bill keys; `/bills/post` enqueues the same two after commit.

---

## 9. Error codes (all 4xx; `code` is what the client switches on)

| code | status | when |
|---|---|---|
| SALES_BILL_POSTED / _CANCELLED | 409 | create/delete on a posted id; post on a cancelled one |
| SALES_RIGHT_POST / _CANCEL / _AMEND / _OVERRIDE | 403 | `user_menus` flag false |
| SALES_AMEND_OFF | 409 | `sales.allow_posted_amend` false |
| SALES_REVISION_STALE | 409 | `baseRevision` ≠ current |
| SALES_RETURN_LOCKS_BILL · SALES_ALLOCATION_LOCKS_BILL | 409 | cancel/amend blocked |
| SALES_IRN_LIVE | 409 | create / amend on a document that has an IRN — lock 2 (§1.3a) |
| SALES_EWB_LIVE | 409 | amend on a document with an e-way bill and no IRN — lock 2 |
| GST_DECLARED_LOCKED | 409 | any write to the transport band once an IRN or EWB is GENERATED — lock 2 |
| GST_EWB_EXPIRED | 409 | a Part-B update at the portal after `gdw_valid_upto` |
| SALES_DAY_CLOSED | 409 | any document write while `locks.dayClosed`; also blocks re-tender |
| SALES_RETENDER_AMOUNT_MISMATCH | 422 | new tender total ≠ voided total (§2.13) |
| SALES_RETENDER_PDC_MOVED | 409 | re-tender of a PDC already deposited or cleared |
| SALES_RETENDER_RIGHT | 403 | `um_can_retender` false |
| SALES_IRN_WINDOW_PASSED · SALES_EWB_WINDOW_PASSED · GST_IRN_CANCEL_FAILED | 409 | |
| SALES_DAY_CLOSED · SALES_YEAR_LOCKED · SALES_BACKDATE | 409 / 422 | |
| SALES_CASH_LIMIT · SALES_PAN_REQUIRED · SALES_CREDIT_LIMIT · SALES_RATE_BELOW_MIN · SALES_DISC_CAP · SALES_TENDER_DAILY_LIMIT · SALES_TENDER_MIN_MAX · SALES_LOYALTY_CAP · SALES_PROMO_NOT_LIVE · SALES_CONTRACT_RATE_MISMATCH(skipped) · SALES_DC_LINE_OVER · SALES_ORDER_LINE_OVER · SALES_STOCK_NEGATIVE · SALES_EWAY_TRANSPORT_MISSING · SALES_TEMP_CREDIT_DETAILS_MISSING · SALES_TEMP_CREDIT_DAYS · SALES_TEMP_CREDIT_AMOUNT · SALES_TEMP_CREDIT_OPEN · SALES_RRN_RETIRED · GST_NIC_FIELD_* · HSN_DIGITS · SALES_DELIVERY_ORDER | 422 (REFUSE) or WARN | as `sales_posting_flow.md` §9 and the settings decide |
| SALES_STOCK_QTY_MISMATCH | 500-class, refused | Σ stock rows ≠ Σ line qty (should never fire; the assertion) |

---

## 10. What the client sends that the server must IGNORE

`sbStatus`, `sbPostedVoucherId`, `sbDocRegisterId`, `sbRevisionNo`,
`sbCogsAmt`, `sbiCogsAmt`, every fulfilment cache (`sdi_billed_qty`,
`soi_delivered_qty`, `sb_returned_amt`…), `atc_balance_amount`. Strip them
with `@Exclude` on input; echo them on output.

---

## 11. Cross-cutting, all six documents — SPEC, not yet built

From `SIX_TXN_CONTROLS.md`, raised 2026-09-22. These are the same on the
quotation, order, challan, DC return, bill and sale return, so they are
specified once here rather than six times above.

### 11.1 Scan a ref no and LOAD the document

```
GET /sales/resolve-scan?companyId&branchId&accYear&code=<scanned>
→ { "docType": "SALE_BILL", "docId": "…", "accYear": "2026-2027",
    "refno": "bil00042", "status": "POSTED",
    "branchId": "…", "sameBranch": true, "sameYear": true }
```

One route for all six. The operator points the gun at the customer's slip
instead of typing a number.

* It **resolves**, it does not load — the client then calls that document's own
  `/get`. One route that returned six different document shapes would be a
  second contract to keep in step.
* **`sameBranch` / `sameYear` are the point.** A scan that lands in another
  branch or another year must offer to switch, never silently return nothing.
* Each screen declares what it ACCEPTS: a bill ref scanned on the sale return
  loads it as the source bill; on the DC return it is rejected (billed goods are
  a sale return). The server returns the docType; the screen decides.
* `SALES_SCAN_NOT_FOUND` 404, `SALES_SCAN_AMBIGUOUS` 409 when a code matches
  more than one document.

### 11.2 Barcode on the item grid

```
GET /sales/resolve-item-scan?companyId&branchId&code=<scanned>&godownId=
→ { "kind": "ITEM" | "WEIGHT" | "LOT",
    "itemId": "…", "itemName": "…", "uomId": "…",
    "qty": 1.000, "rate": null,        // WEIGHT barcodes carry both
    "lotId": null, "batchNo": null }
```

* **`kind: WEIGHT`** is the embedded qty/price barcode already specified for the
  bill (Q48) — the same parser serves all six, and the server owns it so the
  prefix rules live in one place.
* **`kind: LOT`** is how goods go back to the lot they left on, on both returns,
  without typing a batch number.
* **Increment, never append.** The CLIENT rule, stated here because the server
  cannot enforce it: scanning the same tin five times gives one line of qty 5,
  matched on item + lot + godown + bucket. Five lines of 1 makes a printed bill
  unreadable and hides a voided line in the noise.
* On the two RETURN screens a scan may only touch a line that **already exists**
  — the source document is the ceiling. An unknown scan there is an error, not
  a new line.

### 11.3 GST first, then the document — the cancel gate

Already half-present in §1.3a as a refusal; stated here as the **order of
operations**, because A3's `/cancel` has to implement it:

> If an IRN or an e-way bill has been generated, the GST document must be
> cancelled **before** the transaction can be cancelled.

| state | `/cancel` answers | the screen shows |
|---|---|---|
| IRN live, inside the window | `SALES_IRN_LIVE` | "Cancel IRN first" + the button |
| IRN live, **window passed** | `SALES_IRN_WINDOW_PASSED` | **no button.** The correction is a credit note, and the screen must say so rather than offer a dead action |
| EWB live, inside `EWAY_CANCEL_HOURS` | `SALES_EWB_LIVE` | "Cancel e-way bill first" + the button |
| EWB expired | — | cancel proceeds |
| both live | `SALES_IRN_LIVE` | IRN first, then EWB, then the document |

Both windows come from `fn_statutory_limit` per company — **never a constant**.
`locks` (§1.3) already carries `irnCancelWindowUntil` and `ewbValidUpto`, so the
client greys from those and never recomputes the rule.

---

## 12. Void and the till session — **DEFERRED 2026-09-22, by the user**

> **PARKED, NOT CANCELLED.** The user: *"no need void bills problem and session
> issue we can handle later."* Nothing in this section is to be built now, and
> **it no longer blocks §11**.
>
> It is kept in full because the reasoning is the expensive part and will not be
> cheaper to rediscover. When it comes back, the order still stands: **the
> session table first, then void** — a void without a session is an event with
> no owner.
>
> **Partial cover exists in the meantime.** `public.print_log` already records
> every print attempt (§12.4), so "the printer wasn't working" is still a
> checkable claim today: no row at all means no print was ever attempted. That
> is not the whole control, but it is not nothing.

`SIX_TXN_CONTROLS.md` §7 is the reasoning. The short version: **cancel is an
accounting correction and cannot answer "was there money on the counter that is
now not in the drawer?"** — which is the question a supermarket must be able to
ask about any shift without trusting the operator's account of it.

### 12.1 The session is the missing table

`*_session_id` is already on all six headers, on `acc_tender_detail`,
`acc_voucher_header`, `acc_bill_adjustment` and `acc_temp_credit` — and it is a
**free-floating uuid with no foreign key to anything**. `audit.user_login_sessions`
is a LOGIN session (token, IP, user agent); it has no float, no count, no close.

```
POST /till-sessions/open   { companyId, branchId, deviceId, openingFloat, remarks? }
                           → { sessionId, openedOn, openedBy }
GET  /till-sessions/current?companyId&branchId&deviceId
POST /till-sessions/close  { sessionId, counted: [ { tenderId, countedAmount } ], remarks? }
                           → { expected[], counted[], difference[], needsApproval }
GET  /till-sessions/summary?sessionId
                           → bills, value, voids (count + value), voided lines,
                             tenders by type, expected vs counted
```

`close` returns the **difference per tender** and `needsApproval` when it
exceeds the threshold. That difference is a number somebody signs for, per
shift — not per month.

### 12.2 Void

```
POST /bills/void              { ...keys, reasonCode, remarks, baseRevision }
POST /delivery-challans/void  { ...keys, reasonCode, remarks, baseRevision }
```

**Only the bill and the challan.** A quotation void is meaningless and an order
is cancelled; those four keep cancel only. (Confirm — §13.)

* **The number is KEPT** and the document stays visible with a VOID badge. A
  missing number is the loudest possible alarm, so numbers must never go missing.
* `reasonCode` from a controlled list — `CUSTOMER_LEFT`, `WRONG_ITEM`,
  `PRICE_DISPUTE`, `PRINTER_FAILURE`, `OPERATOR_ERROR`, `SYSTEM_ERROR` — plus
  free-text `remarks`. Free text alone is not a control.
* The server stamps, at void time and never recomputed: session, device,
  minutes since post, and **`wasPrinted`** — whether a SUCCESSFUL `print_log`
  row existed for this document at that moment. That field is the whole control.
* It reverses exactly what cancel reverses. The accounting is identical; the
  **evidence** is the difference.
* **Approval** required when `wasPrinted`, or over `VOID_APPROVAL_AMOUNT`, or
  more than `VOID_SESSION_LIMIT` already in this session, or later than
  `VOID_MINUTES` after the post. All four are settings, per company.

### 12.3 Void line

Every line removed or reduced **after it was added** is logged — item, qty,
value, who, when, session, and whether the document had already been printed.
**Drafts included:** "it was only a draft" is exactly the cover this exists to
remove. The per-session count and value is what gets watched; one is a mis-scan,
forty in a shift by one operator in the last second of each sale is not.

### 12.4 `print_log` is the control that already exists

`public.print_log` records every print ATTEMPT — `plg_status`, `plg_error`,
`plg_printed_on`, `plg_printed_by`, `plg_copy_no`. So "the printer wasn't
working" is checkable:

| `print_log` shows | means |
|---|---|
| a failed row with an error | the printer really did fail — the claim is true |
| **no row at all** | **no print was ever attempted** |
| a successful row, then a void | a document was printed, handed over, and then voided |

**`print_log` is currently missing from `ensure_acc_year_partitions`** —
tolerated because it only bites on the roll into 2028-29. If it becomes an
anti-theft control, that gap closes first.

### 12.5 New error codes

| code | status | when |
|---|---|---|
| `SALES_NO_OPEN_SESSION` | 409 | a counter write with no open till session (if sessions are made mandatory — §13) |
| `SALES_SESSION_CLOSED` | 409 | writing into a session that has been closed |
| `SALES_VOID_APPROVAL` | 403 | a void needing approval, without it |
| `SALES_VOID_NOT_ALLOWED` | 409 | void on a document type that has none |
| `SALES_SCAN_NOT_FOUND` / `SALES_SCAN_AMBIGUOUS` | 404 / 409 | §11.1 |

---

## 13. Open questions — **CLOSED 2026-09-22: all three were §12**

All three (void on all six vs bill+challan; whether a session gates the counter;
session per device vs per operator) belong to §12, which is deferred. **They are
not blocking anything.** They come back with §12.

**Nothing in this file is waiting on the user. §11 is the live work.**

