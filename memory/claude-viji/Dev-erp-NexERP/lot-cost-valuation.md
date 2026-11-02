---
name: lot-cost-valuation
description: User rule 2026-10-06 — plain stock on the item WAVG, every tracked item at its own lot cost; notes 92 written for Prathap, server not built yet
metadata:
  type: project
---

**User rule (2026-10-06):** "only for plain stock keep total average, otherwise all [tracked items at their own batch cost]". This applies to physical stock, adjustment, sale COGS, transfers and reports alike.

**Today (server):**
- `postingCte` relieves EVERY outward line and count shortage at `sic_avg_cost_rate` (item × branch).
- `applyItemCost` stamps that average onto every holding, so the count sheet, pick-stock and batch stock all show it.
- All presets and policies are `WAVG`. `LOT_ACTUAL` is allowed by the CHECK but no code reads `valuation_method`.
- `slt_cost_rate` = the FIRST receipt only (stale).

**Asked in** `share/schema/stock/notes-92-lot-cost-valuation.md`:
- `applyLotCost()` (Rev 3: writes the lot's branch rate onto its `stock_balance` rows; no new table);
- LOT_ACTUAL branches in `priced`;
- outward value = actual `sml_cost_value` in `applyItemCost`;
- a per-method stamp;
- the item-stock-balance `MAX(avg)` → Σ value ÷ Σ qty;
- price-bulk `mrp_cost` (notes 75) moved onto `stock_balance` rates;
- a backfill that reports, never posts.

**Client:** no number changes are needed, because the screens show what arrives. After the server is built, reword Stock Adjustment's "valued at the branch average" texts (`stock_adjustment_entry.cpp` ~619/621), then run the notes-92 §5 acceptance on MRP STOCK ITEM (expected 25 / 250 / 400 / 500).

Related: [[stock-posting-engine]], [[stock-adjustment-screen-built]], [[physical-stock-live-check]].

**Rev 2 (2026-10-06, user: "override 92, I didn't run it"), edited IN PLACE at the user's say-so. The original is in that session's scratchpad.** Checked against branch-wise offline → online sync:
- every cost cache is a REBUILD from the ledger (value = Σ signed `sml_cost_value` + Σ `stock_cost_revaluation`), never "old + delta";
- the backfill goes through the new append-only `stock_cost_revaluation` table, because `ck_sml_qty_nonzero` rules out a 0-qty ledger fix;
- §7: the centre never re-costs a synced document; `applyBalances` must become a rebuild; deterministic uuid-v5 `slt_id` (otherwise two offline branches minting the same batch collide on `ux_slt_identity`); `slt_total_on_hand` / status rebuilt at the centre; `refreshTransitIn` also runs on the receiver;
- `refreshReserved` / `refreshTransitIn` / lot on-hand were already rebuilds.

**Rev 3 (same day, user: "should we definitely add new table" → no): NO new tables.**
- A lot's cost per (company, branch, lot) is stamped on every `stock_balance` row of that lot in the branch.
- Posting reads it with a lateral `MAX(sbl_avg_cost_rate)`.
- `stock_cost_revaluation` is only conditional: create it only if the backfill's step 0 finds tracked lots that already had an outward. The box had 0 on 2026-10-06.
- Rev 2 backup is in the scratchpad too.
- Neither table was ever created live.

**Prathap BUILT it 2026-10-06 (snapshot 11:30); server DEPLOYED; migration NOT applied.**
- Code reviewed against Rev 3 and matches:
  - `applyLotCost`, `rebuildLotCosts`, `rebuildItemCosts`, `stampItemAverage` (WAVG only), `rebuildBalances` (assigned, not added), `rebuildLotTotals`;
  - deterministic uuid-v5 `lotIdentityUuid` (pgcrypto `digest` in public; the expression was verified in psql);
  - `ck_spt/stp_valuation_tracks` CHECKs; the item-stock-balance Σ value ÷ Σ qty; price-bucket on `sbl`.
  - Re-lot IN = qty-weighted OUT lot rates.
- `POST /stock/admin/rebuild-costs?dryRun=true` (rolls back). Run while still all-WAVG, it gave:
  - 0 balance-qty mismatches (157 holdings); 92 balance rows rewritten = dates / batch-cache columns only;
  - 2 item totals moved = old incremental drift that the rebuild corrects: item "001" qty −106.945 → −107.945 (ledger and balances agree on −107.945); ZT-ITEM-A value 39,183.78 → 39,769.74;
  - assertion [].
- `_prisma_migrations` lacks `20261006100000_lot_actual_costing_notes_92`; presets are still WAVG.
- Next: Prathap runs `prisma migrate deploy` → dry run → real run. Then I run the notes-92 §5 acceptance (expect 25 / 250 / 400 / 500).

**LIVE 2026-10-06 17:14:** migration applied and rebuild run. The count sheet shows 25 / 250 / 400 / 500 (acceptance §5.1 PASS); MRP ITEM 252.63. §5.2-7 (posting tests) are not run yet.

**Found while checking → notes 93** (`share/schema/stock/notes-93-outward-phantom-lot.md`):
- bil00964 / bil00967 sold MRP ITEM with `sbiMaxPrice` 0 from a godown with no stock. The engine opened a phantom "MRP 0" lot (−103).
- Server: `lotlessOutwardLine` treats only NULL as "not stated", and `resolveLots` mints lots for non-REASON outward lines.
- Client FIXED: the sale bill + DC send MaxPrice null when the MRP is ≤ 0.
- Sale bill gap: it never reads `/item-price` `buckets` (MRP price rows), so an MRP item bills at the headline price (0 for MRP ITEM). Asked the user how the cashier picks the MRP.

**Notes 93 LIVE-VERIFIED 2026-10-06** with `tests/salebill/run-full.sh mrp93` (new mode in test_full.cpp), 14/14:
- T1: MRP STOCK ITEM @ MRP 550 from Coimbatore. Posted bil00975, issued from KL8525 at 1 @ 500.00 (its own cost), COGS leg 500. Cancelled and restored (20 on hand; sic 55 / 18,750).
- T2: the same from namakkal (the sale bill's DEFAULT godown, which holds none) → refused: "names no lot and this godown holds no stock … matching MRP 550". Its draft bil00976 is already is_deleted.
- T3: 0 tracked lots opened by SALES.
- Not exercised: the server's literal MRP 0 (the client now sends null) and a plain item sold before its receipt.
- Notes 92 §5 still open: shortage, write-off, return, transfer.
- Observation: the sale bill has no godown selector, and the lookup defaults to namakkal for everything.
