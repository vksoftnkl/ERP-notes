# Sales transactions — the posting flow, document by document, row by row

**2026-09-20, all defaults.** Six documents — Quotation, Sales Order, Sale
Bill, Sale Return, Delivery Challan (DC), DC Return — and what each one
WRITES when it posts, cancels and amends. Read against the live
192.168.0.106/ERP on 2026-09-20 (every column named below exists there or
is added by `../29_sales_documents.sql` / `../30_sales_alters.sql`), the
3.0 function bodies for the bill / return / loyalty / scheme / e-invoice
(never the challan — by rule), TallyPrime and GoFrugal for the behaviours,
and the GST / income-tax rules in force on the date, resolved PER COMPANY.

The analysis that led here is `analysis_2026-09-20.md`; the fifty decisions
are its §12, every one taken as the stated default. The picture is
`sales_txn_plan.png`.

**The whole thing in one line:** a sale is goods leaving (stock rows and,
perpetually, DR COGS / CR Stock in Hand at cost) and money owed (DR Party /
CR Sales + output GST, then the tenders against the party); the two halves
are posted by whichever document actually does that thing — the challan
moves the goods, the bill raises the debt, a bill against a challan raises
only the debt — and a return or a DC return is the same rows with the sign
reversed, never a delete.

---

## 0. The tables

| table | partitioned | written by | role |
|---|---|---|---|
| `sales.sale_quotation(_item)` | yes | Quotation | no posting (Q22) |
| `sales.sale_order(_item)` | yes | Order | post = CONFIRMED; fulfilment caches |
| `sales.sale_dc(_item)` **29** | yes | DC, then the bill and the DC return write `sdi_billed_qty` / `sdi_returned_qty` | goods out, un-billed |
| `sales.sale_bill(_item)` + **30** | yes | Bill; the return writes `sb_returned_amt`, `sbi_return_qty` | the invoice |
| `sales.sale_return(_item)` **29** | yes | Return | the credit note |
| `sales.sale_dc_return(_item)` **29** | yes | DC Return | un-billed goods back |
| `public.txn_charge_detail` + **30** | yes | every document with charges | charge rows; `cd_src_cd_id` carries an order charge into its bills |
| `stock.stock_ledger` | yes | DC, direct bill, return, DC return | the movement; keyed by `sml_src_doc_id`, idempotent on `ux_sml_source` |
| `stock.stock_lot`, `stock_balance`, `stock_item_cost` | — | the stock engine | lots, on-hand, cost |
| `stock.stock_reservation` | yes | Order (CONFIRMED) | Q19: soft reservation, WARN policy |
| `accounts.acc_voucher_header` / `acc_vouchers` | yes | DC (COGS), bill, return, DC return (COGS) | the legs; every non-party leg carries `av_role` |
| `accounts.acc_tender_detail` | yes | bill, return | instruments |
| `accounts.acc_bill_balance` | yes | bill (SALES row), return (SALES_RETURN credit), order (ADVANCE — exists) | bill-wise |
| `accounts.acc_bill_adjustment` | yes | bill (advance set-off, credit-note set-off), return (ADJUST mode) | settlement rows, reversed by negative counter-rows |
| `accounts.acc_voucher_doc_register` + `_doc_detail` | no | DC, bill, return, DC return | the GST document; HSN/rate lines |
| `accounts.acc_voucher_doc_einvoice` / `_ewaybill` | no | the GSP worker | IRN / EWB — see `../gst/` |
| `sales.loyalty_member` / `loyalty_ledger` **13** | yes | bill (EARN, REDEEM), return (REVERSE) | points |
| `sales.promotion_usage` **15§6** | yes | bill, return (claw-back) | applied schemes |
| `public.txn_status_log` | yes | every document, every transition | the trail |
| `audit.audit_log` | — | amend | before/after of an amend (R20) |

**Line and header shapes.** `sale_dc_item`, `sale_return_item` and
`sale_dc_return_item` carry the ESSENTIALS of `sale_bill_item` under their
own prefix — item / special / scheme discount perc + amt, the charge shares
before / after tax, cess per unit, act / min price, price level, EAN, size,
salesman, scheme (the return line adds `is_free` and the bill-scheme share
it reverses); the headers carry the `has_*` flags, discount totals,
freight / load / unload, bags, calc types. Not carried: `addl_disc1/2`,
the discount `*_qty` columns, additional cess, `net_gross`, per-line
freight/load/unload, `sold_price` / profit, MRP savings, the bill's
pay / tender caches (29's header lists the reasons).

Server-owned columns the DTO never carries: `*_posted_voucher_id`,
`sb_doc_register_id`, `*_revision_no`, every `*_status`, every fulfilment
cache. The client sends the document; the server says what it became.

---

## 1. The two decisions, applied

**COGS is PERPETUAL** (`accounts.cogs_mode`, Q16). Cost is recognised when
the goods MOVE, at the lot cost the stock engine resolves (`sml_cost_value`):

| document | stock rows | COGS legs | revenue legs |
|---|---|---|---|
| Delivery Challan | `DC_ISSUE` − | DR COGS / CR INVENTORY | none |
| Bill, direct line | `SALE` − | DR COGS / CR INVENTORY | yes |
| Bill, line from a DC | none | none (posted at the DC) | yes |
| Sale Return | `SALE_RETURN` + | DR INVENTORY / CR COGS at the bill line's cost | reversed |
| Free return (no bill) | `SALE_RETURN` + at current cost | none | reversed at current price |
| DC Return | `DC_RETURN` + | DR INVENTORY / CR COGS at the DC line's cost | none |

PERIODIC (setting flipped) removes the COGS column only; nothing else
changes. Stock rows still carry `sml_cost_value`, so the closing-stock
journal has its figure.

**One `sale_bill`, `sb_bill_mode` POS | WHOLESALE** (Q15). The mode decides
the number series (POS = device, WHOLESALE = branch MAIN, Q46), the default
customer (POS = `sales.default_customer_id`, Q14), which screen opens it
and which columns the print shows. It never decides what is posted — a POS
bill and a wholesale bill write the same rows.

---

## 2. Lifecycle — the receipt pattern, on every document

```
DRAFT ──post──▶ POSTED ──cancel──▶ CANCELLED
  │                │
  │ delete         │ amend  (sales.allow_posted_amend, default TRUE)
  ▼                ▼
 gone         POSTED, revision + 1  (= cancel-without-the-status-change, then post again,
                                     same id, same number, in ONE transaction)
```

* **DRAFT** writes the document tables and nothing else. Delete is allowed
  ONLY here (Q11) and is a hard delete of a thing that touched nothing.
* **POST** is the one-way door: everything in §3–§8 in one transaction,
  status flipped LAST, `txn_status_log` POSTED row. Refusals in §9.
* **CANCEL** never deletes: stock reversal rows (`sml_is_reversal`,
  `sml_reverses_id`), a reversal voucher of the same type
  (`avh_reversal_voucher_id` on the original), negative counter-rows in
  `acc_bill_adjustment`, `abl_status` CLOSED with the bill amount as write-
  back, register row CANCELED, loyalty REVERSE, usage reversed, fulfilment
  caches restored on the source. Needs `um_can_cancel` and a reason
  (`ck_tsl_reason_required`).
* **AMEND** (R20 shape, Q9): the full create+post payload plus the four
  keys, `baseRevision` and `editRemark`. Server: cancel-work with the status
  left POSTED, then post-work with the new payload, `sb_revision_no + 1`,
  the before/after into `audit.audit_log`. Shares cancel's refusal list, and
  adds: refused if `baseRevision <> sb_revision_no`.
* The order has one extra state: POSTED for an order means **CONFIRMED**
  (Q22), and its terminal states COMPLETED / CANCELLED come from fulfilment
  and cancel. The quotation has no post at all — CONVERTED is written by the
  order or bill that consumes it.

Every transition is a `txn_status_log` row: `tsl_src_module 'SALES'`,
`tsl_src_doc_type` from the vocabulary in 30 §5, `tsl_event` = the verb,
from/to = the statuses. Delivery events (Q27) ride the same table:
`PACKED`, `DISPATCHED`, `DELIVERED` with `tsl_to_status` = the delivery
status and `sb_delivery_status` as the cache.

---

## 3. The Sales Order

**Post = CONFIRMED.** Writes:

1. `stock.stock_reservation` one row per item line (`srv_src_doc_type
   'SALES_ORDER'`, `srv_reserved_qty` = line base qty, `srv_expires_on` =
   `soi_reserve_expires_on` or `so_delivery_date`); `soi_is_reserved` true.
   Policy WARN (Q19): a shortfall is reported in the response
   (`warnings[]`), never a refusal — an order is a promise, not a movement.
2. Advance: the existing order-advance posting (`ARc`, complete today) is
   unchanged. It writes the ADVANCE bill-balance row the bill later sets
   off (§5.6).
3. `txn_status_log` CONFIRMED.

**Fulfilment** is written by the consumers: a DC or a bill line with
`sbi_src_doc_type 'SALES_ORDER'` and `sbi_src_item_id = soi_id` adds to
`soi_delivered_qty` (generated `soi_pending_qty` follows), consumes the
reservation (`srv_consumed_qty`), and the header's `so_fulfil_status`
becomes PARTIAL / COMPLETED. Over-delivery is refused unless
`sales.allow_bill_over_order_qty`.

**Cancel** releases every open reservation (`srv_status` RELEASED) and
refuses if any line has `soi_delivered_qty > 0` — cancel the bills first,
or use the per-line cancel (`soi_cancelled_qty`, exists) for the rest.

**Amend** is allowed while `so_fulfil_status = 'OPEN'`; after a partial
delivery only the undelivered lines may change (line-level check).

**Many orders → one bill** (Q26): the bill screen appends lines from any
CONFIRMED order of the same customer; each line keeps its own
`sbi_src_doc_id`; the header's `sb_src_doc_id` is the FIRST order; every
order touched is updated line by line. Charges: §7.

---

## 4. The Delivery Challan

Goods out, no debt. **Post** writes, in this order:

1. **Guards** (§9), then the DC-specific ones: purpose in
   `comp_dc_purposes`; `sales.dc_requires_order`; a line from an order may
   not exceed `soi_pending_qty`.
2. **Stock** — `StockPostingService.post(SALES, 'DELIVERY_CHALLAN', sdc_id)`:
   one `stock_ledger` row per line, `sml_txn_type 'DC_ISSUE'`,
   `sml_direction -1`, `sml_bucket = sdi_bucket`, lot resolved by the
   engine's `resolveLots` (FIFO / explicit `sdi_lot_id`), then
   `attachLotsToLines` writes `sdi_lot_id` and `sdi_cost_price` BACK onto
   the line (the bill inherits the lot; the DC return goes back to it).
   Negative stock: `comp_negstk_apl` decides, as today.
3. **COGS** (PERPETUAL): one voucher, type `DCh`, `avh_src_doc_type
   'DELIVERY_CHALLAN'`, two legs per tax-rate group or one pair for the
   whole document (one pair — cost has no rate):
   `DR COGS Σ sml_cost_value · CR INVENTORY Σ sml_cost_value`, both with
   `av_role`. `sdc_total_cost` and `sdc_posted_voucher_id` stamped.
4. **Register** — one `acc_voucher_doc_register` row: `gdr_source_module
   'SALES'`, `gdr_tran_nature 'DELIVERY_CHALLAN'`, `gdr_doc_flow 'OUTWARD'`,
   `gdr_doc_sign 1`, `gdr_doc_type 'DELIVERY_CHALLAN'`, party / ship-to /
   dispatch blocks from the header, values from the priced lines,
   `gdr_is_einvoice_applicable false` (never an IRN, Q8),
   `gdr_is_ewaybill_applicable` = §8.2's rule. `_doc_detail` rows per HSN
   + rate. Q32: purpose maps to the e-way bill's *Supply type* sub-type
   (SUPPLY → Outward-Supply, JOB_WORK → Outward-Job Work, APPROVAL /
   EXHIBITION → Outward-Others, OWN_USE → Outward-For Own Use, LINE_SALES →
   Outward-Line Sales).
5. **E-way bill** — if applicable and `sales.eway_auto_fire`: a
   `gst_api_log` job row; the worker (`../gst/`) writes `_doc_ewaybill`.
6. Source order fulfilment (§3), `txn_status_log` POSTED, status.

**The DC is the bill's source** — `sbi_src_doc_type 'DELIVERY_CHALLAN'`,
`sbi_src_doc_id = sdc_id`, `sbi_src_item_id = sdi_id`, `sbi_lot_id =
sdi_lot_id`, `sbi_cogs_amt 0`. Q7: one bill may take lines from many DCs,
one DC may feed many bills, and direct lines may sit beside DC lines on the
same bill. A DC of purpose other than SUPPLY converts only when the
operator confirms the purpose change (logged).

**Cancel** — refused if any line has `sdi_billed_qty > 0` or
`sdi_returned_qty > 0` (rule: DC return locks the DC; a bill locks it too)
and if the e-way bill is live past `EWAY_CANCEL_HOURS` (cancel the EWB
first, or let it expire). Otherwise stock reversal rows, COGS reversal
voucher, register CANCELED, order fulfilment restored, reservation
re-opened.

**Amend** — same refusals; the whole payload, revision + 1.

**The clock** — `DC_RETURN_WINDOW_DAYS` (180, per company): a POSTED DC
with `sdi_open_qty > 0` older than that appears on the "open challans"
grid in red, WARN on the bill screen; nothing is refused (s.31(7) makes the
invoice due, it does not forbid it).

---

## 5. The Sale Bill

### 5.1 Order of work at POST

```
 1 guards                       §9 — company law, party, credit, dates, rights
 2 source lines                 DC lines: sdi_open_qty; order lines: soi_pending_qty
 3 stock                        direct lines only — SALE rows, lots attached
 4 legs                         one voucher, type Bil, §5.3
 5 COGS                         direct lines only, into the SAME voucher
 6 bill balance                 abl SALES row, §5.5
 7 tenders + set-offs           acc_tender_detail; advance / credit-note adjustments, §5.6
 8 register                     INVOICE or BILL_OF_SUPPLY + detail rows, §5.7
 9 loyalty                      EARN (+ REDEEM was a tender in 7), §5.8
10 promotion usage              one row per applied scheme, §5.9
11 charges                      order-charge carry rows closed out, §7
12 source caches                sdi_billed_qty / soi_delivered_qty / reservation consume
13 status log + status          POSTED last
14 after commit                 GSP jobs (IRN, EWB), print job, WhatsApp
```

### 5.2 Worked example — a wholesale credit bill

Bill B-101, 2026-09-20, customer Acme (registered, same state), two lines:

| line | source | qty | rate | taxable | GST 18% | line total | lot cost |
|---|---|---|---|---|---|---|---|
| 1 | direct | 10 | 100 | 1,000.00 | 180.00 | 1,180.00 | 620.00 |
| 2 | DC-7 | 5 | 200 | 1,000.00 | 180.00 | 1,180.00 | (posted at DC-7: 700.00) |

Freight charge (after tax, own ledger) 100.00. Bill-level cash discount
50.00. Round-off +0.00. Bill amount 2,410.00. Advance 500.00 held from
order SO-9. Loyalty: earns 24 points (1 per 100 of taxable).

**stock_ledger** — one row: line 1, `SALE`, −10, cost 620.00, lot from
FIFO. Line 2: nothing (DC-7 moved it).

**acc_vouchers** (type Bil, one header, `avh_src_doc_type 'SALE_BILL'`):

| row | Dr/Cr | ledger (by role) | amount | av_role |
|---|---|---|---|---|
| 1 | DR | Acme (party, `cus_id` IS `led_id`) | 2,410.00 | — |
| 2 | CR | Sales | 2,000.00 | SALES |
| 3 | CR | Output CGST | 180.00 | OUTPUT_CGST |
| 4 | CR | Output SGST | 180.00 | OUTPUT_SGST |
| 5 | CR | Freight Recovered (`cd_ledger_code`) | 100.00 | — (charge row carries the ledger) |
| 6 | DR | Discount Allowed | 50.00 | DISCOUNT_ALLOWED |
| 7 | DR | Cost of Goods Sold | 620.00 | COGS |
| 8 | CR | Stock in Hand | 620.00 | INVENTORY |
| 9 | DR | Advance from Customers | 500.00 | ADVANCE_RECEIVED |
| 10 | CR | Acme | 500.00 | — |

Debits 3,580.00 = credits 3,580.00 (`ck_avh_balanced`, totals by the live
trigger). Rows 9–10 are the advance set-off; a cash bill adds
`DR Cash / CR Acme` per tender instead. Round-off, when non-zero, is one
leg on ROUND_OFF with the sign that balances. `sb_cogs_amt 620`,
`sbi_cogs_amt` 620 / 0.

**acc_bill_balance** — one row: `abl_bill_type 'SALES'`, `abl_dr_cr 'DR'`,
`abl_bill_amount 2,410.00`, `abl_alloc_amount 500.00` (the advance),
`abl_pending_amount 1,910.00`, `abl_status 'PARTIAL'`, `abl_due_date` from
`sb_due_days`, `abl_src_doc_type 'SALE_BILL'`, `abl_voucher_id` = the Bil
voucher.

**acc_bill_adjustment** — one row: `ADVANCE_ADJUST`, `abj_bill_id` = the
new SALES row, `abj_against_bill_id` = SO-9's ADVANCE row, 500.00,
`abj_settlement_mode 'ADVANCE'`. The ADVANCE row's `abl_alloc_amount` +=
500 (its `abl_status` CLOSED when fully used). `so_advance_adjusted_amt`
+= 500.

**acc_voucher_doc_register** — one row: `INVOICE`, OUTWARD, sign 1,
`gdr_supply_nature 'INTRA_STATE'`, taxable 2,000, CGST 180, SGST 180,
other_charge 100, discount 50, bill value 2,410, `gdr_is_einvoice_applicable`
= company applicable ∧ date ≥ `comp_einvoice_date` ∧ party registered,
`gdr_is_ewaybill_applicable` = §8.2 (line 1's goods move now; line 2's
moved under DC-7's EWB — the bill's EWB covers line 1's value only when
the company ships the DC and the invoice separately; default: the bill
carries `sb_transport_*` and the EWB is for the whole consignment that
leaves WITH the invoice, i.e. direct lines). `_doc_detail`: two rows
(HSN × 18%). `sb_doc_register_id` stamped.

**loyalty_ledger** — one EARN row, 24 points, `lle_src_doc_type
'SALE_BILL'`, expiry lot per the scheme (13). `sb_loyalty_earned 24`.

**promotion_usage** — one row per scheme id that appears in
`sbi_scheme_id` / the header scheme.

**sale_dc_item** DC-7 line: `sdi_billed_qty` 5, `sdi_line_status` BILLED
(or PARTIAL); header `sdc_fulfil_status`. **sale_order_item** SO-9: as §3.

**txn_status_log**: `POSTED`, DRAFT → POSTED.

### 5.3 The legs, generally

| leg | when | Dr/Cr | ledger | av_role |
|---|---|---|---|---|
| party | always | DR | `sb_cust_id` (walk-in = the default customer's ledger, Q14; the printed name / address / GSTIN are the `sb_cust_*` snapshot, typed on the bill for a walk-in — the register row reads the snapshot, so a typed GSTIN makes it B2B) | — |
| sales | always | CR | role SALES (per `alm_supply_nature` if mapped) | SALES |
| output tax | per rate line | CR | OUTPUT_CGST / _SGST / _IGST / _CESS / _ACESS | those |
| charge | per `txn_charge_detail` row with `cd_sep_post` | CR (DR for a negative charge) | `cd_ledger_code` | — |
| charge tax | charge row with `cd_tax_apl` | CR | OUTPUT_* | those |
| discount | `sb_cash_disc` (bill-level, after tax) | DR | DISCOUNT_ALLOWED | DISCOUNT_ALLOWED |
| scheme discount | only if `sales.post_scheme_disc_separately` | DR | SCHEME_DISCOUNT | SCHEME_DISCOUNT |
| round-off | `sb_round_off ≠ 0` | DR or CR | ROUND_OFF | ROUND_OFF |
| TCS | `sb_tcs_amt > 0` (rate resolves to 0 today, Q18) | CR | TCS_PAYABLE | TCS_PAYABLE |
| COGS pair | direct lines, PERPETUAL | DR / CR | COGS / INVENTORY | those |
| tender | per `acc_tender_detail` row | DR tender ledger / CR party | `td_tender_ledger_id` | — |
| loyalty redeem | tender type 10 | DR LOYALTY_REDEMPTION / CR party (the master's `tnd_ledger_id` is ignored) | role | LOYALTY_REDEMPTION |
| credit tender | tender type 9 CREDIT | no leg — the party stays debited; the bill-balance row stays OPEN for exactly the CREDIT amount (this IS the outstanding) | — | — |
| temp credit | tender type 8 TEMP_CR | as CREDIT, plus one `accounts.acc_temp_credit` row: WHO borrowed (name, mobile, place, days → due date) riding on the bill's balance row — §5.14 | — | — |
| RRN | tender type 7 | **refused** — a credit note is an adjustment (`NOTE_ADJUST`), never a tender | — | — |
| advance set-off | `sb_advance_amt` | DR ADVANCE_RECEIVED / CR party | role | ADVANCE_RECEIVED |
| credit-note set-off | a credit note applied on the bill | DR party (the note's row closes) / CR party | — | — (an adjustment row, no leg — the note's legs already exist) |

Line-level discounts (item / special / scheme) net into the taxable value
and therefore into Sales — Tally's default and 3.0's; the separate ledger
is a setting, default off. Charges that are "before tax" are part of the
taxable value (analysis §2, the before-tax trap) and post inside SALES
unless `cd_sep_post`.

**Amounts are always positive; `av_dr_cr` carries the side.** Write order
is fixed (party, sales, taxes, charges, discount, round-off, TCS, COGS pair,
tenders, set-offs) so `av_row_no` is stable across amend diffs.

### 5.4 Cash bill

Same legs, plus per tender row `DR td_tender_ledger_id / CR party`; the
bill-balance row is written with `abl_alloc_amount = Σ tenders` and
`abl_status CLOSED` when fully tendered. Yes, a POS bill writes a party
leg pair and a bill row: bill-wise consistency (the return's "original
bill first") is worth the two rows. Change given (`td_change_amt`) is in
the cash tender's `td_received_amt`.

**Cash limit** (Q39): Σ CASH tenders on one bill, and Σ cash received from
the party on the date across bills + receipts, against
`CASH_TXN_LIMIT_269ST` → REFUSE (per company, `stl_enforce`).
**PAN / Form 60** (Q40): a cash sale whose bill value exceeds
`PAN_REQUIRED_CASH_SALE` needs `cus_pan_no` or a FORM60 certificate row
(or `sb_form60_ref` typed now) → REFUSE.

### 5.5 Bill balance

One `acc_bill_balance` row per bill, always (credit or cash): the return,
the receipt and the credit-note set-off all key on it. `abl_credit_days`
from `sb_due_days`; `abl_due_date` ≥ `abl_doc_date` (CHECK). Maintained by
the live balance trigger + `BillBalanceRecomputeService` as the receipt
does.

### 5.6 Set-offs on the bill

* **Advance** (rule: SO advance adjusted in the bill): every ADVANCE row
  of the party with `abl_pending_amount > 0`, the order's own first, up to
  the bill amount; each an `ADVANCE_ADJUST` adjustment + the leg pair.
* **Credit note** (Q25): the party's open SALES_RETURN credit rows, listed
  in the tender panel like the receipt's credits band; applying one is a
  `NOTE_ADJUST` adjustment against the note's row, no new leg.
* Both are the party-adjustment panel already designed for the receipt
  (memory: party-adjustment-design) — one widget, two hosts.

### 5.7 Register

`gdr_doc_type`: INVOICE for a registered company selling taxable goods;
BILL_OF_SUPPLY when `comp_gst_reg_type` is COMPOSITION or every line is
exempt / nil-rated (`gdr_taxability`). `gdr_supply_nature` from
`sb_pos_stcd` vs the company state (memory: place-of-supply-split — POS is
its own fact). `gdr_party_gstin` NULL ⇒ B2C. HSN digit count on the detail
rows checked against `HSN_DIGITS` for the company class (WARN).

### 5.8 Loyalty (Q20) — checked against 12 / 13, 3.0 `fsales` and the tender dialog (2026-09-20)

**The wallet** is `sales.loyalty_member` (one per company × customer, never
per branch; mobile / card are lookup handles, `lmb_cust_id` is the
identity); **every movement** is a `sales.loyalty_ledger` row (EARN /
REDEEM / GIFT / EXPIRE / ADJUST / TRANSFER / OPENING); caches on the member
are trigger-maintained; consumption is FIFO by `fn_loyalty_lots`
(oldest-expiry first); what may be spent today is `fn_loyalty_redeemable`
(cooling period `lsc_activation_days` + not lapsed); the redeem write is
`LoyaltyLedgerService.consume()` (one REDEEM row per lot, serialised on the
wallet by `SELECT … FOR UPDATE`, never `SKIP LOCKED`). The READS are
database functions by design — one opinion about FIFO order for the service,
the till and the expiry job; the WRITE is the service's, inside the bill's own
transaction (plan-backend §3.4; the rule is in `README.md`).
Points are STATISTICAL — earning posts nothing to the GL; only the
redemption tender does (13's rule).

**Member resolution at post:** `sb_loyalty_member_id` (the screen looked
it up by mobile / card / customer via `party-context`) else the member of
`sb_cust_id`; none and `sales.loyalty_auto_enrol` (default true) → enrol
the customer on this first earning bill; the WALK-IN ledger never earns
(D59) — the quick-add customer is how a shopper joins.

**EARN (server, at post):**
1. scheme = the highest-priority APPROVED `loyalty_scheme` of type EARN /
   BOTH live for the date, time window and weekday, whose branch / customer
   / item scopes admit the bill, `lsc_bill_type` matches CASH / CREDIT,
   `lsc_min_bill_amount` ≤ base — ONE scheme per bill (D59);
2. base per `lsc_apply_on`: BILL_AMOUNT / BILL_QTY over the eligible lines,
   ITEM_AMOUNT / ITEM_QTY per line through `loyalty_scheme_item` (kind
   ITEM / GROUP / CATEGORY / BRAND / SECTION, `lsi_is_exclude`, factor /
   points / max) and `loyalty_scheme_slab` (exceeds / upto / each →
   points or factor); amount type GROSS / NET / TAXABLE, `lsc_include_tax`,
   `lsc_earn_on_discounted`, `lsc_earn_on_charges`; excluded always: free
   items, lines whose `item_allow_loyalty` is false (3.0's rule), and —
   when `lsc_earn_with_redeem` is false — the rupees paid by points;
3. round per `lsc_rounding_method` / `lsc_points_decimals`, cap
   `lsc_max_earn_points`;
4. one EARN row: points, `lld_base_amount / _qty / _rate / _factor`,
   `lld_lsc_id / _lss_id / _lsi_id`, `lld_active_from` = bill date +
   `lsc_activation_days`, `lld_expires_on` per `lsc_expiry_basis`
   (EARN_DATE + valid days / MONTH_END / YEAR_END / SCHEME_END_DATE / NONE),
   `lld_src_doc_type 'SALE_BILL'` (`ux_lld_src_row` makes it idempotent);
   `sb_loyalty_earned` cache.

**REDEEM (from the tender row type 10):** points = `td_units_used`, rate =
the scheme's `lsc_redeem_value_per_point` (the scheme WINS; the tender
master's `tnd_conversion_rate` is only the fallback — 13's rule, so the
tender dialog's `m_loyaltyRate` must come from `party-context`, not the
master); amount must equal points × rate; checks: `fn_loyalty_redeemable` ≥
points, `lsc_allow_point_redeem`, `lsc_min_redeem_points`,
`lsc_max_redeem_points`, `lsc_max_redeem_perc` of the bill,
`lsc_redeem_min_bill_amount`, `lsc_redeem_multiple`, pool mode /
cross-branch; then `LoyaltyLedgerService.consume(tx, member, points, 'REDEEM', …,
tenderId)` inside the post transaction; the leg DR LOYALTY_REDEMPTION / CR
party; `sb_loyalty_redeemed`.

**Cancel:** the EARN reversed by an opposite-sign row naming it (capped at
the live balance if some were already spent — shortfall in `lld_remarks`,
never abort: 13's write-path rule); every REDEEM row reversed (the lots
come back); the tender leg goes with the voucher reversal.
**Return:** `lsc_return_mode` REVERSE → claw-back = earned × returned
taxable ÷ bill taxable, same cap rule; IGNORE → nothing. Points spent on
the returned bill are NOT restored — their rupees are refunded / credited
as money (D59). **Expiry:** `LoyaltyLedgerService.expiryRun()` nightly per
company on the Nest scheduler (Prathap's jobs list) — deterministic run id
`md5(company|date)` so a re-run writes nothing twice; `couponExpiryRun()`
sweeps the 14 vouchers on the same schedule, run id `md5(company|coupon|date)`. **Preview:** `/bills/validate` and `party-context`
return `{memberId, balance, redeemable, rate, minPoints, maxPoints,
maxRedeemAmount, multiple, earnPreview}` — the tender panel shows them.
Gifts (`loyalty_gift_redeem`, 14 coupons) — later.

### 5.9 Promotion (Q21)

The client applies schemes (shared/promotions); the bill line carries
`sbi_scheme_id` / `sbi_is_promo` / `sbi_free_type`; the server VALIDATES
the scheme is live for the date/branch/party and writes `promotion_usage`
(chain-wide usage counts are the reason it is a table, 15 §6). A free item
is a stock row at cost with `sbi_is_free` and taxable 0 (or taxed per
`sales.free_item_tax`).

### 5.10 Cancel

Refusals: any `sale_return` against it (rule: return locks bill); any
`acc_bill_adjustment` against its balance row other than its own set-offs
(a receipt has allocated to it — cancel the receipt first); IRN live and
older than `EINV_CANCEL_HOURS` (Q10: within the window the cancel FIRST
cancels the IRN through the GSP and proceeds only on success; past it,
refused with "raise a credit note"); EWB live and past `EWAY_CANCEL_HOURS`;
day closed; year locked; no `um_can_cancel`.

Then: stock reversal rows (direct lines), reversal voucher (all legs
mirrored, same type, `avh_reversal_voucher_id`), `abl_status CLOSED` via a
negative-amount counter row per set-off and a WRITEOFF-shaped close of the
pending amount marked `abj_reversal_reason 'CANCELLED'`, register CANCELED
(`gdr_doc_cancel_reason`, `gdr_doc_canceled_on`), loyalty REVERSE of the
EARN and of the REDEEM (points come back), usage reversed, DC / order
caches restored, reservation re-opened, `txn_status_log CANCELLED` with
the reason.

### 5.11 Amend (Q9)

`sales.allow_posted_amend` TRUE by default (the receipt's is FALSE; a bill
is corrected far more often than a receipt and the IRN rule already blocks
the case that matters). Same refusals as cancel; plus `baseRevision`. The
IRN case: an amend on a bill whose IRN is live within the window cancels
the IRN and generates a new one after the post; past the window, refused —
GST wants a credit/debit note, not a changed invoice.

### 5.13 The tender popup, and when it opens

`sales.tender_type` (DEVICE; exists live): `all_bills` — F5 always opens
the popup; `cash_bills` — a CREDIT-term bill saves straight, cash bills
tender; `none` — never. The popup is a modal `QDialog`; nothing is written
until Save (F5) / Save & Print (F6). An amount on the **CREDIT** row is
the outstanding: no leg, the bill-balance row keeps it as
`abl_pending_amount`, the receipt collects it.

### 5.14 Temp credit (31_temp_credit.sql)

A non-credit party (the walk-in, usually) pays part and promises the rest
in a few days. At the counter: the **TEMP_CR** row, then the panel — name,
mobile (mandatory), place, pay-in days → due date, notes — pre-filled from
the bill's BILL-TO band. Post writes the tender row (type 8, no leg, the
balance row OPEN for the amount) **and one `acc_temp_credit` row** on that
balance row: the WHO. Guards: `sales.temp_credit_max_amount` (WARN),
`sales.temp_credit_max_days` (REFUSE), an OPEN temp credit already on that
mobile (`sales.temp_credit_block_open`: WARN with the balance, or REFUSE).
`atc_balance_amount` is maintained = the balance row's `abl_pending_amount`
by `BillBalanceRecomputeService` — a receipt against the bill, a write-off,
a credit-note set-off all move it; nobody edits it. Bill cancel → row
CANCELLED (never deleted). The receipt finds the borrower by mobile / name
(open-items on the walk-in ledger carries the temp-credit block) and
settles the BILL like any bill. Menu "Temp credits" lists OPEN / PARTIAL
rows with days overdue, for the counter to see at a glance who owes what.

### 5.12 Delivery status (Q27), delivery mode

`sb_delivery_status` NA for a POS bill and any bill without
`sales.delivery_status_tracking`; else PENDING at post; menu 226 posts
events PENDING → (VERIFIED, when `sales.require_verification_before_dispatch`
— 3.0's validation screen, D54) → PACKED → DISPATCHED → DELIVERED. No
stock effect — the stock left at post (direct) or at the DC. A POSTED
bill's remarks may be changed text-only through `/bills/update-remarks`
(D57, logged).

---

## 6. The Sale Return

### 6.1 Post

1. Guards (§9) + return-specific: bill POSTED and not CANCELLED; per line
   `sri_return_qty + sri_free_qty ≤ sbi_bill_qty − sbi_return_qty`;
   `sales.return_window_days` (WARN); cross-branch ok (Q31: the return's
   branch is where the goods land); `item_allow_sales_return`;
   `CREDIT_NOTE_CUTOFF` (Q38): a return dated after 30 Nov of the FY after
   the bill's FY WARNs "GST credit note not declarable — post as a
   financial credit note" and posts with `gdr_taxability` carried but
   `gdr_doc_type 'CREDIT_NOTE'` and a remark; the accountant decides in
   GSTR-1.
2. Stock: `SALE_RETURN` rows, +qty, `sml_bucket = sri_bucket` (Q3: a
   damaged return lands in DAMAGED and is not sellable), `sml_lot_id =
   sri_lot_id` (the bill line's lot when it still exists; else a new lot
   with the bill line's cost).
3. Legs (type `SRt`): `DR SALES_RETURN taxable · DR OUTPUT_* tax · CR party
   return amount`; charges refunded on their own ledgers; round-off; then
   the COGS reversal `DR INVENTORY / CR COGS` at `sri_cost_price × qty`
   (not on a free return).
4. Bill balance: a `SALES_RETURN` row, `abl_dr_cr 'CR'`, amount = return
   amount — the credit note the party holds.
5. Settlement (Q2):
   * **CASH** — tender rows (`td_src_doc_type 'SALE_RETURN'`, `td_dr_cr
     'CR'`), legs `DR party / CR tender ledger`, the CR row closed
     (`abl_status CLOSED`, alloc = amount).
   * **ADJUST** — `NOTE_ADJUST` adjustments from the CR row against the
     ORIGINAL bill's row first (up to its pending), then the party's other
     open bills by due date; whatever remains stays open on the CR row.
   * **ADVANCE** — nothing more; the CR row is the credit note, applied
     on a later bill (§5.6) or receipt.
   `sr_refund_amt / sr_adjusted_amt / sr_credit_amt` are the three caches;
   `ck_sr_settle_sum` keeps them honest.
6. Register: `CREDIT_NOTE`, `gdr_doc_flow 'OUTWARD'`, `gdr_doc_sign -1`,
   `gdr_doc_note_no / _date` = the ORIGINAL invoice (`chk_gdr_note_pair`),
   `gdr_doc_note_reason` = `sr_return_reason`; detail rows; e-invoice
   applicable on a CN when the invoice was (Q: IRN on a credit note is
   required for an e-invoicing company); e-way when goods travel back by
   transporter (`sr_*` has no transport block by default — added on demand
   as on the DC return).
7. Loyalty REVERSE pro-rata; promotion claw-back (Q29): free items not
   returned with the goods that earned them are re-priced — the server
   computes the claw-back as a positive line on the credit note
   (`sr_promo_clawback_amt`) and reports it; `promotion_usage` reversed.
8. Caches on the bill: `sbi_return_qty`, `sb_returned_amt`,
   `sb_return_status` (PARTIAL / FULL). This is the lock.
9. Status log, status.

### 6.2 Cancel / amend

Refused if the CR row has been applied anywhere (`abl_alloc_amount > 0`
from another document) — release those first; if the CN's IRN is past the
window. Then the mirror of §6.1.

### 6.3 Return-and-re-bill (Q28)

A screen gesture, not a document: "Return & re-bill" posts the return in
ADJUST mode and opens a new bill for the same customer with the returned
lines pre-loaded, source = the return. Two documents, two numbers, the
audit sees both.

---

## 7. Charges across part deliveries (Q37)

The order holds charge rows (`cd_doc_type 'ORDER'`). When a bill (or DC)
takes SOME of the order's lines, each order charge row is offered to the
bill with a proposed amount:

* `PRORATA` (default): `cd_amount × (this bill's order-line taxable ÷ the
  order's total taxable)`, rounded; the bill that completes the order takes
  `cd_amount − cd_carried_amt` so the sum closes exactly.
* `FULL`: the first bill takes it all; later bills see `cd_carried_amt =
  cd_amount` and get nothing.
* `MANUAL`: the operator typed the figure; `NONE`: declined on this bill.

The bill's charge row: `cd_src_cd_id` + `cd_src_acc_year` → the order row;
`cd_carry_basis` the choice. Post updates the source row's
`cd_carried_amt`; cancel subtracts. Over-carry (`cd_carried_amt >
cd_amount`) is refused unless `um_can_override`. A DC never carries a
charge (it has no money); the bill against a DC that came from an order
resolves the charge from the ORDER via the DC line's `sdi_src_doc_id`.

---

## 8. The law, per company

### 8.1 What is read, and from where

| question | source |
|---|---|
| does GST apply at all | `comp_gst_applicable`, `comp_gst_reg_type` |
| e-invoice on this bill | `comp_einvoice_applicable` ∧ `sb_bill_date ≥ comp_einvoice_date` ∧ party GSTIN present (B2B / SEZ / export) ∧ doc INVOICE / CREDIT_NOTE / DEBIT_NOTE; `EINV_AATO_THRESHOLD` × `company_aato` is what the accountant checks when setting the flag — the guard reads the FLAG |
| e-way on this document | §8.2 |
| HSN digits | `HSN_DIGITS` for `comp_aato_class` |
| IRN / EWB cancel window | `EINV_CANCEL_HOURS`, `EWAY_CANCEL_HOURS` |
| cash limits, PAN | `CASH_TXN_LIMIT_269ST`, `PAN_REQUIRED_CASH_SALE` |
| TCS | `comp_tcs_applicable` ∧ `led_is_tcs_applicable` ∧ `TCS_206C1H_*` resolve on the date (they do not, from 2025-04-01: rate 0, no leg) — infra kept, Q18 |
| credit note cutoff | `CREDIT_NOTE_CUTOFF` |
| challan window | `DC_RETURN_WINDOW_DAYS` |

Every guard message names the code, the value, the effective-from date and
whether it came from a company override — so the accountant can find the
row.

### 8.2 E-way bill applicability

Goods (not services) ∧ the document moves them (DC, direct bill lines,
DC return, return with transport) ∧ consignment value (taxable + tax of the
moving lines) > `EWAY_VALUE_LIMIT` resolved with `applies_to` = the
destination state code when the supply is intra-state (state rows, e.g.
'33' for TN at 1,00,000) else 'INTER_STATE' ∧ `comp_eway_applicable` ∧
`comp_eway_date ≤ doc date`. `comp_eway_intra_apl` / `_limit` on the
company row are the L0 override of the state row. Below the limit the
register row says `gdr_is_ewaybill_applicable false` and nothing fires;
the operator may still generate one by hand from the register (152/153).
Part-B (vehicle) may be filled later from the register when the DC leaves
without a vehicle number.

### 8.3 Tally export

Bil → Sales, SRt → Credit Note, DCh → Delivery Note, DCR → Rejections In
(`vchr_tally_*`, 30 §9). A CANCELLED document exports as a cancelled
voucher (Tally keeps the number, Q41); the reversal voucher itself is not
exported (the exporter skips `avh_reversal_voucher_id IS NOT NULL` whose
original is CANCELLED). Ledger names via `led_tally_name`; units via
`unit_tally_name`; the COGS pair exports as a Journal under a "Perpetual
COGS" narration, or is filtered out when the Tally company runs periodic —
a per-company export flag, later.

---

## 9. Guards, in order, on every post

1. rights: `um_can_post` on the menu; `um_can_override` when a WARN is
   being passed (the client sends `overrides[]` = the codes it accepts;
   the server re-checks and logs each)
2. `assertAccYearWritable`, `assertVoucherPartitionExists` (receipt's)
3. date: not after today; before today per `sales.backdate_mode`; day not
   closed (accounts day-close on the LEDGER, memory: accounting-scope)
4. company: GST reg type vs the document (a composition dealer raises a
   BILL_OF_SUPPLY, collects no tax, gets no IRN); AATO class present
5. party: not deleted, credit allowed for a CREDIT bill,
   `sales.credit_limit_mode` against limit / bill count / days (server,
   Q47), GSTIN shape when present, PAN / Form 60 (§5.4)
6. lines: qty > 0; rate vs min (Q4, WARN); discount caps (Q36, WARN);
   `HSN_DIGITS` (WARN); source-line open qty; duplicate-item rule
7. stock: negative-stock policy (`comp_negstk_apl`), lot BLOCKED/EXPIRED
   refused
8. money: Σ lines + charges − discounts + round-off = bill amount (±0.01);
   tenders ≤ bill amount unless `sales.allow_excess_tender`; cash limit
9. law: everything in §8 for this document type

A refusal is a 409/422 with the guard's code; a WARN is returned in
`warnings[]` on the dry-run (`/validate`) and is a refusal on `/post`
unless its code is in `overrides[]` and the user may override.

---

## 10. Offline contract (Q43)

**Guards, not just payloads.** `offline-sync-invariants.md` settles which
constraints may still refuse once writes arrive late and out of order: a CHECK
may guard one row's own arithmetic, never a contended shared resource. Six
over-draw constraints in `13`, `14`, `29` and `31` are now recorded over-draws
plus a partial index — the policy moved to the posting services, which refuse
at post while online, and the reconciliation report is what catches the rest.


Posting is ONLINE — the counter cannot see chain stock or the party's
ledger. The payload is offline-ready: ids uuidv7 minted at the counter,
`sb_bill_slno` from the device's own series, every snapshot column filled
(customer, prices, tax) so a bill drafted offline posts unchanged when the
link returns, and `/create` is an upsert on the id (a re-sent draft is a
no-op). The register, the legs and the stock rows are always the server's.

---

## 11. Rules, restated as checks

| rule | enforced by |
|---|---|
| a return locks its bill | `sale_bill.cancel` refuses when a `sale_return` exists |
| a DC return locks its DC | `sale_dc.cancel` refuses when `sdi_returned_qty > 0` |
| a bill locks its DC | `sale_dc.cancel` refuses when `sdi_billed_qty > 0` |
| SO advance is adjusted in the bill | §5.6, automatic, order's own advance first |
| DC deducts stock; bill against DC does not; direct bill does; returns add back | §1 table; `sbi_src_doc_type` decides per LINE |
| IRN / EWB block amend | §5.11 / §4 |
| DC return only for un-billed lines | `sdi_open_qty` guard |
| free return allowed, flagged | `sr_is_against_bill false`, no COGS reversal |
| walk-in is a customer | `sales.default_customer_id`, its ledger |
| nothing posted is deleted | reversal rows everywhere; delete is DRAFT-only |

---

## 12. Decisions recorded here (all "default", 2026-09-20)

The fifty in `analysis_2026-09-20.md` §12, plus two taken while writing:

* **D51** the COGS pair is one leg pair per document, not per line —
  cost has no rate and the stock ledger has the line detail.
* **D52** a POS bill writes the party pair and the bill-balance row like
  a wholesale bill — bill-wise consistency over two rows saved.
