# 3.0 `public.fsales` — every behaviour, and where it lands in the plan

**2026-09-20, on the user's ask ("check fsales function in 3.0 db is everything
landed in our plan").** `fsales` is 2,306 lines, 59 flags. Flag 1 is the
insert/edit; 5 delete; 6/24 hold; 38/48 tender-later; the rest are lookups
the screen calls. Read from the `3.0` database; the challan is excluded by
rule (nothing here is about `fx_sale_chln_entry`).

Legend: ✅ landed · ➕ landed now (this audit added it) · ⏭ deferred, named ·
✖ deliberately not carried.

## 1. Flag 1 — insert / edit

| fsales does | ours |
|---|---|
| config 20 `deltaken` (delivery taken by default) | ✅ `sb_delivery_status` NA/PENDING by `sales.delivery_status_tracking` |
| config 19 `closeorder` (close a mobile order on partial bill) | ⏭ mobile/web orders — see §4 |
| config 32 `mastercost` (cost from the item master, not the lot) | ✅ cost is the lot's (`stock_item_cost` policy in the engine); a "master cost" valuation is the engine's policy row, not a bill setting |
| config 83 `allgdnstk` (sell from any godown's stock) | ➕ `sales.stock_from_any_godown` (BRANCH, default false): the engine's `resolveLots` may take FIFO across the branch's godowns when the line's godown is short — D53 |
| `otherstate` = company state ≠ bill state → IGST | ✅ `sb_pos_stcd` vs company state → `gdr_supply_nature` (place-of-supply split) |
| delete the hold row on save | ✅ txn_hold consume |
| numbering from `sales_categories`: User Wise / Reset Daily / Monthly, `[Bill No]` `[YY-YY]` format, padding | ✅ `acc_voucher_seq` (prefix, width, reset NEVER/DAILY/MONTHLY/YEARLY), POS per device / WHOLESALE per branch; ✖ a per-USER series — the device series is the counter's identity (Q46) |
| `customers.billed_date` stamp | ✖ derived (last bill per party is a query) |
| config 108 "postpone account entries" (bill saved Pending, posted when Delivered) | ✅ the DRAFT → POST model itself; `sales.auto_post` off = post later |
| edit path: audit snapshot, un-apply SO / mobile / quotation, reverse stock, delete every accounting row, re-insert | ✅ `/bills/amend` = cancel-work + post-work, `audit.audit_log`, revision + 1 — never delete |
| tender loop: LOYALTY (type 6) → points row; RRN (type 5) → return credit applied via `acc_pendings`; `acc_sess_tenders` (per-session tender rows); `acc_tender_detail` | ✅ REDEEM rows via `LoyaltyLedgerService.consume()`; RRN retired → `NOTE_ADJUST`; session tender totals = `acc_tender_detail` (`td_session_id`, `td_device_id`) — the POS session module is separate |
| `sales` row: `vehicle_tripno` (trip no per vehicle per day, flag 23) | ⏭ dispatch / load-sheet module (`fsales_dispatch`, `fx_godown_dispatch`) — P6, not a bill column |
| `disc_brands` (brand-wise discount list on the bill) | ✅ promotions engine (brand-scoped scheme) |
| `print_count[3]`, `del_chit_no`, `bill_del`, `del_taken`, `del_date` | ✅ `sb_print_count` (one counter), `sb_delivery_status` + events |
| `sales_valid` (a checker verifies the bill before dispatch — flag 29, the "sales validation screen") | ➕ delivery event `VERIFIED` between PENDING and PACKED + `sales.require_verification_before_dispatch` (BRANCH, default false) — D54 |
| `dispatch_id`, `packed_by[]`, `supervisor_id[]` | ✅ `sb_packed_id`, `sb_supervisor_id` exist; dispatch id → P6 |
| `agent_id` + `agent_comm_amt`, `sal_sup_type`, `addl_disc1` | ✅ agent + commission amount on the bill; supply type = register; addl_disc1 dropped (user's trim) |
| `sales_tranlog` (the GST outbox row) | ✅ `acc_voucher_doc_register` |
| stock: FIFO by godown + stock_date, batch by MRP (1) / batch no (2) / any; `stocks` row created when none | ✅ `resolveLots` (track signature: batch / MRP / expiry / serial), lot created on inward only — a sale never invents a lot |
| **`fsplit_stock` auto unit-break** (`item_master.item_auto_break`): selling 3 pcs when only a 12-pack exists → break one pack, then sell | ➕ engine task: when `item_auto_break` and the line's unit is short, `resolveLots` writes a REPACK_OUT / REPACK_IN pair for the parent pack before the SALE row (the REPACK the engine still owes) — D55 |
| negative-stock check (commented out; per-item `NegStock` flag) | ✅ `comp_negstk_apl` + the engine's policy |
| `voucher_taxlog` (per-line tax snapshot) | ✅ `acc_voucher_doc_detail` per item line |
| `sale_bill_items` (flat per-customer item history for "last rates") | ✅ `sale_bill_item` is the source; the Last Rates quick button is a grid over it |
| `delivery` (the stock movement rows) | ✅ `stock_ledger` |
| SO: `item_del_qty`, `so_completed`, `billed_date`; quotation `quote_completed`; mobile order completed | ✅ fulfilment service; quotation CONVERTED; mobile → §4 |
| `fx_agent_loyalty_points` (incentive points for the sale AGENT / broker) | ⏭ agent incentive scheme — the `sale_agents` module has no points table today; P6 with commission |
| **loyalty earn, 4 scheme kinds**: Item Master PV, Invoice Amount (slab factor), Invoice Item Amount (brand / group / item), Invoice Item Qty; only items with `allow_loyalty` | ✅ 12's `lsc_apply_on` BILL_AMOUNT / BILL_QTY / ITEM_AMOUNT / ITEM_QTY + slabs + item scopes; ➕ `item_allow_loyalty = false` excludes the line (spec §5.8) |
| `fsalesman_commision` (config 40) | ⏭ commission rows per bill — P6 with the agent module; the bill keeps `sb_salesman_id[]` and `sb_agent_comm_*` |
| e-invoice applicability (Regular + Tax Invoice + company flag/date + config 73), e-way (inter/intra limits) | ✅ `DocRegisterService` + statutory rows; "taxable bill" rule from `feinvoice_system` flag 21 |
| SMS auto-send on post (`sms_settings.auto_sms`) | ✅ after-commit hook (print / WhatsApp / SMS jobs) |
| Dr/Cr verification ±0.50 | ✅ exact balance, refused before write |
| **stock data verification** (Σ sale qty of non-service lines = Σ delivery qty) | ➕ `SalesPostingService` asserts Σ `sml_base_qty` = Σ non-service line base qty after the engine returns — a test and a guard |

## 2. Flags 5 / 6 / 24 — delete, hold

| fsales | ours |
|---|---|
| delete: reverse stock, un-apply SO / mobile / quotation, delete all accounting, `deleted_transactions` row, status false, `faudit_log` | ✅ `/bills/cancel`: reversal rows everywhere, status CANCELLED, `txn_status_log` + audit — never delete; `/bills/delete` DRAFT only |
| hold: a `sales` row with `sales_held`, `Hld/n` ref, items from XML; delete-hold | ✅ `txn_hold` (snapshot JSON, grid 96) — a held bill never touches `sale_bill` |

## 3. Lookup flags — each has a home

| flag | fsales | ours |
|---|---|---|
| 7 | customer detail (credit, points, area, type) | ✅ `/bills/party-context` |
| 8 | freight detail by distance | ✅ `fetchFreightRates` (exists) |
| 9, 55, 56 | item detail; re-rate on price-level change (batch / non-batch) | ✅ `fetchItemPrice` + price level (exists) |
| 10, 40 | barcode / search code | ✅ + weight barcodes (Q48) |
| 11, 12 | promotion schemes / offers | ✅ promotions engine (client) + `promotion_usage` |
| 15, 16, 17, 31 | SO list / detail / cancel / SO→invoice (many SOs) | ✅ `/bills/open-sources`, multi-order append, `/sale-orders/cancel` |
| 19 | existence of payment / live IRN / EWB → edit blocked | ✅ `/bills/get` `locks` |
| 20, 21 | quotation import | ✅ exists |
| 22 | has return → locked | ✅ |
| 25, 26 | tender types / detail | ✅ tender module |
| 27 | return redemption list (credit notes of the party) | ✅ F4 adjust panel (grid 123) |
| 28 | loyalty redemption info (points, gift points, ₹/pt) | ✅ `party-context` loyalty block |
| 29 | sales verification flag | ➕ VERIFIED event (D54) |
| 30, 33, 47, 57, 58 | delivery status / delivered / packed-by / supervisor / status text | ✅ `/bills/delivery-status` events (menu 226) |
| 32 | customer by mobile | ✅ customer dropdown by mobile; `?mobile=` on temp credits |
| 34 | TCS master | ✅ statutory rows (TCS infra kept, rate resolves to 0 today) |
| 35 | **copy bill to another company** | ⏭ Copy - Alt+Y is same-company today; cross-company copy = later (it is a new bill in the other company's series with re-priced lines — a feature, not a flag) |
| 36 | **qty-based rate** (`item_qty_rates`: rate by qty slab per price level) | ⏭ not in 16 §20 pricing; listed for the pricing pass (D56: `inventory.item_qty_rates` — slab from/to qty × price level → rate) |
| 37, 38 | select bills by ref + **bulk tender entry** (settle many delivered credit bills at once) | ✅ that is a RECEIPT (menu 99) against the bills; the bulk-tender screen is retired |
| 39 | compliment gift | ⏭ 13/14 gift redeem — later (schema exists) |
| 42, 46 | **contract items** (customer-specific rates) | ⏭ **skipped by the user (2026-09-20)** — `sales.cust_item_rates` exists live; wiring it into the price lookup is a later pass, not part of the sales-txn build |
| 43, 44, 50 | e-invoice detail / update / tranlog detail | ✅ registers + `/gst/*/preview` |
| 45, 49 | customer balance detail; **credit limit based on customer TYPE** (config 70) | ✅ per-customer limits (`cus_credit_*`); ⏭ group-level limit (a `cust_groups` column) — later |
| 48 | tender entry later, from the main window (session-by-bill config 79) | ✅ DRAFT then Settle (F5) later; a POSTED credit bill → receipt |
| 51, 52, 53, 54 | last counter bill; voucher by bill no; negative stock list; employees | ✅ Last Bill F11; search; stock report; dropdowns |
| 59 | **update remarks on a posted bill** | ➕ `PUT /bills/update-remarks` `{keys, remarks, editRemark}` — the receipt's `/update-header` shape: text only, no money, logged |

## 4. Deferred, named (so they are not lost)

* **Mobile / web orders** (`web_orders`, flags 13, 14, 18, config 19):
  **skipped by the user (2026-09-20)**. When wanted: a `sale_order` with
  `so_src_doc_type 'MOBILE'` and `so_delivery_mode` — the columns exist.
* **Dispatch / load sheet / vehicle trip** (`fsales_dispatch`,
  `fx_godown_dispatch`, `vehicle_tripno`, `dispatch_id`): P6.
* **Agent incentive points + salesman commission rows**
  (`fx_agent_loyalty_points`, `fsalesman_commision`): P6 with the agent
  module.
* **Qty-slab rates** (`item_qty_rates`) and **contract rates**
  (`cust_item_rates`): the pricing pass (D56; D58 withdrawn).
* **Cross-company copy** (flag 35), **group-level credit limit** (config
  70), **gift redemption** (flag 39): later.

## 5. Decisions taken by this audit

* **D53** `sales.stock_from_any_godown` (BRANCH, false): FIFO across the
  branch's godowns when the line's godown is short.
* **D54** delivery event `VERIFIED` + `sales.require_verification_before_dispatch`
  (BRANCH, false) — the 3.0 validation screen, as an event.
* **D55** auto unit-break at sale (`item_auto_break`) = an engine REPACK
  pair before the SALE row — the REPACK the engine still owes (P1).
* **D56** qty-slab rates go to the pricing pass; not a sales-txn column.
* **D57** `PUT /bills/update-remarks` — text only on a POSTED bill.
* **D58** withdrawn — contract rates (`cust_item_rates`) skipped by the
  user; later pass.
* **D59** loyalty: one scheme per bill by priority; `item_allow_loyalty`
  excludes a line; walk-in earns nothing unless enrolled through a customer
  row (`sales.loyalty_auto_enrol`, default true, enrols a customer on the
  first earning bill).
