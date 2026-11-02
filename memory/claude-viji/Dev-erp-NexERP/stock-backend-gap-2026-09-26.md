---
name: stock-backend-gap-2026-09-26
description: "The 2026-09-26 audit of NestJS stock vs the plan set — what is built (opening, physical, sales shadow posting), what is broken (transfers call absent SQL fns), what is absent; plans WRITTEN 2026-09-26"
metadata:
  node_type: memory
  type: project
  originSessionId: 0dbdc790-b14e-4a2b-8980-56a32216f95e
  modified: 2026-09-26T12:52:08.740Z
---

Audit of 2026-09-26 (nexerp-api-ro + live .106, read-only). The user asked "move all SQL-file functions into backend… write after my confirmation". NOTHING has been written yet.

**BUILT in TS** (`stocks/stock-voucher/stock-voucher-posting.helper.ts`, the only writer, via `StockPostingService`):
- OPENING and PHYSICAL (freeze on sml_doc_datetime).
- Lot resolve, and policy precedence item > group > company.
- Balances, the WAVG item cost with restamp, and the negative-stock policy.
- Cancel as mirror reversals.
- Sales bill/DC/return/DC-return post through a shadow ISSUE/RECEIPT voucher (`sales/posting/sales-stock.service.ts`).
- Sale-order reservations (FEFO).
- Tests: `test/stock-engine-ts.e2e-spec.ts` and the single-writer spec.

**BROKEN:** transfer despatch/receive/cancel still call `stock.fn_svh_post_transfer`, `fn_svh_receive_transfer` and `fn_svh_cancel`, none of which exist (controller `postFunction`). stock_transit has 0 rows.

**ABSENT:**
- ADJUSTMENT/DAMAGE/EXPIRY/ISSUE/RELOT screens and rules; REVALUE; REPACK (needed by D55).
- fn_svh_recompute: header totals come from the payload.
- sbl_reserved_qty / sbl_transit_in_qty refresh: reserved is 0 while 28 reservations are OPEN.
- Issue strategy FIFO/FEFO in posting.
- LOT_COST/MRP rate_source (falls to 0).
- Rebuild/assertion job, stock-as-of, ageing, valuation snapshot, demand daily, forecast.
- stock_mrp_price, not deployed: the price-bulk gateway returns 503.
- Reason and ageing-slab CRUD.
- Purchase module entirely.

**Also:**
- The legacy inventory.item_stock_* tables are read (item-stock-balance routes, `/master-lookup/item-price` stock qty, the reconciliation queue) but never written, so those reads return 0.
- The sales shadow numbering has no advisory lock.
- BUG_2026-09-09 IS FIXED in code (`stock-voucher.service.ts:1736` uses `unreversedLedgerRow()`); the bug file was simply never updated.

Related: [[stock-engine-not-deployed]], [[sql-vs-nestjs-rule]], [[share-truncation-2026-09-24]], [[plans-separate-file]].

**Stock → accounts gap (the user raised it on 2026-09-26):** stock vouchers write NO `acc_voucher_header` row. That was deliberate (plan-nestjs-opening-stock.md:200-215). The header is the Dr/Cr book, not a register of every transaction: Quotation and Sales Order have 0 rows, and only types with `vchr_affects_accounts`=t appear.

But `accounts.cogs_mode` = PERPETUAL, and bill/DC/return already post DR COGS / CR INVENTORY (`sales-leg.sources.ts` `buildCogsLegs`). Opening stock and physical excess never DEBIT INVENTORY, so the Stock-in-Hand ledger drifts negative and never equals the stock valuation.
- The link needs no new column: use `avh_src_module='STOCK'` + `avh_src_doc_type` + `avh_src_doc_id`, like sales does.
- Voucher types 1 OPENING and 6 Phy exist with `affects_accounts`=f.
- Roles INVENTORY / COGS / OPENING_DIFFERENCE exist. There is no shortage/excess/damage role.

**Day-end "what is still pending" = `public.txn_status_log`, NOT acc_voucher_header** (the user's real reason for asking, 2026-09-26). The status log is already the one cross-module table: every document's lifecycle, stock included (doc type STOCK_ADJUSTMENT for opening/physical). Pending = the latest event per `tsl_src_doc_id` whose status is DRAFT/HELD/CONFIRMED.
- ORDER BY `tsl_seq_no`, NEVER `tsl_changed_on`: create+post stamps CREATED about 50 ms AFTER POSTED. With seq ordering, the stock log matched `stock_voucher` exactly (3 drafts).
- Gaps found:
  - Sale-order DRAFTs are never logged (41 in the table, 0 in the log).
  - Some doc types are logged as a vague 'OTHER'.
  - The sales shadow stock vouchers are not logged, which is correct.

**PLANS WRITTEN 2026-09-26 (user confirmed):**
- `schema/stock/plan-nestjs-stock-engine-port.md`, phases 1-6 + Appendix A (every former SQL object mapped to its NestJS home) + decisions D1-D6.
- `schema/stock/plan-nestjs-stock-adjustments.md` (phase 2): one module and one screen for ADJUSTMENT/ISSUE/DAMAGE/EXPIRY/RELOT, per-line direction, reason-ledger legs, reason maintenance, decisions D-A1..5.
- The spec bodies are in `schema/stock/spec_prestrip/`.

User decisions of that day:
- One company-level Stock-in-Hand ledger (they said "really I don't know" and accepted the recommendation).
- The reason's `srm_gl_ledger_id` names the ledger, falling back to the STOCK_SHORTAGE/STOCK_EXCESS roles.
- Opening stock posts a voucher per document.
- The adjustment plan goes in a separate file.
- 17_printing is already built, so its partial recovery was not restored.

**PHASE 1 + ADJUSTMENTS LIVE (verified 2026-09-28):** migrations stock_engine_phase1 (11:12) + stock_adjustments (12:17). Routes: /stock/transfer/{validate,despatch,receive,receive/post,receive/prefill,receive/inbound,receive/settle-short,cancel}, /stock/adjustment/{validate,post,cancel,pick-stock}, /stock/reasons/{list,get,usage,deactivate}, /stock/admin/{balance-assertion (GET, detect only), post-missing-vouchers (POST)}, /txn-status/pending (companyId+accYear required). DB: stt_cost_rate_wot ✓, reserved reconciles (65=65), STOCK_SHORTAGE/EXCESS roles mapped, vtypes 1/6 affects_accounts=true, 54 stock docs carry an accounts voucher, ck_tsl has DC_RETURN/CHEQUE_*, TRANSIT_LOSS reason, srm_gl_ledger_id, still only fn_create_stock_partitions. Source: old SQL fn calls gone, recompute/issue-strategy/LOT_COST in helper, MAX+1 only in the locked allocator, legacy tables only in comments. NOT built (out of phase 1-2): stock_mrp_price (phase 3), reports (4), cron jobs 5.2-5.5. Transfers untested live (0 transit rows).
Balance assertion 2026-09-28: 1 finding ITEM_COST_QTY item 019fa296-… stored -100.278 vs ledger -101.278; the item moved ONLY by SALE (25 rows, last 16:20) — a lead on the sales shadow → stock_item_cost path, not yet a confirmed defect.
**Adjustment mockup DONE 2026-09-28:** share/schema/stock/stock_adjustment_ui_mockup.{py,png} — one screen, Type selector (Adjustment/Issue/Damage/Expiry write-off/Re-lot), one godown, per-line reason → Dir chip, outward cost read-only (branch avg), Pick-from-stock popup (/pick-stock) or blank = FEFO, re-lot pair panel, client-side accounts ESTIMATE card (validate returns no legs), User 2026-09-28: accounts card = screen ESTIMATE (ok); D-A3 MOVE TO DAMAGED = BUILD (so the shop sees what returns to which supplier) → sixth type "Move stock" (bucket→bucket, same lot, no accounts) + damaged list grouped by the lot's supplier; asked in share/schema/stock/notes-60-move-to-damaged.md (BUCKET_MOVE voucherType, line toBucket, MOVE_DAMAGED/MOVE_SALEABLE reasons, supplier on pick-stock, confirm sales never pick DAMAGED); ONE menu for all kinds (rights split later). Server still owes list grid (D-A5) + menu id.
