# notes (40) — `/bills/party-context` cannot yet replace the customer lookup

**Date:** 2026-09-23 · **Screen:** sale bill, §1.5 party panel · **Severity:** small, one block

## What the plan says

`plan-qt-sales.md` §1.5: `/bills/party-context` on customer pick "replaces
`fetchPartyCredit` + `fetchCustomerDetail` — one call". The BILL-TO band
(sb_cust_name / addr / place / pin / phone / gstin) is filled from it.

## What the route returns (live, 2026-09-23, party 019f659c-3942-7237-89b0-c4899603dd7a)

```
party: { ledId, name, gstType, gstin, stateCode, isWalkIn,
         panNo, panVerifiedOn, form60On, creditAllowed, defaultPriceLevel }
```

No address, place, PIN or phone. `bill-read.service.ts partyContext()` selects
`cus_id, cus_name, cus_gst_type, cus_gst_no, cus_state_code, cus_pan_no,
cus_pan_verified_on, cus_form60_on, cus_credit_allowed, cus_price_level_id,
cus_credit_*` — the four address columns are not in the SELECT.

## What the client does today

* `/master-lookups/party-credit` → **replaced** by `/bills/party-context`
  (credit block, loyalty wallet, advances + credit notes, PAN / Form 60,
  ship-to count, temp credits, open DC / orders). Done.
* `/master-lookups/customer-detail` → **kept**, for the BILL-TO band and the
  area / distance / freight flags / allow_* flags / salesman it also carries.

So it is two calls replacing two, not one replacing two.

## The ask

Add to `party`, from `sales.customers`:

| key | column |
|---|---|
| `addr` | `cus_addr1 ‖ ', ' ‖ cus_addr2 ‖ ', ' ‖ cus_addr3` (the same join `shipTo[].addr` already uses) |
| `place` | `cus_city` |
| `pin` | `cus_pin` |
| `phone` | `cus_phone1` |

That is the whole `sb_cust_*` snapshot, and the screen can then drop
`customer-detail` **for the band**. It would still need area / distance /
freight flags / salesman / allow_discount / allow_promotion / allow_loyalty
from somewhere — if those also move into `party`, the one-call goal is met.
Not urgent: the screen works as built.

## Still open from 2026-09-22

`rights.retender` is not in `/bills/get`'s `rights` (four keys, checked again
today). `sales-context.service.ts rights()` loads `um_can_retender` and then
returns only post/cancel/amend/override. §1.7b (re-tender dialog) waits on it.
See `USER_MENU_POSTING_RIGHTS.md` follow-up.

---

## Added 2026-09-23 evening — the rest of plan §1 is built; these are the server-side gaps it met

| # | what | where it bites |
|---|---|---|
| 1 | `rights.retender` still absent from `/bills/get` (`sales-context.service.ts rights()` drops it) | §1.7b: Ctrl+F6 opens the picker + dialog anyway and says the right is unconfirmed; the post will answer `SALES_RETENDER_RIGHT` |
| 2 | `posting.ewb` carries no `vehicleNo` | §1.6a "show the vehicle from gdw_vehicle_no once an EWB exists" — the band can only say "declared" |
| 3 | `/gst/einvoice/*`, `/gst/ewaybill/*` not on the box | §1.8: e-Inv / e-Way / Part-B buttons call the HANDOVER routes and show the server's answer (404 today) |
| 4 | `fixed.menu_master` has no row 257 | §1.12: the client registers the Temp Credits list on 257; the menu row (parent 1, verbs VIEW/EDIT/PRINT/EXPORT) is yours to add |
| 5 | `proposals.charges[]` has no `cdChgId` | not needed now — the client matches by `cdSrcCdId`, which it stamps at order import |
| 6 | grid runner `search`: grids 114 / 115 answered 0 rows for a bill no that exists, right after their columns were flagged filterable | suspect the preloaded column cache (`configured-grid-sql.service.ts:185`) — a restart may clear it; please check |

**Grid / dropdown ids the box assigned (the plan's numbers were taken):**
grid **113** MAIN LIST - BILL DELIVERY (plan said 134 · menu 226) ·
grid **114** MAIN LIST - TEMP CREDITS (plan 131 · menu 257) ·
grid **115** POPUP - RECENT BILLS FOR RE-TENDER (plan 133) ·
dropdowns **55** TRANSPORTERS / **56** VEHICLES (as planned) ·
ui_table 22 cols **94–98** Src Type / Src Line / Src Year / Lot Id / Bucket (hidden).
JSON in `share/grids/`.

**Runner facts the SQL now follows** (`configured-grid-sql.service.ts`): a grid_param
token is bound as `$N` in EVERY occurrence, quoted or bare; an UNSENT token stays as
literal text — so the client always sends every optional token ('' = unset) and the
SQL compares `col::text = 'token'` rather than casting (a cast of '' folds at plan
time and fails). `search` is not a token: it is ILIKE over the columns flagged
`grid_column_filter`.


---

## Checked again 2026-09-23 after the restart — most of it landed, thank you

| # | state |
|---|---|
| party-context `party` | **DONE** — addr / place / pin / phone + areaId/Name, distanceKm, freight/loading/unloading flags, allow_* flags, salesman. For the client to drop `customer-detail` entirely it still needs: `ename` (e-invoice name), `addr1/addr2/addr3` separately (the bill snapshot keeps line 1 / line 2 apart), `stateName`, `overdueBilling`, `tcsCompany` / `tcsCustomer`, `ecommerceGstin`, `discPerc`. Until then the bill keeps both calls — one writer per field. |
| 1 `rights.retender` | **DONE** — `/bills/get` serves five keys; Ctrl+F6 is gated on it now. `/bills/retender` dry-run on bil00155 (void CASH 400 KEYED_WRONG → CASH 400): **OK** once the new row is numbered AFTER the voided one (tdRowNo 2). Two small things found on the way: (a) the same payload with `tdRowNo: 1` answered a bare **500** on the full row but a clean 409 "Duplicate tender line number" on a minimal row — the unique-violation is mapped on one path and not the other; (b) `/tender-context` rows carry no `tdRowNo`, so the client numbers from `tenders.length + 1` (voided rows included) — sending the number would be safer. |
| 2 `posting.ewb.vehicleNo` | **DONE** — the band reads it once declared |
| 3 GST routes | still absent — the buttons keep showing the server's answer |
| 4 menu 257 | **DONE** (Temp Credits, VIEW/EDIT/PRINT/EXPORT) |
| 6 runner search cache | **DONE** — 114 and 115 search after the restart |
