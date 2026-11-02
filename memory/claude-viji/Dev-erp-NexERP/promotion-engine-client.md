---
name: promotion-engine-client
description: Client-side promotion APPLICATION engine — built and wired to Quotation 2026-08-24; test schemes live on Acme Foods; screen test still pending
metadata: 
  node_type: memory
  type: project
  originSessionId: d53e5ac5-c399-4c45-b65a-d6d098966e50
  modified: 2026-08-24T14:15:53.453Z
---

The Qt client's promotion **application** engine (distinct from the authoring
master — see [[promotion-schema-design]]). Built 2026-08-24. **Resume point:
exercise it on the Quotation screen** — everything below it is verified, the
screen itself is not.

**Shape.** `src/shared/promotions/` holds the widget-free layers —
`promotion_rules` (structs + matchers), `promotion_scheme_cache`,
`promotion_engine` (decides: lines in, outcomes out). The grid write-back is
`src/modules/sales/common/sales_promotion_binder`, beside `SalesItemCols` so
`shared/` does not depend on a module. Same split as `ChargeGridController`.
Wired to Quotation only; Sale Bill and Sale Order keep their empty
`applySchemePromotion()` stubs, and all three call sites were already in place.

**User's locked decisions:** quote promotions are BINDING with a re-check
warning on conversion; multi-scheme uses `prm_priority` + `prm_stack_mode`
(not 3.0's stack-everything); engine + Quotation only in the first pass.

**Rules that were arrived at the hard way, not from the schema:**
- The engine writes **rates** (`SchPerc`/`SchPerQty`) and lets
  `applyLineDiscounts` derive the amount, so the line prints the campaign's own
  terms. It falls back to a bare `SchAmt` in exactly two cases: a cap bit, or a
  second scheme already claimed the line's one rate slot.
- **A bill-level flat amount MUST be converted to a percentage.**
  `quotation_entry.cpp:973` writes `BillSchDiscAmt` *unconditionally* from
  `BillSchDiscPerc`, so a directly written amount is destroyed on the next
  recalc. The item-level `SchAmt` is safe — that branch is an `else if` chain.
- **Never read `NetQty`/`NetGross`/`TaxableAmt` from the grid.** The engine runs
  BEFORE `recalcDocumentTotals()` at every call site, so those hold the previous
  quantity *and* the previous scheme discount. The basis is recomputed from the
  operator's inputs instead. Reading them caused "qty changes do nothing" (the
  promotion lagged one edit) and a percentage feedback loop.
- **FIXED_PRICE is applied as a discount, never by overwriting `Rate`** —
  nothing records the catalogue price to restore. Consequence: it can only ever
  bring a price DOWN, and a fixed price above the selling price does nothing.
  The schema's wording ("whatever it rings up at") could be read as
  unconditional. **User has not yet confirmed which they want.**
- Free rows are **reconciled**, never cleared-and-reinserted: re-inserting
  re-runs the host's price lookup, which is a NETWORK call, on every keystroke.
  That fetch storm is what made 3.0's applier feel slow.

**Performance work already done** (user raised it before scheme counts grow):
scope rows are hashed at parse time (`KIND|scopeId`), so item matching is five
lookups not a scan; and the document-level gate (`schemeIsLive` /
`branchQualifies` / `customerQualifies`) is memoised on a context fingerprint +
cache generation, so it runs per context change, not per keystroke.

**Test schemes exist on the live server**, company `019c8ea6…` (Acme Foods),
created 2026-08-24, all APPROVED and deliberately STACKABLE priority 5 so they
coexist: `TPERC` (ITEM_QTY×DISC_PERC 10%), `TQTY` (₹3/unit), `TFLAT`
(ITEM_AMOUNT×DISC_AMT ₹100 flat), `TFIX` (FIXED_PRICE 99), `TFREE`
(BILL_AMOUNT×FREE_ITEM, 1 per ₹5000 max 3), `TPERITEM` (DISC_PER_ITEM 7%).
Delete them when done. The user's own `2026` and `FRRE` are EXCLUSIVE at
priority 1/2 — do not disturb.

**`TFIX` looks broken but is not:** item 001 in unit DZ sells for ₹40 against a
fixed price of ₹99, so there is nothing to give. Item 001/Bottle is ₹400, where
it would apply. Note also that `ck_pri_unit` makes the UNIT part of an ITEM
rule — "001 in DZ" is not "001 in Bottle", a common authoring trap. Both
outcomes now explain themselves via `binder.notes()`, but **nothing displays
those notes yet** — wiring them to a status line is the obvious next job.

**Still open, backend side:** `prm_approved_by_name` missing from the header
payload; `master-lookups/customer-detail` returns no `cgr_id` and no city id, so
`CUSTOMER_GROUP` and `CITY` party rules can never match and fail *silently*;
`promotion_usage` has no API, so caps and budget are unenforceable anywhere.
`/promotion-scheme/list` now returns the full graph (children included), so the
cache is one call — but **do not pass `prm_branch_id`**: it matches the header
column literally and drops company-wide schemes.

**Why:** chain-store ERP; promotions are money given away at the counter, so a
scheme that silently does nothing is as bad as one that gives too much.
**How to apply:** rebuild before testing — the instance running on 2026-08-24
predates most of the day's fixes. See [[qt-offscreen-test-harness]] for how the
engine and binder were verified without a screen.
