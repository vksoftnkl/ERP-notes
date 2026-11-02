# Sale Bill (menu 12) — plan §1 vs what is actually built

**2026-09-22 evening.** Every line checked against
`src/modules/sales/salebill/` and the live endpoint list, not from memory.

**Headline: §1.1 is done. Nine of the twelve sub-sections are not started.**
The screen posts, cancels and amends correctly against A3 — and does nothing
else that A3 made possible.

---

## Endpoints: 4 of 11 wired

| route | called | §  |
|---|---|---|
| `/bills/create` · `/get` · `/delete` | ✅ (pre-A3) | 1.1 |
| `/bills/validate` | ✅ | 1.1 |
| `/bills/post` | ✅ | 1.1 |
| `/bills/cancel` | ✅ | 1.1 |
| `/bills/amend` | ✅ | 1.1 |
| `/bills/open-sources` | ❌ | 1.3 |
| `/bills/party-context` | ❌ | 1.5 |
| `/bills/transport` | ❌ | 1.6 |
| `/bills/tender-context` | ❌ | 1.7b |
| `/bills/retender` | ❌ | 1.7b |
| `/bills/delivery-status` | ❌ | 1.11 |
| `/bills/update-remarks` | ❌ | 1.11 |

---

## §1.1 Post / Cancel / Amend — ✅ DONE

* Save (F5) → `/bills/create`; **Post now SAVES FIRST**, then `/validate`,
  then `/post` — `/bills/post` takes only the keys, so validating the screen
  and posting the stored draft were two different documents whenever the
  operator edited after saving.
* Validate **Ctrl+F7**, Post **Ctrl+Shift+Enter**, Post & Print **Ctrl+Enter**
  — the receipt's chords, F2–F11 left to the existing verb bar.
* `NexWarningStrip` (new, `shared/ui/txn/`) in `root` row 4. Refusals never
  tickable; the override box appears only with `rights.override`.
* POSTED opens read-only; Cancel + Amend on the **identity strip** (not the
  bottom bar — that row's `btnCancel` closes the FORM). Greyed with the reason
  from `locks`, in the order the operator must act: IRN → EWB → returns →
  allocations → day close.
* **Edit routes to Amend on a POSTED bill.** It used to unlock the fields and
  then fail the save with "this bill is posted, use /bills/amend".
* Delete hidden unless DRAFT. Hold / Pick Held untouched. `sales.auto_post`
  wired.

**Not done in 1.1:** the quick strip gains nothing — **Challan import
(Ctrl+F5)**, **Delivery (226)**, and **e-Inv / e-Way on Shift+F2 / Shift+F3**
are all absent.

## §1.2 Mode — ❌ NOT DONE
`sb_bill_mode` appears nowhere in the screen. Menu 12 must open WHOLESALE and
the POS screen POS; it is not a combo.

## §1.3 Bill against a challan / several orders — ❌ NOT DONE
No `Ctrl+F5`, no grid 114, no `/bills/open-sources`. Order import exists but
**replaces** rather than appends — `applyImportedOrder` must merge, grid 118,
`sales.multi_order_bill`. No DC-age red flag.

## §1.4 Charges "from order" column — ❌ NOT DONE
`ChargeGridController` is wired (38 references) but has no source column,
no basis combo (PRORATA / FULL / MANUAL / NONE), and no
`cdSrcCdId` / `cdSrcAccYear` / `cdCarryBasis` on save.

## §1.5 Party panel — ⚠️ PARTLY THERE (corrected after reading the .ui)

An earlier draft of this audit said "no BILL-TO band". **Wrong — most of it
exists.** `grpCustomer` already has `txtCustName`, `txtCustAddr`,
`txtCustPlace`, `txtCustPhone`, `txtCustGstin`, and `grpCredit` has
`lblCreditLimit`, `lblOutstanding`, `lblCrOverdue`, `lblCrOverdueDays`,
`lblAvailBalance`, `lblLoyaltyPoints`.

| mockup band 2 | state |
|---|---|
| customer combo | ✅ `ddCustomer` |
| credit position (limit / used / open / oldest) | ✅ `grpCredit` |
| loyalty balance | ✅ `lblLoyaltyPoints` |
| BILL-TO name / address / place / phone / GSTIN | ✅ already fields |
| BILL-TO **PIN** | ❌ no widget |
| **"Edit for this bill"** lock + override | ❌ — the fields are always open, so a listed customer's master text can be edited with nothing saying it is a one-off |
| PAN / Form 60 state | ❌ |
| advances held · credit notes count | ❌ |
| ship-to address count | ❌ |
| **+ Quick add** customer | ❌ |
| **one call** `/bills/party-context` | ❌ still `fetchPartyCredit` + `fetchCustomerDetail` |

## §1.6 Ship-to / dispatch / transport band — ⚠️ THE CREW HALF EXISTS

Corrected too. §1.6 says `driverId`, `supervisorId`, `loadmanId[]`, `vehicleId`
and `vehicleNo` **stay header fields** — and they are already on the screen:
`ddDriver`, `ddSupervisor`, `ddLoadman`, `txtVehicleNo`, in `grpDelivery`.

**The consignment half is missing entirely:** ship-to picker (grid 110),
dispatch godown, transporter (dd 55), vehicle (dd 56), LR no + date, distance,
mode — i.e. the nested `transport` object that feeds the IRN's ShipDtls /
DispDtls and the e-way Part-A/B. `btnQuickShipping` sits in the `.ui`
**unconnected**, and `PUT /bills/transport` — the band's own verb, the one
thing editable between lock 1 and lock 2 — is never called.

`/bills/get` DOES return a `transport` object (verified on a live response),
so the data is waiting for a widget.

## §1.6a Locking — ⚠️ HALF
`locks.editable.document` is read and greys the verbs. **`transportBand` is
read and then ignored**, because there is no band. Also missing: once an EWB
exists the vehicle must be shown from `gdw_vehicle_no`, not the document —
the plan calls this "the single easiest thing to get wrong on the read side".

## §1.7 Tender dialog — ✅ NOTHING TO ALTER (as planned)
`sale_bill_tender.*` has the rows + hotkeys, card panel, loyalty panel
(`lblLoyalty*` in the `.ui`), F4 `PartyAdjustPanel`, totals. The tender path
runs through the new flow: `onTenderValues` → `saveBill` → amend-or-create →
auto-post.

## §1.7a Cash-sale identity — ❌ NOT DONE
No cash-limit line (269ST) and no PAN / Form 60 box in the tender dialog;
`sbCustPan` / `sbForm60Ref` are never filled.

## §1.7b Re-tender — ❌ NOT DONE
**The counter's most common correction, and none of it exists.** No `Ctrl+F6`,
no grid 133 picker, no `/bills/tender-context`, no `/bills/retender`, no void
ticks or reason combo in the tender dialog, no voided-vs-new footer.
Note `um_can_retender` is granted on menu 12 for `tester1` and nothing reads it.

## §1.8 IRN / e-way bill — ❌ NOT DONE
`btnQuickEInvoice` and `btnQuickEWay` exist in the `.ui` and are **not
connected**. Neither are the mockup's **badge row** (IRN status + ack no + time,
EWB number + valid-upto, COGS posted + voucher) nor **Generate IRN / Part-B
vehicle / Print EWB**. `/bills/get`'s `posting` block already carries all of it
— `irn{status,number,ackNo,ackOn}`, `ewb{status,number,validUpto}`, `cogsAmt`,
`voucherRefno` — verified live. No IRN/EWB badges on the identity strip, no Part-B update, no
registers on Shift+F2 / Shift+F3. Only 3 of 18 quick-strip buttons are wired
at all (`ImportQuote`, `ImportOrder`, `DiscountAll`).

## §1.10 Weight barcodes — ❌ NOT DONE
No `sales.weight_barcode` handling in `SalesItemDelegate`.

## §1.11 Delivery status (menu 226) — ❌ NOT DONE
No list screen, and no **Edit remarks** on the read-only bill.

## §1.12 Temp credits list (menu 257) — ❌ NOT DONE
`src/modules/sales/temp_credit/` does not exist. The backend is live
(`/temp-credits/open`, `/follow-up`) and `HANDOVER_endpoints.md` §7 documents
it.

---

## Also absent vs the mockup, not called out in §1 text

* **Identity strip:** THEIR REF / REF DATE / DUE have no widgets at all.
* **Item grid columns:** the mockup adds **Src** (a DC/SO chip), **Lot / batch**
  (the resolved lot, not the typed batch) and **COGS**. `SalesItemCols` has
  `SrcDocId` (data, not a visible chip), `BatchNo` and `CostPrice` — but **no
  COGS column**; cost price is not COGS.
* **Footer** matches: taxable / GST / total / adv set off / balance all exist.
* **Verb bar** is untouched, exactly as the mockup says it should be.

## Suggested order

1. **§1.7b re-tender** — the most-used correction at a counter, the backend is
   live, and the right is already granted.
2. **§1.5 party-context** — one call replacing two, and the BILL-TO band is
   what the printed bill actually carries.
3. **§1.6 transport band** — shared by four screens, so it is built once and
   reused; also closes the `transportBand` half of §1.6a.
4. **§1.3 challan import** — what makes "bill against a challan" real.
5. **§1.8 IRN / e-way** — buttons already sit in the `.ui`.
6. §1.4 charges, §1.2 mode, §1.10 weight barcode, §1.11, §1.12.
