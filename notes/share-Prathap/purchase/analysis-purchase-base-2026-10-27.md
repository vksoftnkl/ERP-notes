# Purchase module — base analysis (rev 1)

Date: 2026-10-27. From: Viji / Claude. For: the user, then Prathap.

The user's base doc ("VK NEX ERP – PURCHASE MODULE: DESIGN & GAP ANALYSIS", GoFrugal / Tally / Odoo) was checked against three things:
1. **3.0's purchase module, line by line** (`NextLvl/3.0/pos/Purchase`, `fpurchase()` and the 2024 "GRN generation" tree with Purchase Note / Shortage Receipt / expense master);
2. **the market and Indian law as of Oct 2026** (Tally, GoFrugal, Ginesys, Odoo, Zoho; CGST Act/Rules, IMS, Income-tax Act 2025, MSMED);
3. **what NexERP already has** (live DB on .106 + the NestJS source + the Qt client).

This file is analysis only. Each phase gets its own plan + mockup files afterwards.

**User decisions (27-Oct):**
- goods arrive before the supplier bill → **GRN is in phase 1**;
- purchase **may be keyed at offline branches**;
- **textile** in phase 1;
- **Purchase Order** in phase 1.

---

## 1. Verdict on the base doc

The base doc gets the shape right:
- the document chain;
- shortage outcomes;
- cash discount kept out of entry cost;
- the landed cost voucher;
- the duplicate-bill guard;
- line-level allocation.

Keep all of that. What it lacks falls into five groups:
- **(a) features 3.0 clients already use** that it never names (§3);
- **(b) whole flows** the market treats as standard: consignment/SOR, job work, central buying for branches, three-way match (§4);
- **(c) the 2024–26 law changes**: IMS, the 180-day rule, GST 2.0 rates, the new Income-tax Act numbering, MSME scope (§5);
- **(d) NexERP platform defects** that purchase will hit on day one (§6);
- **(e) house-rule conflicts** (§7).

---

## 2. The module map — what "purchase" must contain

| Area | Documents / pieces | Phase |
|---|---|---|
| **Masters** | Supplier gaps (§8); supplier–item map (supplier code, pack, last rate per supplier, lead time, preferred); purchase expense master (= `charge_master` module 'P'); payment terms; purchase agent; transporter (exists as ledger group) | 1 |
| **Buying** | Purchase Order (pending, clone/merge, short close, amendment rev, WhatsApp/PDF), Reorder → PO | 1 |
| | Branch indent → consolidated HO PO; central PO with branch-wise ship-to split | 2 |
| | RFQ / quote comparison, blanket order / rate contract, open-to-buy budget | 3 |
| **Receiving** | GRN (from PO / direct; billed vs received vs accepted; damaged/QC bucket); GRN label printing; DC Return (GRN rejection before the bill) | 1 |
| | Gate Inward + packages/bales | 2 |
| | Mobile GRN | 3 |
| **Billing** | Purchase Invoice (direct, from GRNs, from POs); shortage register → Shortage Receipt; cash purchase with tender; RCM self-invoice; hold / auto-save / attachments; price setting + labels | 1 |
| | Landed Cost Voucher (late freight bill); rate-difference / price-protection notes; supplier schemes and target claims | 2 |
| | Import (Bill of Entry, BCD/SWS/IGST), foreign currency | 3 |
| **Returns** | Purchase Return / Debit Note (bill-linked, batch-exact, reasons, claim receivable, replacement) | 1 |
| | Expiry / breakage claim cycle | 2 |
| **Special stock** | Consignment / sale-or-return (supplier-owned stock, settlement invoice); job work out/in + ITC-04 | 2 |
| **Statutory** | ITC tag per line + "ITC claimable" state | 1 |
| | GSTR-2B / IMS work-queue via GSP; Rule 37 (180-day) and 37A reversal/re-claim; MSME due dates + MSME-1; 393(1) TDS on purchase (at payment — exists) | 2 |
| **Payment side** | Payment 100 already settles PURCHASE bills (works as soon as bills exist) | — |
| | Payment-term slabs, cash discount / late charge proposed at payment, "pay by" planner | 2 |
| **Reports** | PO pending, GRN not billed, billed not received, shortage ageing, purchase register, HSN in/out, supplier outstanding / ageing, last-rate history, margin report | 1 |
| | Vendor rating, IMS status, ITC ledger, MSME dues | 2 |
| **Integration** | Import supplier Excel/JSON; e-invoice QR read + IRN check | 2 |
| | NexERP-to-NexERP (distributor sale → retailer purchase); supplier portal / ASN | 3 |

---

## 3. From 3.0 — features to KEEP that the base doc does not name

Source: `3.0/pos/Purchase/purchaseentry.cpp` (pe:) and the `fpurchase()` dump (sql:).

**Entry speed**
- **Typing Cost back-solves the rate:** rate = (cost − cess/unit) / (1 + (GST%+cess%)/100) (pe:1111). This is how the bill "as printed" is matched.
- Cells accept `a/b` and `a*b` (pe:1034).
- In a price cell, `10%` = MRP less 10% (pe:947).
- Line defaults (rate, discount, scheme) come from the **last purchase of that item**, or from the PO when one is loaded (sql ~765).
- Picking the supplier opens their **pending POs** and pops up **supplier notes** (pe:333).
- Side panels: last purchases of this item, and the supplier's bills.
- An unknown barcode → "save as a new EAN for this item?" (pe:614).
- Keys:
  - **Ctrl+F4** batch/godown split; **F12** batch popup.
  - **Ctrl+F11 / F12** return all / selected from this bill.
  - **Ctrl+F8** bill-wise payment; **Ctrl+F9** quick stock correction.
  - Alt+C / Alt+A create / alter a master from the grid.

**Calculation**
- Discount cascade per line: item % or per-qty or amount → scheme % on the remainder → additional discount amount.
- Header CD % or amount, with "affects tax" yes/no.
- Cess % plus cess per unit, plus additional cess.
- **GRN value ≠ supplier payable** (pe:1640):
  - own freight, unloading, agent commission and miller commission go into **item cost**, not into what we owe the supplier;
  - supplier freight and loading go into both.
- Landing cost absorbs free qty and shortage: cost ÷ (qty + free − short).
- **Selling price from each unit's profit type**:
  - By % / By ₹ / By User;
  - rounded to the item's round-off step;
  - old A/B/C kept on the line;
  - live bill profit ₹ and %.
- Unloading / commission / bata per BAG / QTL / TON by item weight.

**Controls**
- **Invoice Amount must equal Net Payable** before save (pe:1736).
- Duplicate supplier invoice no (case-insensitive, per supplier, per year).
- Price A ≥ cost unless allowed; prices ≤ MRP unless "skip MRP".
- The batch split must equal qty + free.
- Edit is refused once paid, warns if returned or consumed, and needs a reason when configured.
- Delete needs a reason and is audited.

**Settings that clients choose (keep as per-company settings)**
- `price_fixing`, one of:
  - update selling prices;
  - update cost only;
  - open Change Selling afterwards;
  - do nothing.
- Other settings: close the whole PO on first bill; agent mandatory; allow price below cost; default return mode (cash / adjust / advance); default entry type (direct / from gate inward); sticker print after save; textile mode.

**Second-tree ideas (2024) worth keeping**
- **Expense master** (`purchase_calc_master`): ADD/SUB; unit = % / qty / kg / qtl / ton; "consider in landing cost / purchase value / both"; allocate by qty / weight / value; own ledger, GST yes/no, separate posting, alternate party ledger.
  - NexERP's `charge_master` already has `chg_landing_cost`, `chg_cost_alloc`, `chg_before_tax`, `chg_sep_post`, `chg_tax_id` → **extend it, don't add a table.**
- Purchase Note = costed GRN pulling several POs, with "as received" and "as billed" copies of each line.
- Shortage register (`purchase_short_items`) closed by Shortage Receipts.

**Do NOT copy** (bugs found in 3.0)
- PO delivered qty is re-added on every re-save (no voucher filter, sql:373).
- Charges are spread **by qty only**; the value-based spread is commented out.
- Edit = delete + re-insert, and it **rewrites the prices of older lots with the same MRP**.
- Delete never checks whether the stock was sold.
- Return and PO use one GST ledger (no C/S/IGST split); no RCM, no purchase TDS.
- Hard-coded ledger codes. The CD ledger choice tests a pointer (always 100148).
- Auto-save to a local `purchase.txt`; an 86-column positional grid stored in the hold JSON.
- Orphan detail lines (76 lines without a header on the box).

---

## 4. From the market — flows the base doc misses

Must = phase 1 or 2 for our client base.

| # | Feature | Who has it | Why | Priority |
|---|---|---|---|---|
| 1 | **Three-way match + tolerance + payment block** (bill vs PO rate, bill vs GRN qty, % tolerance; an exception blocks payment) | Odoo, GoFrugal GoSure | Stops over-billing. Linking documents alone does not. | **1** |
| 2 | **GRN barcode labels**: one per piece, qty = received, design/size/colour/MRP | Ginesys, GoFrugal | Piece-barcode textile cannot sell without it | **1** |
| 3 | **Pre-pack / ratio-pack entry** (size ratio 1:2:2:1 × packs, design-wise PO) | Ginesys, Logic | Apparel is bought in sets; a matrix alone is slow | **1** (textile) |
| 4 | **Value-based GST rate** on the purchase line (apparel ≤ ₹2,500/pc → 5 %, above → 18 %, GST 2.0) | — | Wrong slab = wrong ITC and cost | **1** |
| 5 | **Multi-MRP / old-stock MRP change**: new MRP = new price bucket, old stock untouched | GoFrugal, Marg | 3.0 rewrote old lots — a bug, not a feature | **1** |
| 6 | **Cash purchase with tender** + ₹10,000/day cash warning | Busy, Marg | Cash-and-carry, mandi | **1** |
| 7 | **Composition / unregistered supplier mode** (bill of supply, no tax fields, notified-RCM check) | Tally, Busy | Small mills, weavers | **1** |
| 8 | **Minimum shelf-life % at GRN** (refuse near-expiry) | Marg, GoFrugal | Pharma / FMCG | **1** if those clients |
| 9 | **Consignment / SOR**: supplier-owned stock, not in our valuation; periodic settlement invoice for qty sold; return of unsold | Ginesys, Logic | Routine in textile/footwear. A "purchase type" is not enough: ownership differs. | **2** |
| 10 | **Job work out/in**: Rule 45 challan, fabric out → garment in (other item), process loss, job charge into cost, 1/3-year deemed-supply ageing, **ITC-04** | Tally, Ginesys | Stitching / dyeing / embroidery | **2** |
| 11 | **Branch indent → HO PO; central PO with branch ship-to split**, cross-dock to branches via the transfer screens already built | Ginesys, GoFrugal HQ | Chains buy centrally | **2** |
| 12 | **Rate-difference / price-protection notes**, GST (sec 34, needs sec 15(3)(b) pre-agreement) vs financial (Circ. 92/11/2019) | Tally, Busy | Getting it wrong reverses ITC wrongly | **2** |
| 13 | **Supplier target / turnover schemes** settled later (not invoice-linked) | Marg, Busy | Distribution margins | 2 |
| 14 | **Damage / write-off → ITC reversal** (17(5)(h)) from Stock Adjustment | — | The base doc stops at damage on receipt | **2** |
| 15 | **QC hold bucket**: receive into QUARANTINE, accept/reject later (`ck_svi_bucket` already has it) | Tally, Odoo | Separates gate damage from inspection rejection | 2 |
| 16 | **Supplier statement reconciliation** (import their ledger, match) | — | Monthly squaring | 2 |
| 17 | **Vendor rating** (on-time, fill rate, rejection %, rate variance) | Odoo, Ginesys | Supplier choice | 2 |
| 18 | **Fabric by roll/metre, catch-weight, repack bulk → loose** | Logic, Marg | Fabric and FMCG | 2 |
| 19 | **Inter-company / multi-GSTIN purchase** auto-created from the sister GSTIN's sale | Odoo, Tally | Same group, many states | 2 if multi-GSTIN |
| 20 | Imports + forex gain/loss | Tally, Odoo | Fabric / accessory imports | 3 |
| 21 | Expense / service / capital-goods purchase in the same flow (asset link, 60-month capital-goods ITC) | Tally | Blocked credits | 3 |
| 22 | Open-to-buy budget; RFQ; supplier portal / ASN | Ginesys, Odoo | Large chains | 3 |

---

## 5. Indian law — what the ERP must do (Oct 2026)

Confirm items marked **(CA)** with the client's CA before coding the rule. Every figure is read **per company through the statutory limits**, never written in code.

| Rule | What the ERP must do |
|---|---|
| **Sec 16(2)** | ITC only when: invoice held, **goods received (lots → last lot)**, in GSTR-2B and not restricted, supplier paid the tax, our return filed. Store an "ITC availed period" per invoice line. **The short qty of a bill carries no ITC until the Shortage Receipt.** |
| **Sec 16(4)** | Hard stop on claims after 30 Nov of the next FY (or the annual return, if earlier). |
| **Rule 37** | Supplier unpaid **180 days** from the invoice date → reverse that ITC (with interest) in the next return; re-claim on payment. Fed from AP ageing. |
| **Rule 37A** | Supplier filed GSTR-1 but not 3B by 30 Sep after FY end → reverse by 30 Nov; re-avail when they file. |
| **Rule 36(4)** | No provisional ITC: only what is in 2B. |
| **Sec 17(5)** | Blocked: motor vehicles, food/catering, club, works contract, personal use, **free samples / gifts**, goods **lost / destroyed / written off**. Decided per line by rule, not by a hand flag. |
| **Rule 42/43** | Common credit for mixed taxable/exempt supplies: monthly reversal, annual true-up; capital goods over 60 months. (CA) |
| **IMS (from Oct 2024; changed from the Oct 2025 period)** | Per supplier document: Accept / Reject / Pending; **no action = deemed accepted**; rejected docs leave 2B; actions allowed until 3B is filed, then 2B is recomputed. Credit notes can stay Pending **one period**. New declaration of the ITC amount to reverse on a CN. → an **IMS work-queue linked to our purchase docs, via the GSP**. |
| **GSTR-3B** | Outward liability hard-locked from the Jul-2025 period; 3-year bar on late returns. ITC-side locking signalled (unverified). |
| **Supplier e-invoice** | A supplier with AATO > ₹5 cr must give an IRN invoice → flag a B2B bill without IRN from such a supplier, validate the QR. Suppliers with AATO ≥ ₹10 cr cannot report a document to the IRP after **30 days** → a late CN from them will fail; the shortage / claim flow must chase CNs early. |
| **E-way bill, inward (Rule 138)** | Over ₹50,000. **We generate it when buying from an unregistered person.** Reject within 72 h or it is deemed accepted. No EWB on documents older than 180 days (from Jan 2025). |
| **RCM** | GTA, legal, security, director, sponsorship. **Commercial rent from an unregistered landlord** (from 10-Oct-2024; composition recipients excluded from 16-Jan-2025). **Metal scrap from unregistered sellers** (Ch 72–81, 10-Oct-2024). **Self-invoice within 30 days (Rule 47A)** + payment voucher (31(3)(g)). Time of supply: goods = earliest of receipt / payment / 31st day. |
| **GST-TDS 2 % (sec 51)** | On scrap bought from registered sellers over ₹2.5 lakh. |
| **GST 2.0 rates (22-Sep-2025)** | 5 / 18 / 40. 12 % and 28 % gone. Compensation cess nil (tobacco from 1-Feb-2026). **Rates are dated** — historical bills keep their rate. |
| **Job work (sec 143, Rule 45)** | Challan; return within 1 yr (inputs) / 3 yrs (capital goods) or it is a deemed supply; **ITC-04** half-yearly if AATO > ₹5 cr, else annual. |
| **TDS on purchase of goods** | Old 194Q is now **sec 393(1) Table Sl. 8(ii), Income-tax Act 2025** (from 1-Apr-2026): 0.1 % above ₹50 lakh per seller per FY, if our turnover > ₹10 cr last FY. Deducted at **payment** (exists: `tds_rates` + Payment 100). Section codes must be dated (194Q before 1-Apr-2026, 393 after). (CA for the no-PAN rate.) |
| **TCS 206C(1H), 206AB, 206CCA** | **Omitted from 1-Apr-2025.** No purchase-side TCS-on-goods logic. Supplier TCS on scrap / minerals / vehicles still exists → TCS receivable. |
| **MSME (old 43B(h) → sec 37(2)(g), IT Act 2025)** | **Micro and small only, manufacturers and service providers — traders excluded.** 15 days without a written agreement, 45 days max with one, counted from acceptance of the goods (= GRN date). Store Udyam no + type + category (exist on the ledger) + "written agreement" + acceptance date. MSME-1 half-yearly for companies with MSE dues > 45 days. |
| **Legal Metrology** | Pre-packs received must carry MRP, net qty, unit sale price, packer, month-year, consumer care → optional GRN checklist. |
| **Drugs & Cosmetics / FSSAI** | Supplier drug licence + expiry; batch + expiry mandatory; buy only from licensees. FSSAI licence + validity. (Only for those clients.) |
| **Audit trail** | Companies (Accounts) Rules 3(1): every edit logged with who/when/before/after, **cannot be disabled** → purchase screens use the existing `log_rev_no` snapshots from day one. Records kept 72 months (CGST s.36) / 8 years (Companies Act s.128(5)). |

**Base doc items that are legally wrong or out of date:**
- "TDS 194Q" → sec 393(1).
- "Supplier TCS" (206C(1H)) is gone.
- "MSME 45-day" → 15/45 days, micro/small only, traders excluded.
- "GSTR-2B reconciliation" alone → IMS actions are now the control.
- Damage stops at receipt → write-offs reverse ITC.
- Consignment as a "purchase type" → wrong ownership: no ITC and no stock value until the settlement invoice.

---

## 6. NexERP platform defects purchase will hit (fix first)

Verified on .106 today.

| # | Finding | Why it blocks purchase | Fix |
|---|---|---|---|
| P1 | **`ux_avh_voucher_slno`** is UNIQUE on (company, acc_year, voucher_slno) — company-wide, across branches | Two offline branches posting purchase bills both mint slno N → the second sync aborts | Include branch + device (as the stock numbering does), or derive slno at the centre only |
| P2 | **`ck_abl_settled`**: alloc + disc + writeoff ≤ bill_amount on `acc_bill_balance` | A supplier bill paid at HO and adjusted by a return at an offline branch → correct data fails at sync. A CHECK on a contended total breaks [offline-sync-invariants] | Drop it → overpaid partial index + report (same pattern as the six loyalty CHECKs) |
| P3 | `stock_item_cost.sic_last_purchase_rate` is taken from **any** inward (opening, transfer in, sale return, adjustment +) | "Last purchase rate" — the most-used purchase default — would be wrong | Filter to `sml_txn_type = 'PURCHASE'` |
| P4 | The sales shadow stock writer never sets `svh_supplier_id` / `svi_supplier_id` / `svi_landed_rate` / `svi_barcode`, puts the **doc rate into `svi_sale_price` (part of the lot key)**, and never calls `assignAutoBatchNumbers` / `syncBatchDetails` | Purchase cannot reuse it as is; textile batches and size/colour would not be written | A purchase shadow writer (sibling of `sales-stock.service.ts`) |
| P5 | `item_price_change_log.ipcl_source` CHECK has no `PURCHASE` | Price setting from purchase cannot be logged | Add `PURCHASE` (+ `PURCHASE_RETURN`?) |
| P6 | `txn_status_log`, `txn_transport_detail.ttd_doc_type` CHECKs lack GRN / DEBIT_NOTE / DC_RETURN purchase types | Status trail and transport band refuse purchase docs | Extend the CHECKs |
| P7 | Voucher types: no purchase-bill type that is BOTH accounts + inventory, no purchase-return type; `gstDoc()` maps DEBIT_NOTE to OUTWARD | A supplier debit note would land in GSTR-1 instead of the inward side | New types + INWARD mapping with sign −1 |
| P8 | Missing ledger roles: GRNI (goods received, bill pending), SHORTAGE_RECEIVABLE, CLAIM_RECEIVABLE, ITC_INELIGIBLE / ITC_REVERSAL, INPUT_TAX_PENDING (not yet in 2B), TCS_RECEIVABLE, RCM_CESS_PAYABLE, PURCHASE_PRICE_VARIANCE (or reuse RATE_DIFFERENCE), IMPORT_DUTY | Postings cannot be mapped | Seed roles + ledger map |
| P9 | Accounts post INVENTORY at `sml_cost_value`, which is **with tax** | With ITC, Stock-in-Hand would include recoverable GST that is also debited to Input GST → double count | Purchase states `svi_cost_rate` = landed cost **net of eligible ITC**; gross only on ineligible lines (decision D4) |
| P10 | The textile tag routes (`/textile-tags/*`) are called by the Qt client but do not exist; no `ilt_*` table. Meanwhile auto batch + `stock_batch_detail` (size/colour/model) is **live** on Opening (notes-149) | Two textile designs | **Purchase uses the live auto-batch + batch-detail path.** The 09-27 tag-table plan is superseded for the batch/label part (decision D9). Also fix notes-149's open items (a cancelled draft leaves its sbd row live). |
| P11 | `stock_lot` first-writer cost/source columns are overwritten by the cloud-sync PK upsert; `txn_charge_detail.cd_carried_amt` is incremented | PO→GRN→bill charge carry and first-receipt cost can drift between branch and centre | Rebuild at the centre; carry = derived, not incremented |
| P12 | `item_master.item_supplier_id` has no FK | Preferred supplier can point at nothing | FK, or move it to the supplier–item map |

---

## 7. Base doc vs house rules (change before building)

| Base doc | House rule | Change |
|---|---|---|
| "extend audit trigger" for the price log | No DB functions/triggers | The NestJS `PriceChangeLogService` exists; purchase is one more writer (after P5) |
| `pur_hold` | `public.txn_hold` already lists PURCHASE_* | Reuse |
| Shipping fields as new Gate Inward columns | `txn_transport_detail` + crew/vehicle on the doc | Reuse; add only e-way no, packages, seal, weight, freight paid/to-pay |
| 10 qty buckets on every line | One home per fact | Ordered on the PO line, received/accepted/rejected on the GRN line, billed on the invoice line; pending = through `pur_doc_link`; at most a rebuilt roll-up |
| Partition "where volume warrants" | Every txn table partitioned by acc_year, composite PK | All of them |
| Unique (supplier, inv no, year) index | No unique constraint on what two offline branches can both create | **Offline is possible (user)** → refuse at save when online (the register's `liveInwardHolderOf` check already exists), plus a duplicate-detect report; the index stays non-unique |
| Price change → "label print queue" table | — | A queue is new; fine. It must be rebuilt from GRN lines, never incremented |
| `pur_invoice_item_alloc` | Stock docs use split rows (`svi_split_no`) | Keep **alloc** for purchase: the invoice line must mirror the supplier's printed line (HSN, rate, tax) for 2B/IMS; the alloc rows are the stock side. The only doc type built this way — documented. |

---

## 8. Masters — gaps

- **Supplier / ledger** (supplier id = ledger id). Already present: GST reg type, ITC eligibility, reverse charge, TDS deductee + nature, Udyam no, MSME type, PAN, SEZ, bill-by-bill, bank, lower-deduction certificates, credit days, cash disc %. **Add:**
  - MSME category (micro/small/medium) + activity (manufacturer/service/trader) + written-agreement flag (for the 15/45-day rule);
  - default payment term;
  - import/overseas + currency;
  - default godown;
  - drug licence + expiry and FSSAI + validity (the table has a drug licence column; it is not on the form);
  - supplier notes popup flag.
  - `purchase.suppliers` duplicates ledger facts behind a copy map. Leave it, but never add new duplicated facts.
- **Supplier–item map (new):** supplier item code / barcode, purchase unit and pack, last rate per supplier, lead time, MOQ, preferred, active. Replaces `item_supplier_id`.
- **Item:**
  - purchase default unit;
  - cash-discount eligible;
  - ITC class override (blocked / ineligible);
  - minimum shelf-life %;
  - shortage % (3.0 `shortage_qty`).
  - Profit type + round-off step: check `item_price_master` covers them, else add.
- **Payment terms (new):** term, basis date (invoice / GRN / LR), slabs (days from/to, DISCOUNT | CHARGE, %, basis).
- **Expense master:** `charge_master` module 'P' + the 3.0 expense attributes (unit basis kg/qtl/ton, landing cost vs purchase value vs both, alternate party ledger).

---

## 9. Design decisions I am locking (stock/costing side is mine)

| # | Decision |
|---|---|
| D1 | **Costing** stays as built: WAVG for plain items, lot cost for tracked (notes-92). Backdated purchases are handled by the ledger rebuild. Posted COGS is never re-costed; the difference stays in stock value. |
| D2 | **Free qty** shares the lot cost (one lot, one cost) → cost is spread over qty + free. Not a setting. |
| D3 | **Stock in** = a purchase shadow voucher, RECEIPT with `ledgerTxnTypes ['PURCHASE']`; return = ISSUE `['PURCHASE_RETURN']`; GRN rejection = ISSUE. No new stock voucher types. |
| D4 | **Stock value** = landed cost net of eligible ITC. Ineligible / blocked / composition / unregistered GST goes into cost. Own-side charges (own freight, unloading, commissions) go into cost and not into the payable. |
| D5 | **GRN posts stock** (Dr INVENTORY, Cr GRNI) at the PO rate, or the last purchase rate when there is no PO. **The invoice clears GRNI** at GRN value: the rate difference goes to INVENTORY for qty still on hand and to PURCHASE_PRICE_VARIANCE for qty already sold. A direct invoice (no GRN) = one step, no GRNI. |
| D6 | **Shortage**: stock in = received qty only. Value of the short qty → SHORTAGE_RECEIVABLE; its GST → INPUT_TAX_PENDING (no ITC until received). Outcome chosen at invoice: Shortage Receipt / supplier note / claim / write-off. The same pattern applies to excess and damage on receipt. |
| D7 | **Duplicate supplier bill**: online refusal (the register check) + detect report; no unique index (offline). |
| D8 | **Numbering**: every purchase doc numbered per device with `dev_code` (like stock: `GRN/2026-2027/D1/00001`). Accounts voucher slno must not be company-wide (P1). |
| D9 | **Textile**: auto batch + `stock_batch_detail` (size/colour/model), one label per piece; ratio-pack entry fills the matrix. |
| D10 | **Old stock is never repriced by a purchase.** A new MRP or selling price = a new price bucket / lot; existing lots keep theirs (3.0 bug). |
| D11 | **Edit** after post: allowed while nothing is paid, returned or consumed. Otherwise cancel + re-enter or a note. A cancel adds reversing rows, never deletes. Snapshots via `log_rev_no`. |
| D12 | **Three-way match**: per-company tolerances (rate %, qty %). An exception marks the bill "hold for payment"; Payment 100 warns / refuses. |

---

## 10. Questions for the user (business only)

1. **Which client types go first?** Textile retail / FMCG distribution / pharma / hardware. This decides whether shelf-life, drug licence, catch-weight and imports are phase 1.
2. **Consignment / sale-or-return** — used by your textile clients today? (Phase 2 unless yes-urgently.)
3. **Job work** (send fabric for stitching/dyeing) — used? (Phase 2.)
4. **Central purchase**: does HO buy for branches (one PO, delivery per branch)? Or does each branch buy its own?
5. **Same group, several GSTINs** buying from each other?
6. **Price setting default** — of 3.0's four `price_fixing` modes, which do most clients use? (All four stay as a setting; this picks the default.)
7. **Cash discount**: do suppliers give "pay in 3 days, 5 % off"? GST credit note or plain financial note? (Your CA.)

---

## 11. Next

1. User answers §10.
2. I write **plan-purchase-phase1.md** (separate file):
   - DDL for the `purchase` schema;
   - routes under `/purchase/*`;
   - posting legs;
   - the fixes P1–P12 that phase 1 needs, as their own notes file;
   - mockups: PO, GRN, Purchase Invoice, Return, Shortage Receipt, supplier–item map.
3. Prathap builds the server; Qt follows the mockups.

Evidence: 3.0 function dumps are in the session scratchpad (fpurchase.sql etc.).
