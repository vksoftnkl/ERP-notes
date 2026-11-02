# NestJS — Stock adjustments: adjustment, issue, damage, expiry write-off, re-lot

Written 2026-09-26. **Phase 2 of `plan-nestjs-stock-engine-port.md`**, and it needs that plan's
§1.4 (lot pick) and §1.9 (accounts posting) first. It is a new file and edits no existing plan.

Until now the adjustment family had **no flow doc, no screen plan and no mockup**
(REVIEW_2026-09-05:181, :239-240). The engine side was designed in `19_stock_posting.sql`;
its spec is `spec_prestrip/19_stock_posting.prestrip.sql.txt`, which covers
`fn_svh_txn_map` 19:93-206 and `fn_svh_post` 19:207-585.

**The client mockup comes first** (PNG, the house pattern), before any client code. This file
fixes the contract the mockup is drawn against.

---

## 0. One module, one document, one screen

The five kinds are all `stock.stock_voucher` rows; only `svh_voucher_type` differs. **Build
one module (`stocks/stock-adjustment/`) and one Qt screen with a Type selector.** Do not build
five. That is the same reasoning as the transfer plan's "one endpoint set": the document, the
lines, the validation and the post are identical, and five modules drift.

| Type (screen) | `svh_voucher_type` | direction | ledger txn type (19:93 map) |
|---|---|---|---|
| Adjustment | `ADJUSTMENT` | per LINE, from its reason (IN → +1, OUT → −1) | `ADJUST_PLUS` / `ADJUST_MINUS` |
| Issue (internal use, sample, gift) | `ISSUE` | −1 | the reason's own txn type (`SAMPLE_ISSUE`, `GIFT_ISSUE`, …) else `ADJUST_MINUS` |
| Damage | `DAMAGE` | −1 | `DAMAGE` |
| Expiry write-off | `EXPIRY_WRITEOFF` | −1 | `EXPIRY_WRITEOFF` |
| Re-lot (wrong batch fix) | `ADJUSTMENT` with the RELOT pair | −1 on the old lot, +1 on the new, **same document** | `ADJUST_MINUS` + `ADJUST_PLUS` |

**Two deliberate changes to 19's map:**
- **ADJUSTMENT direction is per line, not per document.** 19 read it from the header reason and
  raised on a BOTH reason. A real stock-taking correction has both signs in one sheet, and
  `svi_reason_id` exists on the line for this. Rule: the line reason wins, then the header
  reason. A reason whose `srm_direction` is BOTH needs a signed quantity on the line (+ in,
  − out). An IN/OUT reason fixes the sign, and a quantity of the wrong sign is refused.
- **ISSUE no longer trusts `srm_allowed_txn_types[1]` blindly** (REVIEW SHOULD-FIX). The txn
  type is taken from the reason only when the reason lists exactly one type **and** that type is
  an issue type (`SAMPLE_ISSUE`, `GIFT_ISSUE`, `ADJUST_MINUS`). Otherwise it is `ADJUST_MINUS`.

`TRANSFER_*`, `REPACK_*`, `OPENING`, `PHYSICAL` and `RECEIPT` are refused by this module.

---

## 1. Routes (own URL family)

All under `/api/v1/stock/adjustment`, with rights on the adjustment menu id (client registration).

| method + path | purpose |
|---|---|
| `POST /stock/adjustment` | save a DRAFT (create or update; `svhId` present = update) |
| `GET /stock/adjustment?svhId&accYear&companyId&branchId` | load one document, with item, lot, reason and godown names |
| `GET /stock/adjustment/validate?svhId&accYear&companyId&branchId` | preflight, per-line messages (§3) |
| `POST /stock/adjustment/post` | post (engine + accounts voucher, one transaction) |
| `POST /stock/adjustment/cancel` | DRAFT → delete header; POSTED → mirror reversal + `Rev` voucher |
| `DELETE /stock/adjustment` | soft delete a DRAFT |
| `GET /stock/adjustment/pick-stock?companyId&branchId&godownId&itemId?&bucket?&search?` | **the "pick stock from balance" picker** (REVIEW query 5): balance-grain rows (godown × lot × bucket, available > 0) with item, batch, expiry, MRP and average cost. Outward lines are chosen from this |
| `GET /stock/reasons?companyId&voucherType&direction?` | the reason picker, i.e. 16q **Q17**: shared + company rows merged, a company row hides the shared one with the same code, filtered by `srm_allowed_txn_types` for the chosen type |

The list screen is a grid (house rule: no `/list` route). It uses a `ui_table` plus a grid
registration over `stock_voucher` with `svh_voucher_type IN (the five)`, and its bare params
always sent.

## 2. The document

**Header:** company, branch, acc_year, device, doc date/datetime, `svh_voucher_type`,
`svh_from_godown_id` (outward) or `svh_to_godown_id` (inward; an ADJUSTMENT with both signs uses
one godown for both, in `svh_from_godown_id`), `svh_reason_id` (the default for the lines),
`svh_rate_source` (inward only, default `AVG_COST`), and remarks.

**Line:** item, uom (`iuc_id`), qty (**signed** only for a BOTH reason), free qty (none on
adjustments: refuse it), lot:
- **outward:** `svi_lot_id` from the picker, or none and then picked by issue strategy (engine
  §1.4, D1);
- **inward:** identity (batch, expiry, MRP, SP, serial, supplier), resolved by the engine's
  lot resolve, exactly like an opening line;

plus bucket, `svi_reason_id`, `svi_remarks`, and `svi_cost_rate` (inward only; outward is always
stamped by the engine, so a keyed outward cost is **ignored and overwritten**).

**Re-lot is one document with a pair of lines.** `RELOT_OUT` on the wrong lot and `RELOT_IN` on
the right identity, with **equal base qty**. The IN posts at the OUT's stamped cost rate, so the
value is carried. It is refused unless the pair balances per item. This is the sanctioned
wrong-batch fix (REVIEW:261-263); a half on its own is shrinkage.

## 3. Validation (`/validate` and again inside `post`)

Per line, messages keyed by line number like the opening preflight:

1. The reason is present on the line or the header, active, and allows this voucher type's txn
   type (Q17's filter). `srm_require_remarks` → remarks required (**API-enforced; no trigger
   does it**).
2. The direction agrees with the reason (§0).
3. **Outward:** available in the godown ≥ qty for that lot and bucket. The negative-stock
   policy is **forced to BLOCK** for this module whatever the item says (D-A1 below): writing
   off stock you do not have is a data error, not a sale.
4. **Expiry write-off:** the lot's expiry ≤ doc date + `expiry_writeoff_grace_days` (company
   setting, default 0). Otherwise use DAMAGE.
5. **Inward:** cost resolves to > 0 after the rate source (AVG_COST / LAST_PURCHASE / LOT_COST /
   MRP / MANUAL); a zero cost is refused (19's rule). The tracked identity facets are complete
   (the same check as the opening preflight, **including supplier and sale price**, which the
   opening `/validate` was found to miss).
6. **Re-lot:** the pair balances per item; the OUT and IN lots differ; same item, same uom base.
7. Freeze: `assertNotFrozen` (a godown under a DRAFT PHYSICAL with freeze on) refuses, on
   `sml_doc_datetime`.

## 4. Post

One `$transaction`:

1. Lock the header (DRAFT, not deleted).
2. Validate (§3).
3. `StockPostingService.post(tx, source)`. The rules record gets
   `postShape: 'SIMPLE'` and a **per-line direction** resolver (§0) in place of the
   document-wide `isInward`.
4. §1.2 header recompute from the ledger.
5. §5 accounts voucher.
6. Header POSTED.
7. `txn_status_log` DRAFT→POSTED (`STOCK_ADJUSTMENT`).

Cancel mirrors all of that: reversal rows, recompute, `reverseLegs`, CANCELLED.

## 5. Accounts legs (PERPETUAL only; PERIODIC posts none)

The amount is the ledger rows' `sml_cost_value`, **never** the payload. The ledger is the line's
reason's `srm_gl_ledger_id`, else the header reason's, else the role below through
`acc_ledger_map`.

| kind | legs | default role |
|---|---|---|
| ADJUSTMENT, out | DR reason ledger / CR `INVENTORY` | `STOCK_SHORTAGE` |
| ADJUSTMENT, in | DR `INVENTORY` / CR reason ledger | `STOCK_EXCESS` |
| ISSUE (internal / sample / gift) | DR reason ledger / CR `INVENTORY` | `STOCK_SHORTAGE` (set the SAMPLE and GIFT reasons' ledgers to the real expense, e.g. Advertisement / Sales Promotion) |
| DAMAGE | DR reason ledger / CR `INVENTORY` | `STOCK_SHORTAGE` |
| EXPIRY_WRITEOFF | DR reason ledger / CR `INVENTORY` | `STOCK_SHORTAGE` |
| Re-lot pair | **none**: same value out and in | — |
| a move into the DAMAGED bucket (no write-off) | **none**: still the company's stock | — |

One voucher per document. Legs net per ledger, so a sheet with ten shortage lines is one DR
line.

**Voucher types:** add `StkAdj` (Stock Adjustment) to `accounts.acc_voucher_types`, category
BOTH, nature `STOCK_JOURNAL`, `affects_accounts` and `affects_inventory` true, own numbering
(`SADJ`), print title "STOCK JOURNAL". The five kinds share it. The stock voucher's own refno
prefix is per kind: ADJ / ISS / DMG / EXP.

## 6. Reason master maintenance (the screen was missing)

`GET /stock/reasons/list` (16q **Q18**, with `is_overridden` + `usage_count` + `can_delete`),
`GET /stock/reasons/get?srmId` (Q19), `GET /stock/reasons/usage?srmId` (Q20),
`POST /stock/reasons` (create/update), `POST /stock/reasons/deactivate`.

The three write rules the schema cannot enforce (16q, after Q20):
1. a company may not edit or delete a SHARED row (`srm_company_id IS NULL`); to change one it
   creates its own row with the same code, which then hides the shared one;
2. `srm_code` is immutable once any ledger row cites the reason;
3. delete means deactivate (`srm_is_active = false`); `srm_is_deleted` only for a reason never
   used.

The form also carries `srm_gl_ledger_id` (ledger picker), so a reason's expense ledger is set
where the reason lives.

**Seed additions**, as data in a migration: `TRANSIT_LOSS` (OUT, for the transfer settle-short
in the engine plan §1.1). Leave the 16 live reasons as they are.

## 7. Tests (e2e, beside `stock-engine-ts.e2e-spec.ts`)

- ADJUSTMENT with one + and one − line: two ledger rows, average moves only on the inward, one
  accounts voucher with an EXCESS pair and a SHORTAGE pair.
- BOTH reason with a signed qty; an IN reason with a negative qty → refused.
- `srm_require_remarks` without remarks → refused.
- DAMAGE beyond available → refused (forced BLOCK) even for an ALLOW item.
- EXPIRY_WRITEOFF on an unexpired lot → refused.
- Re-lot: the pair moves qty, value is carried, no accounts voucher, the old lot closes at 0.
- A lotless outward line on a FEFO batch item: split across the two earliest-expiry lots
  (`svi_split_no` 1, 2).
- Cancel: mirror rows + `Rev` voucher, header totals back to 0.
- §5.1 balance assertion = 0 after each.

## 8. Open decisions

| # | question | default |
|---|---|---|
| D-A1 | force BLOCK on adjustment outward whatever the item policy? | yes |
| D-A2 | expiry grace days as a company setting? | yes, `stock.expiry_writeoff_grace_days`, default 0 |
| D-A3 | should DAMAGE move to the DAMAGED bucket (kept, returnable to supplier) or write off? | two kinds: "Damage → write off" (DAMAGE) and "Move to damaged" (a BUCKET_OUT/BUCKET_IN pair, no accounts leg). The second needs the bucket-move txn types to be allowed on `ADJUSTMENT`; confirm before building |
| D-A4 | one menu for all five, or separate menu ids for rights (e.g. only managers write off)? | one screen, one menu. Rights per kind later through the register's per-type pattern if needed |
| D-A5 | menu id + `ui_table` id + grid id | assigned by the box at registration |
