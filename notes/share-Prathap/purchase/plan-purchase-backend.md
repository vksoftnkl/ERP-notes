# Purchase phase 1 — BACKEND plan (NestJS + PostgreSQL)

Date: 2026-10-27. From: Viji / Claude. Implemented by Claude on Prathap's machine (`~/Dev/erp/ERP server`, branch **DEV-v1**) from 28-Oct.

**Inputs, in order:**
1. `plan-purchase-phase1.md` (§2 master register = scope);
2. `…-addendum-1.md` (A1–A3);
3. `…-addendum-2.md` (R1–R28 + owner answers B1–B7);
4. `notes-162-purchase-platform-prereqs.md` (P1–P20);
5. the mockups (visual spec).

The frontend plan is `plan-purchase-frontend.md`.

## 0. Working rules

- **Repo, numbering, sync:**
  - Work in the current folder on **DEV-v1** as it is (owner, 27-Oct: no separate branch, no clean-up first). Prathap's uncommitted files are never touched, stashed or committed by me. I commit only my own files by explicit path (never `git add -A`). If a file has his open changes too, I tell the owner before committing it.
  - `nest build` / `prisma generate` rewrite `dist/` + the client in that folder. The pm2 API keeps running its loaded code until a restart, which happens only on the owner's / Prathap's word.
  - One commit per step below, message `feat(purchase): <step> …`.
  - **Migrations:** `prisma/migrations/2026MMDDhhmmss_purchase_<step>/migration.sql`. Each SQL goes to the owner first; it is applied (`prisma migrate deploy`) only after their OK.
  - **Existing tables** (notes-162 fixes) are separate migrations, flagged "touches existing".
  - The live API (pm2, :3000) is restarted only when Prathap says so. Until then my build runs on **:3012** against the same DB, and the Qt client points there for purchase tests.
- **House rules:**
  - no DB functions / triggers (CHECKs + indexes OK);
  - every transaction table `PARTITION BY LIST (acc_year)`, PK (id, acc_year), uuidv7 ids;
  - `_sync_date` + created / modified / deleted on every table (joins `v_sync_tables` automatically);
  - no CHECK / UNIQUE over anything two offline branches can both create; caches derived, never incremented;
  - Decimal end-to-end (`Prisma.Decimal`, no `Number()` on money, no epsilon);
  - one partyId = ledger id;
  - new module = own URL family `/purchase/*`;
  - **lists come from grids, not /list routes.**
- **Every document:**
  - per-device numbering with `dev_code`: `PO/` `GRN/` `PUR/` `PRN/` `SHR/` + `{year}/{dev_code}/{nnnnn}`;
  - `txn_status_log` on every status change; `log_rev_no` snapshot per save (LAW-15);
  - `/txn/adjacent` registry entry; `txn_hold` support;
  - save with `post: true` = save + post in one transaction.

## 1. Build order (each step: migration → code → unit tests → e2e on :3012 → commit)

| Step | Deliverable | Depends |
|---|---|---|
| **B0** | Prerequisites in existing modules: P1 (voucher slno per device), P2 (drop `ck_abl_settled` → partial index + report), P3 (last purchase rate = PURCHASE only), P5 (`ipcl_source` + PURCHASE), P6 (doc-type CHECKs), P7 (voucher types + INWARD credit note), P8 (roles + ledger map seed), P12 (FK), P14 (charge columns), P17 (company-wide duplicate check helper) | — |
| **B1** | Masters: supplier fields (MST-01) + `pur_supplier_gstin`; item fields (MST-03); `pur_supplier_item`; `pur_agent`; purchase settings keys; menus + rights rows | B0 |
| **B2** | **Calc engine** `purchase-calc.ts` (pure, Decimal) + golden vectors shared with Qt | — |
| **B3** | **Purchase stock writer** (P4) + **revaluation rows** (P13) + batch detail / serial alloc | B0 |
| **B4** | Purchase Order: save / get / short-close / clone / cancel; pending derived | B1, B2 |
| **B5** | GRN: save / post / cancel; stock in; GRNI legs; off-bill charges per loadman; shortage row (qty only); labels source | B3, B4 |
| **B6** | Purchase Invoice: direct / from GRN / from PO / from sister sale bill; validate; post; cancel; ITC amounts; payment hold; cash mode; TCS; price writer (setting-gated) | B5 |
| **B7** | Shortage register + Shortage Receipt + `pur_shortage_close` | B6 |
| **B8** | Purchase Return / Claim / Replace (against invoice / GRN) + claim settlement | B6, B7 |
| **B9** | Lookups: supplier context, item context (last rates all suppliers, PO pending, supplier code), source pickers, label payload | B4–B8 |
| **B10** | Grids for lists + reports RPT-01…11 (grid SQL, no comments) | B4–B8 |
| **B11** | Payment 100 integration: payment-hold refusal + release right; loadman payables visible; return / claim set-off | B6, B8 |
| **B12** | Acceptance suite (plan §9 + addenda) green; Prathap review; restart live API | all |

**P10, P11, P15, P16, P18–P20 slot in where first needed:**
- P10: before B3;
- P15 (value slab): B2;
- P16 (attachments): B6;
- P18 (EAN online-only + detect): B9;
- P19 (inter-company lock): B6;
- P20 (EWB on header): B6;
- P11 (sync rebuild of lot first-writer / charge carry): B3.

## 2. Schema (`prisma/purchase/*.prisma`, schema `purchase`)

Every txn table also carries the standard columns:
- `<p>_company_id`, `<p>_branch_id`, `<p>_acc_year char(9)`, `<p>_device_id`, `<p>_session_id`;
- `<p>_status` (DRAFT / POSTED / CANCELLED, + doc-specific);
- `<p>_slno`, `<p>_refno`, `<p>_doc_date`, `<p>_doc_datetime`;
- `<p>_remarks`, `<p>_print_count`, `<p>_cancel_reason`;
- created / modified on / by, `is_deleted`, `sync_date`.

Listed below: the purpose-specific columns only.

### 2.1 Masters
- `pur_supplier_gstin` (psg_): supplier_id → suppliers, gstin, state_code, address 1–3, city, pincode, is_default, active. Id = uuid v5(company, supplier, gstin).
- `pur_supplier_item` (psi_): supplier_id, item_id, supplier_item_code, supplier_barcode, buy_uom_id (iuc_id), pack_qty, moq, lead_days, is_preferred, active, source (TYPED / AUTO). Id = uuid v5(company, supplier, item). **Last rate / date / bill count are NOT columns** — read from posted invoice lines (view `v_pur_supplier_item_stats`).
- `pur_agent` (pag_): name, phone, ledger_id (optional), active.
- **Supplier / ledger additions** (existing tables, migration flagged):
  - `sup_msme_category` (MICRO / SMALL / MEDIUM / NONE), `sup_msme_activity` (MANUFACTURER / SERVICE / TRADER), `sup_msme_agreement` bool, `sup_msme_agreed_days` smallint;
  - `sup_drug_licence_expiry` date; `sup_fssai_no`, `sup_fssai_valid_to`;
  - `sup_default_godown_id`, `sup_show_notes_on_entry` bool, `sup_is_import` bool.
- **Item additions:** `item_purchase_uom_id`, `item_min_shelf_life_pct` numeric(5,2), `item_itc_class` (ELIGIBLE / INELIGIBLE / BLOCKED_17_5), `item_shortage_pct` numeric(5,2), `item_default_markup_pct` numeric(7,3).
- **Charge:** `charge_master.chg_on_bill` bool default true; `txn_charge_detail.cd_on_bill`, `cd_party_id uuid` (one detail row per payee).

### 2.2 Purchase Order
- `pur_order` (po_): supplier_id, expected_date, deliver_godown_id, agent_id, contact, credit_days, terms_id / terms_text, approval_status (NOT_REQUIRED / AWAITING / APPROVED / REJECTED), revision_no, short_closed bool + reason, totals (gross, tax, net — roll-up of lines, recomputed on save).
- `pur_order_item` (poi_): line_no, item_id, uom_id, base_uom_id, to_base_factor, qty, free_qty, rate, disc_pct, disc_amt, disc_per_qty, sch_pct, extra_disc_amt, tax_id, gst_pct, cess_pct, cess_per_unit, taxable, tax_amt, total, mrp, expected_sell, model / size_id / colour_id (textile, optional), short_closed_qty + reason, remarks.

### 2.3 GRN
- `pur_grn` (grn_): supplier_id, supplier_gstin_id, po_ids (via link), challan_no, challan_date, received_at, godown_id, received_by (employee); crew on the document (house rule) as loadman_ids uuid[], supervisor_id, vehicle_no, driver_name; ewb_no, ewb_date, packages_sent / received; bill_status derived (NOT_BILLED / PART / BILLED via link).
- `pur_grn_item` (gri_): line_no, item_id, uom, factor, challan_qty, received_qty, damaged_qty, damage_reason_id, free_qty, excess_action (HOLD_FOR_BILL / RETURN / KEEP_FREE), provisional_rate (net, w/o tax, per uom), mrp, sell_a (only if `allow_price_update`), remarks.

### 2.4 Invoice
- `pur_invoice` (pi_):
  - supplier_id, supplier_gstin_id, supplier_inv_no, supplier_inv_date, entry date = doc_date, goods_received_on;
  - source_kind (DIRECT / GRN / PO / SISTER_SALE), invoice_amount_printed, mode (CREDIT / CASH), credit_days, due_date (MSME-capped, R27);
  - agent_id, place_of_supply, is_reverse_charge, ewb_no, ewb_date, irn, ack_no;
  - payment_hold bool + hold_reason + released_by / on;
  - tcs_amount, round_off;
  - totals: taxable, cgst, sgst, igst, cess, charges_on_bill, net_payable, offbill_charges, short_value, itc_pending.
- `pur_invoice_item` (pii_) = **the printed line**:
  - line_no, item_id, hsn, uom, factor;
  - billed_qty, received_qty, damaged_qty, short_qty, absorbed_short_qty (B4), excess_qty + excess_action, rejected_at_grn_qty, free_qty;
  - rate, disc_*, sch_pct, extra_disc_amt, gross, taxable, tax_id, gst_pct, cgst / sgst / igst / cess amts, cess_per_unit;
  - **itc_eligible_amt, itc_pending_amt, itc_ineligible_amt, itc_class**;
  - landed_cost_wot per base unit, markup_pct, sell_a…d, mrp, old_prices jsonb (what it was), price_update_result (UPDATED / KEPT_NEWER / OFF);
  - weight, remarks.
- `pur_item_alloc` (pia_) — stock side, **polymorphic parent** (doc_type GRN / INVOICE / RETURN / SHORTAGE_RECEIPT + line id):
  - godown_id, bucket (SALEABLE / DAMAGED);
  - batch_no (null until post when auto batch), mfg, expiry, mrp, sale_price;
  - size_id, colour_id, model;
  - serial_no, warranty_to;
  - qty, free_qty, bill_line_no (textile grouping);
  - lot_id (filled at post), svi_id.

### 2.5 Links, shortages, returns, claims
- `pur_doc_link` (pdl_): src_doc_type, src_doc_id, src_line_id, **src_acc_year**, tgt_doc_type, tgt_doc_id, tgt_line_id, **tgt_acc_year**, base_qty, kind (RECEIVE / BILL / RETURN / SHORT_CLOSE / SISTER_PULL). Partitioned by tgt_acc_year.
- `pur_shortage` (psh_): source doc / line / year, supplier_id, item_id, qty_base, value (null until billed), gst_pending, reason (SHORT_BILL / SHORT_CHALLAN / REPLACE), origin_cost_rate. **Status / open qty derived.**
- `pur_shortage_close` (psc_): shortage_id + year, kind (SUPPLIER_CN / CLAIM / WRITE_OFF), qty, amount, cn_no / date, reason.
- `pur_shortage_receipt` / `_item` (psr_ / psri_): header like the GRN (received_at, godown, challan, received_by, loadmen); lines → shortage_id, qty, alloc.
- `pur_return` (pr_): kind (AGAINST_INVOICE / AGAINST_GRN), source doc id + year, supplier_id, settlement (ADJUST / CASH / ADVANCE), place_of_supply, supplier_cn_no / date, ewb_no, vehicle.
- `pur_return_item` (pri_): source line, outcome (RETURN / CLAIM_SENT / CLAIM_DESTROYED / REPLACE), reason_id (stock reasons, purchase set), qty, bill_rate, value, gst amts, lot_cost_value, itc_reversed_in_books bool, supplier_cn_status (PENDING / MATCHED), claim_against (SUPPLIER / TRANSPORTER / INSURER) + claim_party_id.
- `pur_claim_settlement` (pcs_): return_item id + year, action (CREDIT / SHORT_SETTLED / REJECTED), received_as (CN / CASH / BANK / ADJUST), amount, cn_no / date, reason, reversal_of_id (undo = reversing row).

### 2.6 Indexes
- Non-unique on every FK + `(company, supplier, supplier_inv_no)` (duplicate detect).
- **No unique index on the supplier invoice no** (R1 / P17).
- Unique only on what one device mints: `(company, branch, acc_year, device, slno)` per doc table.

## 3. Module layout (`src/modules/purchase/`)

```
purchase.module.ts
common/   purchase-calc.ts (+ .spec with golden vectors)   purchase-settings.ts   purchase.errors.ts
          purchase-itc.ts (ITC rules: supplier type, item class, ledger, short/pending)
          purchase-stock.service.ts (P4 writer)   purchase-leg.sources.ts (account legs)
          purchase-price.service.ts (D16 + allow_price_update + price log)
          purchase-numbering.ts   purchase-link.service.ts (derive pending / billed / open)
          purchase-doc-store.ts (save lines, snapshot, status log)
order/  grn/  invoice/  shortage/  return/  claim/  supplier-item/  supplier-gstin/  agent/  lookup/  labels/
```

Each document folder: `*.controller.ts`, `*.service.ts` (save / get), `*-posting.helper.ts` (post / cancel), `dto/`, `types/`, `README.md` (the contract, as other modules do), specs.

## 4. Calculation (`purchase-calc.ts`) — server is the authority

One pure function, used by every document and by `/purchase/invoices/validate`:
- **Line:**
  - gross = qty × rate;
  - discounts (pct | per-qty × qty | amount) → scheme % → extra amount = net gross;
  - + before-tax on-bill charge share = taxable;
  - GST by date + value slab (P15, warn only, R25) + cess % + cess / unit;
  - total.
- **Charges:** spread by `chg_cost_alloc` (VALUE default; QTY / WEIGHT), residue to the last line; ADD / DEDUCT sign; before / after tax; on-bill vs off-bill; payee split for off-bill rows (equal by default, given amounts win).
- **Landed cost w/o tax per base unit:**
  - numerator: taxable + landing-cost charges (on- and off-bill, CD as deduction — B2) + ineligible GST;
  - denominator: received + damaged + free, base units;
  - absorbed short (B4) stays inside the value; free shares the cost (D2).
- **Bill:** Net payable = Σ line totals + on-bill after-tax charges (+ their GST) + supplier TCS ± round off.
- **ITC per line:** eligible on (received + damaged) share; pending on the short share; ineligible by class / rule.
- **Golden vectors** (`purchase-calc.vectors.json`) are checked into the server repo and copied into the Qt tests. Both sides must produce identical figures.

## 5. Posting (per document; one DB transaction; idempotent on retry)

| Doc | Stock (purchase stock writer, P4) | Accounts (`purchase-leg.sources.ts` → `VoucherPostingService.postLegs`) | Other |
|---|---|---|---|
| GRN | RECEIPT `['PURCHASE']`: received + free → SALEABLE, damaged → DAMAGED, cost = provisional net rate (+ off-bill landing share) | Dr INVENTORY / Cr GRNI; Dr INVENTORY / Cr each payee (off-bill, bill-wise row per payee) | shortage row (qty only); status log; snapshot |
| Invoice direct | RECEIPT as above at landed cost | Dr INVENTORY, Dr INPUT_* (eligible), Dr INPUT_TAX_PENDING (short), Dr SHORTAGE_RECEIVABLE (short value), Dr TCS_RECEIVABLE, round off; Cr Supplier (bill-wise PURCHASE, due date); Cr off-bill payees. Cash mode: Cr branch cash ledger instead of a supplier balance (B6) | doc register INWARD; price writer; shortage adopt / open; payment hold |
| Invoice from GRN | none (stock is in); revaluation rows for the on-hand share (P13, R2) | Dr GRNI (provisional), Dr INPUT_*, ± INVENTORY (on-hand share) / PURCHASE_PRICE_VARIANCE (rest), on-bill charges split the same way, TCS, round off; Cr Supplier | adopt the GRN's shortage rows (R3) |
| Invoice from sister sale | as direct; lots from the sale lines (batch / MRP / expiry) | as direct | link SISTER_PULL; lock the sale bill (P19) |
| Shortage Receipt | RECEIPT at origin cost | Dr INVENTORY / Cr SHORTAGE_RECEIVABLE; Dr INPUT_* / Cr INPUT_TAX_PENDING | close via receipt |
| Shortage close (CN / claim / write-off) | — | Dr Supplier (or claim party / WRITE_OFF) / Cr SHORTAGE_RECEIVABLE + INPUT_TAX_PENDING | `pur_shortage_close` row |
| Return row | ISSUE `['PURCHASE_RETURN']` exact lot | Dr Supplier (bill-wise PURCHASE_RETURN) or cash; Cr INVENTORY at lot cost; Cr INPUT_* at bill rate; difference → PURCHASE_PRICE_VARIANCE | `itc_reversed_in_books` |
| Claim — goods sent | ISSUE | Dr CLAIM_RECEIVABLE (on claim party); Cr INVENTORY (lot cost), Cr INPUT_*, PPV difference | — |
| Claim — destroyed | ISSUE (write-off) | Dr CLAIM_RECEIVABLE (value); Cr INVENTORY (lot cost), PPV; Dr ITC_REVERSAL / Cr INPUT_* | — |
| Replace | ISSUE | Dr SHORTAGE_RECEIVABLE / Cr INVENTORY (lot cost); Dr INPUT_TAX_PENDING / Cr INPUT_* | opens a shortage row (REPLACE) |
| Return against GRN | ISSUE | Dr GRNI / Cr INVENTORY | — |
| Claim settlement | — | CREDIT: Dr party / Cr CLAIM_RECEIVABLE; SHORT_SETTLED: Dr WRITE_OFF; REJECTED: Dr Input GST (re-avail) + Dr WRITE_OFF (value) / Cr CLAIM_RECEIVABLE | reversal row on undo |
| Cancel (any) | mirror reversal (refused if a holding would go negative — the existing rule) | `reverseLegs` | bill-wise reversal rows; status log |

**Price writer** (only when `purchase.allow_price_update` = on, B3):
- per changed price row, checks the Price Change Log by business time (R14 / D16);
- writes through the existing price services with source PURCHASE;
- records `price_update_result` on the line.

## 6. Routes (`/purchase/*`; DTO keys camelCase with prefix po / grn / pi / pr / psr / pcs; `partyId` for the supplier)

```
POST /purchase/orders/save | /short-close | /clone | /cancel        GET /purchase/orders/get
POST /purchase/grns/save (post:true) | /cancel                       GET /purchase/grns/get | /pending-po-lines
POST /purchase/invoices/save (post:true) | /cancel | /release-hold  GET /purchase/invoices/get | /validate | /sources (GRN / PO / sister bills)
POST /purchase/returns/save (post:true) | /cancel                    GET /purchase/returns/get | /source-lines
POST /purchase/claims/settle | /settle-undo                          GET /purchase/claims/open
POST /purchase/shortages/receipt/save | /receipt/cancel | /close     GET /purchase/shortages/open | /receipt/get
POST /purchase/supplier-items/save | /deactivate                     GET /purchase/supplier-items/get
POST /purchase/supplier-gstins/save | /deactivate                    GET /purchase/supplier-gstins/get
GET  /purchase/lookup/supplier-context | /item-context | /barcode    GET /purchase/labels/from-doc
```

Lists use registered grids (§8).

## 7. Validation catalogue (`purchase.errors.ts`; 422 with `errors[].field = lines.<i>.<key>`)

DUPLICATE_SUPPLIER_INVOICE (online, company-wide) · INVOICE_AMOUNT_MISMATCH (REFUSE / WARN by setting) · RATE_TOLERANCE (→ hold, not refuse) · SHELF_LIFE (WARN / REFUSE) · SERIAL_COUNT · SERIAL_DUPLICATE · ALLOC_SUM_MISMATCH · BATCH_REQUIRED / EXPIRY_REQUIRED (by track preset) · BATCH_TYPED_WITH_SIZE (textile) · GRN_BRANCH_MISMATCH (R2) · GRN_LINE_LOCKED · RETURN_QTY_EXCEEDS · OFFBILL_GTA_REFUSED (R12) · RCM_UNREGISTERED_NOTIFIED (R13) · PRICE_BELOW_COST / PRICE_ABOVE_MRP · EDIT_LOCKED_PAID · EDIT_QTY_BELOW_CONSUMED · PAYMENT_HOLD · SISTER_BILL_ALREADY_PULLED · ITEM_NOT_SHARED (inter-company) · PO_NOT_APPROVED · EAN_OFFLINE.

## 8. Grids, menus, rights, settings

- **Grids** (registered by script in `share/grids/`, SQL without comments):
  - lists: PO, GRN, invoice, return, shortage receipt, claims;
  - pickers: pending PO lines, un-billed GRNs, sister sale bills, open shortages, return source lines;
  - reports: RPT-01…11.
- **Menus:** 23 PO, 24 Purchase Invoice ("Purchase Entry"), 25 Return, 195 Shortage Receipt + **new**: GRN, Claims, Supplier–Item Map, Purchase Settings (ids at registration).
  - Rights per menu (view / add / edit / post / cancel / print); **"Release payment hold"** and **"Change price from purchase"** as extra verbs on 24.
- **Settings `purchase.*`:** default_entry, inv_amount_check, allow_price_update, price_update, price_set_at, rate_tolerance_pct, shelf_life_mode, min_shelf_life_pct, return_settlement_default, close_po_on_first_bill, labels_after_post, duplicate_bill_mode, po_approval_above, cash_purchase_day_limit. The document snapshots the money-steering ones.

## 9. Tests

- **Unit:** calc vectors; ITC rules; leg builders balance (Σ Dr = Σ Cr) per doc type; link derivations; D16 ordering; R2 split.
- **e2e** (`test/purchase-*.e2e-spec.ts`, ZT- data, cleans up by cancelling, never deleting till / stock history), covering plan §9:
  - PO → GRN part → invoice +2 %;
  - direct 5 / 3 short + Shortage Receipt;
  - coolie 2 loadmen + Payment;
  - one return with all 4 outcomes + claim settle / reject;
  - old-bill edit vs newer price;
  - paid-bill edit refusal;
  - offline duplicate detect;
  - inter-company pull + lock;
  - textile ratio pack + labels + scan;
  - serial count / duplicate;
  - cancel paths.
- **Reconciliation after each e2e:**
  - Stock-in-Hand ledger = stock valuation;
  - GRNI per branch = un-billed GRN value;
  - SHORTAGE_RECEIVABLE = open shortage value;
  - CLAIM_RECEIVABLE = open claims.
