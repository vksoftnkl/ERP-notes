---
name: stock-adjustment-family
description: "One module (stocks/stock-adjustment) serves ADJUSTMENT/ISSUE/DAMAGE/EXPIRY_WRITEOFF + re-lot + Move stock (BUCKET_MOVE); per-line direction from the reason (lineDirection REASON, svi_direction), outward always at the branch average, BLOCK forced, StkAdj Stock Journal voucher; menu 264, grid 122, ui_table 41"
metadata:
  node_type: memory
  type: project
  originSessionId: 04f8789e-47a6-43d6-8027-632e2629b5c4
  modified: 2026-09-30T05:40:00.000Z
---

Built 2026-09-28 (plan-nestjs-stock-adjustments). `src/modules/stocks/stock-adjustment/`:
one controller under `/stock/adjustment` for four `svh_voucher_type`s picked by
`header.voucherType`, plus `/stock/reasons` (picker Q17 + maintenance). Rule records set
`postShape: 'SIMPLE'`, `lineDirection: 'REASON'`, `allowsLot`, `blockNegative`,
`allowsRepeatHolding`. The engine reads each line's direction from `svi_direction`
(migration 20260928150000, nullable smallint ±1, stored for ADJUSTMENT lines only), else the
reason's `srm_direction` (line reason, then header reason), and the txn type from the same
rule (ADJUST_PLUS/MINUS per line; ISSUE takes a reason's single issue type; DAMAGE /
EXPIRY_WRITEOFF fixed). Quantities stay magnitudes (`ck_svi_qty_sign`); the API turns a
signed BOTH-reason quantity into |qty| + direction.

Engine facts that changed with it: an OUTWARD line (or a count's minus row) is ALWAYS
valued at `sic_avg_cost_rate`, whatever the rate source or a keyed cost; `resolveLots` opens
lots only for inward lines under REASON; header totals net like a count; the shared
`assertPayloadRules` keeps a `lotId` when `allowsLot`. Accounts: ADJUSTMENT/ISSUE/DAMAGE/
EXPIRY_WRITEOFF post on the 'StkAdj' Stock Journal type (by code, SADJ00001), same
shortage/excess netting as PHYSICAL; RELOT_OUT/RELOT_IN rows are excluded (value carried).
Setting `stock.expiry_writeoff_grace_days` (INT, COMPANY) gates EXPIRY_WRITEOFF.

**Why:** five modules for five `stock_voucher` types would drift; the per-line direction is
the one thing the shared engine lacked.

**How to apply:** a re-lot is one ADJUSTMENT with a RELOT_OUT (lotId) + RELOT_IN (identity)
pair that balances per item; the service forces AVG_COST on it. A BOTH reason usable on an
adjustment must have `srm_allowed_txn_types` empty or containing ADJUST_*; the seed's BOTH
rows are counts-only.

**Move stock (notes 60 / D-A3, built 2026-09-29, migration 20260928210000_stock_bucket_move):**
`header.voucherType: 'BUCKET_MOVE'` is a SAVE-only kind stored as `svh_voucher_type = 'ADJUSTMENT'`;
a loaded doc is a move iff a live line has `svi_to_bucket` (new nullable column,
`ck_svi_to_bucket`: one of the five and ≠ svi_bucket). Rule record `STOCK_ADJUSTMENT_RULES.BUCKET_MOVE`
= voucherType ADJUSTMENT, typeCode ADJ, `postShape: 'BUCKET_MOVE'` (new shape): the engine skips
pick/resolve (lot required) and writes BUCKET_OUT −1 (svi_bucket) + BUCKET_IN +1 (svi_to_bucket)
per line at the line cost; the service stamps `svi_direction = −1` so a BOTH move reason still
values it as outward (branch average). `applyItemCost` EXCLUDES BUCKET_* rows (else the IN half
stamps the average as last purchase rate); `varianceLegs` excludes them (no voucher); header
totals count BUCKET_OUT rows as a magnitude. Seed reasons MOVE_DAMAGED / MOVE_SALEABLE (BOTH,
allowed {BUCKET_OUT,BUCKET_IN}); a move must cite a reason naming a bucket type, and a
bucket-only reason is refused elsewhere. `pick-stock` rows carry the lot's supplierId/Name.
`GET /stock/adjustment` returns `kind` (ADJUSTMENT/RELOT/BUCKET_MOVE/...), same test as grid 122's
`kind_code`. Menu 264 "Stock Adjustment" (one menu for every kind, verbs without AMEND/OVERRIDE),
list grid 122 (tokens icompany_id, ibranch_id, iacc_year, ifrom_date, ito_date, ikind, istatus),
line grid ui_table 41. Sales still accepts any bucket on a line and the item price lookup sums
every bucket — reported, not fixed (the note said that is a sales-side fix).
Tests: `test/stock-adjustment.e2e-spec.ts` (rolled back, 14 cases; 10–14 are moves).

**Notes 65 (2026-09-30, migration 20260930110000_stock_adjustment_list_and_layout):** a RELOT_IN
names no lot, so "IN ≠ OUT lot" is decided by `resolveInwardLots` (same `effectivePolicyCte` +
`lotIdentityKeyColumns` as the engine's attachLotsToLines) at save/validate/post, and again inside
the post via the `afterPost` hook (`assertRelotLandsElsewhere`, stamped lots). The header DTO is
`OmitType(shared header, totals)` + signed totals (see [[dto-redeclare-keeps-base-swagger]]). Grid 122
and `kindOf` exclude sales shadow vouchers: `COALESCE(svh_link_src_module,'STOCK') = 'STOCK'`. The box
(192.168.0.106 = this machine) holds the Qt screen's 31-column ui_table 41; the migration mirrors it
for fresh DBs. costRate is per BASE unit (svi_value = base qty × cost_rate). Tests: 18 e2e cases +
dto/save-stock-adjustment.dto.spec.ts.
