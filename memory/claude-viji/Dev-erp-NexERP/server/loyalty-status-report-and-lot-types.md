---
name: loyalty-status-report-and-lot-types
description: Loyalty Status (menu 79) built 2026-10-05 — lots are EARN/OPENING/ADJUST/TRANSFER via LOYALTY_LOT_TXN_TYPES (D2); ck_lld_lot_required admits a lot on ADJUST; adjust()/drain() are the screen's writers; rights code prefix LST_; 403/422 bodies on these routes nest errors under body.message
metadata:
  type: project
---

Built from the Loyalty Status plan (Prathap, 2026-10-05): `src/modules/reports/loyalty-status`
(nine GET routes under `/reports/loyalty-status/*`, README inside) and the two actions in
`src/modules/sales/loyalty/members` (`POST /loyalty/members/status|adjust`, `GET …/history`).
Migration `20261005100000_loyalty_status_report` made menu 79 visible, added
`ix_lld_scheme_period`, and widened `ck_lld_lot_required` so an ADJUST row may name a lot.

**D2 was applied:** `LOYALTY_LOT_TXN_TYPES` (loyalty.types.ts) = EARN, OPENING, ADJUST, TRANSFER
is what `lots()`, `redeemable()`, the member recompute's next-expiry, `expiryRun()` AND the
report's set SQL all filter on. A positive ADJUST is now a real lot; a negative one draws FIFO
through `consume(…, 'ADJUST', …)` and needs `approvedBy`. `drain()` zeroes a wallet (every open
lot + any unlotted remainder) for a forced CLOSE (D6). Both live in `LoyaltyLedgerService`, the
single ledger writer the source spec enforces.

**Why:** a plan that says "the only DB change is one index" met `ck_lld_lot_required`, which
forbade ADJUST from naming a lot — without the widening, a take-back would lower the wallet
and leave every lot spendable (redeemable > balance). And until D2 a goodwill +50 was never
redeemable, sweepable or visible (plan §6 F1).

**How to apply:** never re-narrow the lot filter to EARN in one place; change the constant.
Rights on 79 use `assertMenuRight` with prefix `LST` (view / edit / delete / print / export).
On routes WITHOUT a module-level exception filter the global filter nests a 403/422's
`errors` under `body.message`, so the harness `codesOf()` returns [] — assert on
`JSON.stringify(res.body)` instead (test/sales/loyalty-status-http.e2e-spec.ts does).
A `loyalty_scheme_gift` row's `lsg_unit_id` is an `item_unit_conversion.iuc_id`, and
(scheme, item, unit) is unique — two gifts need two items.

**Notes 91 (2026-10-05, live check) fixed the same day:** `earn()` now stamps `lmb_lsc_id` on a
wallet's FIRST EARN (migration `20261005120000` backfilled the old wallets from their latest
movement) — the Scheme column, best gift and `eligible` all read that field and it was NULL
everywhere. The scheme summary is driven from the LEDGER and LEFT JOINs `loyalty_scheme`
INCLUDING soft-deleted rows (`status: DELETED`), because 88 live points sit on a deleted scheme.
`summary.outstandingValue` = Σ of the rows' `value`. Open question raised there, not done:
should `DELETE /promotion-loyalty-points/delete` refuse a scheme whose members still hold points?

Related: [[stock-engine-ddl-lives-outside-repo]], [[pre-existing-test-failures]],
[[user-menu-transaction-rights]]
