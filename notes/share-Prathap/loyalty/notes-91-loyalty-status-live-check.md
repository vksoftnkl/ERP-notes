# Notes 91 — Loyalty Status, live check of the build

**Date:** 2026-10-05
**For:** Prathap
**Against:** `plan-backend-loyalty-status.md` (this folder)
**Box:** 192.168.0.106, as VKPOS. Menu 79 was granted to VKPOS today (view / edit / print / export). A full-echo save was diffed: 106 → 107 rows, and the only change is +79.
**Re-run:** `python3 tests/loyaltystatus/run-api.py` (read checks). Add `--write` for the adjust / status checks on ZT-CUST-CASH, which nets to 0.

**Result:** all 9 report routes, both write routes and `/history` answer. **18 checks pass, 4 fail, from 3 defects.**

---

## What passes

- **Rights:** menu-79 view rights are enforced (403 `LST_RIGHT_VIEW` before the grant).
- **Members:** `summary.outstanding` = Σ `lmb_balance_points` (88), and Σ row balance = summary.
- **Per member:** balance = Σ lots.left + unlotted + lapsed; redeemable + cooling = Σ live lots; statement closing = balance, and stays the same with `showReversals=false`.
- **Expiring:** with 365 days, Σ points = the open lots lapsing within a year, and the buckets total the rows.
- **Monthly and gifts** answer, and the last month is `partial`.
- **Export:** `printedAs` carries the filters, and an unknown format gets a 400. Data-only output, as documented, is fine: the client renders it.
- **Adjust:**
  - no approver → 400;
  - beyond redeemable → 422 `SALES_LOYALTY_CAP`;
  - **+10 opens a lot, with unlotted 0, so D2 is built**;
  - −10 restores the balance.
- **Status:**
  - CLOSED with a balance → 422 `LOYALTY_MEMBER_HAS_BALANCE`;
  - SUSPENDED without a reason → 400;
  - suspend and reactivate work, and land in `/history`.

---

## N1 — Scheme summary loses every point on a soft-deleted scheme

All 88 live points (8 EARN rows) sit on scheme `01a0d30a-bb51-…-a2e5` ("Loyalty concurrency test", code `TSTCONCmufew85s`). That scheme has **`lsc_is_deleted = true`**.

`GET /reports/loyalty-status/schemes`:
- returns one row, `…17ed`, a *different*, undeleted scheme with the same name and 0 points everywhere;
- has `summary.outstanding = 0`, while the members tab says 88;
- with `lscId=…a2e5` explicitly returns **`items: []`**;
- `/schemes/monthly` for the same `lscId` returns the 88, so the two routes disagree.

This breaks plan §10 #7: Σ scheme outstanding must equal Σ member balances.

**Fix:** drive the scheme rows from the **ledger**, every `lld_lsc_id` with a non-zero outstanding or a movement in the period. LEFT JOIN `loyalty_scheme` **including deleted rows**, and send `status: 'DELETED'` so the client can grey it. "A scheme stays while its outstanding > 0" (`includeClosedHolding`) should cover deleted schemes as well as closed ones.

**Question (not a defect in this module):** should `DELETE /promotion-loyalty-points/delete` refuse a scheme while members still hold points on it? Here the points outlived their scheme. A soft delete keeps the FK valid, but the scheme vanishes from every picker while its points are still spendable.

## N2 — Members `summary.outstandingValue` is 0.00; the rows sum to 88.00

Each row's `value` is right: it prices the lots at their own scheme's rate (₹1.00 / pt on `…a2e5`, decision D4), giving 28.00 + 60.00. `summary.outstandingValue` is `"0.00"`.

It looks like the same deleted-scheme filter as N1, on the summary path only. **Check:** Σ `items[].value` = `summary.outstandingValue` over the whole filtered set.

## N3 — No member has a scheme, so the Scheme column is blank and best gift can never resolve

`lmb_lsc_id` is NULL on **all 8** live members: the earn path never stamps it. As a result:
- `schemeName` is null on every Members row (the mockup's Scheme column is empty);
- `bestGift` / `eligible` keep to the member's scheme (§4), so they can never be set, and "Eligible for a gift only" always returns nothing, even once a gift exists.

**Fix, either:**
- (a) stamp `lmb_lsc_id` on a member's first EARN, in `LoyaltyLedgerService`, the one writer; or
- (b) in the report, fall back to the scheme of the member's **latest EARN lot**, the same fallback `/members/adjust` already uses for `lscId`.

Recommendation: **(a)**, plus a one-off backfill of the 8 rows from their latest lot, so the till and the report read the same field.

---

## Not defects, for the record

- `/export` is data-only for both `pdf` and `xlsx`. That matches the Ledger Statement export and its own description.
- VKPOS (company CHANDRA SAW MILL) can read another company's wallets by passing that `companyId`. Every other module on the box behaves the same way, because there is no user ↔ company access table. So this is not special to this module, and it is not asked here.
- With the default `hasMobile=true`, the Expiring tab is empty on the box today: no live member has a mobile number. That's test data, not the route.
