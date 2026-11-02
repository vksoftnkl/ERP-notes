# Purchase — Phase 1 plan (rev 1)

Date: 2026-10-27. From: Viji / Claude. For: the user, then Prathap.

Builds on `analysis-purchase-base-2026-10-27.md` (same folder). That file holds the evidence: 3.0, market, law, NexERP. This file holds **what we build and in which phase**.

**Rule for this file:** §2 is the master register. Every feature from the user's base doc, the analysis, 3.0 and the discussion has a line there, with its phase or "dropped". When we come back to purchase later, start from §2 — nothing exists only in chat.

Platform fixes needed first: `notes-162-purchase-platform-prereqs.md` (same folder).

---

## 1. Decisions log (user + mine)

| # | Date | Decision | By |
|---|---|---|---|
| U1 | 27-10 | First client type = **FMCG** (batch, expiry, MRP, schemes, claims) | user |
| U2 | 27-10 | **No consignment / sale-or-return; no job work** | user |
| U3 | 27-10 | HO purchase is followed by **Branch Outward** (existing screens). Branches also buy. No ship-to split on the PO. | user |
| U4 | 27-10 | **Inter-company**: one company's sale bill becomes another company's purchase (multi-company, one DB) | user |
| U5 | 27-10 | Price from purchase: **manual rate or markup % only** (3.0's By % / By ₹ / By User dropped). Keep 3.0's price-update modes as a setting. | user |
| U6 | 27-10 | **Main risk = amend**: check payment first; an old purchase edit must not change newer rates | user |
| U7 | 27-10 | **No fixed freight / loading fields.** Everything through the ONE charge master. "Keep it simple." | user |
| U8 | 27-10 | No cash-discount slabs / payment terms for now (maybe later) | user |
| U9 | 27-10 | Charge master gets **"On supplier bill"** (into supplier balance) vs landing-cost-only. The latter still posts a payable, paid later. | user |
| U10 | 27-10 | Markup % on cost **without tax**; price change from purchase needs a user right | user |
| U11 | 27-10 | **GRN-first and direct bill both allowed**; company default | user |
| U12 | 27-10 | Coolie owed **loadman-wise**; rate **from the charge master** | user |
| U13 | 27-10 | Gate Inward + package/bale tracking → **phase 2** | user |
| U14 | 27-10 | FMCG **expiry / breakage claims from the start**, worked out per return row | user |
| U15 | 27-10 | Goods arrive before bills (GRN phase 1); purchase may be keyed at **offline** branches; **textile in phase 1**; **PO in phase 1** | user |
| D1–D12 | 27-10 | Costing, free qty, stock shadow voucher, stock value net of ITC, GRNI, shortage, duplicate guard, numbering, textile path, no repricing of old stock, edit rules, three-way match. Text in analysis §9. | Claude |
| D13 | 27-10 | An **off-bill charge is always landing cost.** A cost that is neither on the bill nor in item cost belongs on an Expense Voucher (277). | Claude |
| D14 | 27-10 | An **off-bill charge carries no GST in phase 1.** A transporter's own GST bill / GTA RCM = phase 2 (Landed Cost Voucher). | Claude |
| D15 | 27-10 | **Loadman payable posts to the employee's own ledger** (`employee_master.emp_loan_ledger_id`, a PARTY ledger), bill-wise per document. Payment 100 settles it. | Claude |
| D16 | 27-10 | **Price-row rule**: a purchase writes a price row only if no logged change to that row is newer *in business time*. Business time = doc date-time for a purchase, change time for everything else. | Claude |
| D17 | 27-10 | **Purchase Return = one document with a per-row outcome**: Return (debit note) / Claim-sent / Claim-destroyed / Replace. GRN rejection before the bill is the same document, kind "against GRN". | Claude |
| D18 | 27-10 | **ITC is reversed when the return or claim is posted**, not when the supplier's credit note arrives. The CN is matched later (IMS, phase 2). | Claude |
| D19 | 27-10 | **A GRN-to-invoice rate difference needs stock revaluation.** Engine prerequisite: value-only revaluation rows (notes-162 P13). | Claude |

---

## 2. Master register — every feature and its phase

Phase 1 = this plan. Phase 2 = next plan. Phase 3 = later. ✗ = dropped (reason given).

### 2.1 Documents

| ID | Feature | Phase | Source |
|---|---|---|---|
| DOC-01 | Purchase Order: pending qty, clone/merge pending, short close, amendment revision, print/PDF | **1** | base, 3.0 |
| DOC-02 | Reorder screen → PO; pending POs auto-offered when the supplier is picked | **1** | 3.0 |
| DOC-03 | GRN, from PO(s) or direct: received / accepted / damaged per line, batch alloc, labels | **1** | base, U15 |
| DOC-04 | Purchase Invoice: direct, from GRN(s), from PO(s) | **1** | base |
| DOC-05 | Shortage register + Shortage Receipt (also receives replacements) | **1** | base, 3.0 2nd tree |
| DOC-06 | Purchase Return / Claim / Replace, per row; against invoice or against GRN (rejection before bill) | **1** | base, U14, D17 |
| DOC-07 | Claim settlement: supplier CN received, short-settled → write-off | **1** | U14 |
| DOC-08 | Inter-company: load a sister company's sale bill into a purchase | **1** | U4 |
| DOC-09 | Cash purchase with tender at entry; ₹10,000/day cash warning | **1** | market |
| DOC-10 | RCM self-invoice (unregistered notified goods/services) + payment voucher | **1** (basic) | law |
| DOC-11 | Gate Inward + packages / bales; LR, e-way bill, seal | 2 | base, U13 |
| DOC-12 | Landed Cost Voucher: a later transporter / GTA bill re-spread onto a GRN or invoice | 2 | base, D14 |
| DOC-13 | Rate-difference / price-protection notes (GST note vs financial note) | 2 | market |
| DOC-14 | Supplier target / turnover scheme claims | 2 | market |
| DOC-15 | Branch indent → HO PO | 2 | market |
| DOC-16 | Import purchase (Bill of Entry, BCD/SWS/IGST), foreign currency | 3 | market |
| DOC-17 | Expense / service / capital-goods purchase in the same flow | 3 | market |
| DOC-18 | RFQ / quote comparison, blanket orders | 3 | base |
| DOC-19 | Consignment / SOR | ✗ (U2) | market |
| DOC-20 | Job work out/in, ITC-04 | ✗ (U2) | market |
| DOC-21 | Central PO with branch ship-to split | ✗ (U3: Branch Outward instead) | market |

### 2.2 Entry features

| ID | Feature | Phase |
|---|---|---|
| ENT-01 | Supplier pick → credit days, CD % default, pending POs, **supplier notes popup** | **1** |
| ENT-02 | Line defaults from the last purchase of the item (from the PO when one is loaded) | **1** |
| ENT-03 | Side panel: last purchases of this item (all suppliers) + this supplier's bills | **1** |
| ENT-04 | **Typing Cost back-solves the rate** (tax-inclusive entry); typing Gross back-solves the rate | **1** |
| ENT-05 | `a/b`, `a*b` arithmetic in rate / cost cells | **1** |
| ENT-06 | Discount cascade: item % or per-qty or amount → scheme % → extra discount amount | **1** |
| ENT-07 | Free qty; supplier scheme "buy N get M" fills free qty | **1** |
| ENT-08 | Cess % + cess per unit + additional cess | **1** |
| ENT-09 | Value-based GST slab (apparel ≤ ₹2,500/pc 5 %, above 18 %) | **1** (needs P15) |
| ENT-10 | Local vs inter-state from supplier state vs company state; re-split on supplier change | **1** |
| ENT-11 | Composition / unregistered supplier: bill of supply, no tax fields | **1** |
| ENT-12 | Batch / godown / MRP / expiry / mfg allocation popup (F12), sum = qty + free | **1** |
| ENT-13 | Textile: auto batch + size / colour / model; **ratio-pack** entry (1:2:2:1 × packs) | **1** |
| ENT-14 | Serial numbers | 2 |
| ENT-15 | Minimum shelf-life % at GRN (warn / refuse) | **1** |
| ENT-16 | Unknown barcode → "save as a new EAN for this item?" | **1** |
| ENT-17 | Purchase unit (box/case) with conversion; supplier-item map supplies the default unit | **1** |
| ENT-18 | Price setting: manual or markup %; old prices kept on the line; live bill margin | **1** |
| ENT-19 | `10%` in a price cell = MRP less 10 % | **1** |
| ENT-20 | Multi-MRP: new MRP = new price bucket / lot; old stock untouched | **1** |
| ENT-21 | Labels after GRN / invoice (qty = received; one per piece for textile) | **1** |
| ENT-22 | Invoice Amount (typed from the paper bill) must equal Net Payable; setting WARN / REFUSE | **1** |
| ENT-23 | Duplicate supplier bill no: refuse online + detect report (offline-safe) | **1** |
| ENT-24 | Three-way match: rate / qty tolerance → "hold for payment" | **1** |
| ENT-25 | Hold / resume (txn_hold); server auto-save draft | **1** |
| ENT-26 | Attachments (bill scan, damage photo) | **1** (needs P16) |
| ENT-27 | Keys: TxnKeys standard + F12 alloc, F4 unit, Ctrl+F11/F12 return from bill, Ctrl+F8 pay this bill | **1** |
| ENT-28 | Prev / Next, History (audit snapshots) | **1** |
| ENT-29 | Copy bill to another company | ✗ (replaced by DOC-08) |
| ENT-30 | E-invoice QR scan → header fill + IRN check | 2 |
| ENT-31 | Import supplier Excel / JSON | 2 |
| ENT-32 | Mobile GRN | 3 |
| ENT-33 | Catch-weight / fabric by roll / repack bulk → loose | 2 |
| ENT-34 | QC hold bucket (receive into QUARANTINE, accept later) | 2 |

### 2.3 Charges (one charge master — U7)

| ID | Feature | Phase |
|---|---|---|
| CHG-01 | Purchase charges from `charge_master` (module P / B) only; no fixed fields | **1** |
| CHG-02 | New flag **On supplier bill** (U9) | **1** |
| CHG-03 | Off-bill charge = landing cost, credited to a party / ledger, paid later (D13) | **1** |
| CHG-04 | **Party Mandatory** made real: party per charge row on the document | **1** |
| CHG-05 | Coolie / unloading split **per loadman** (U12, D15) | **1** |
| CHG-06 | Spread by value / qty / weight (never qty-only — 3.0 bug) | **1** |
| CHG-07 | Charges on PO (estimate), GRN (own labour), invoice (supplier's charges) | **1** |
| CHG-08 | Off-bill charge with GST / GTA RCM | 2 (D14) |
| CHG-09 | Align transfer loading (one LOADING_PAYABLE) with the loadman-wise model | 2 |

### 2.4 Statutory

| ID | Feature | Phase |
|---|---|---|
| LAW-01 | ITC state per invoice line: eligible / ineligible / blocked 17(5) / RCM / pending (short qty) | **1** |
| LAW-02 | No ITC on short qty until received (16(2)(b)) | **1** |
| LAW-03 | ITC reversed on return / claim / destroyed (17(5)(h)) | **1** |
| LAW-04 | E-way bill inward: we generate it when buying from an unregistered person | 2 |
| LAW-05 | GSTR-2B + IMS work-queue (accept / reject / pending) via GSP | 2 |
| LAW-06 | Rule 37 (180-day unpaid → reverse; re-claim on payment); Rule 37A | 2 |
| LAW-07 | Sec 16(4) claim deadline guard | 2 |
| LAW-08 | MSME: micro/small, manufacturer/service, written agreement → 15/45-day due date; MSME-1 report | **1** fields, 2 report |
| LAW-09 | TDS on purchase of goods 393(1) (old 194Q) | exists (Payment 100) — dated section codes **1** |
| LAW-10 | Supplier e-invoice: flag a missing IRN from AATO > ₹5 cr suppliers | 2 |
| LAW-11 | GST-TDS 2 % on scrap from registered sellers | 3 |
| LAW-12 | Rule 42/43 common credit | 3 |
| LAW-13 | Drug licence / FSSAI on supplier (pharma / food clients) | **1** fields only |
| LAW-14 | Legal Metrology checklist at GRN | 3 |
| LAW-15 | Audit trail (edit log cannot be disabled) via `log_rev_no` snapshots | **1** |

### 2.5 Masters

| ID | Feature | Phase |
|---|---|---|
| MST-01 | Supplier fields: MSME category + activity + written agreement, default godown, drug licence + expiry on the form, FSSAI + validity, import/overseas, notes popup | **1** |
| MST-02 | Supplier–item map: supplier code / barcode, purchase unit + pack, last rate per supplier, lead time, MOQ, preferred | **1** |
| MST-03 | Item: purchase default unit, minimum shelf-life %, ITC class override, shortage % | **1** |
| MST-04 | Charge master: On supplier bill (+ the existing flags) | **1** |
| MST-05 | Purchase agent master + commission (as a charge) | **1** |
| MST-06 | Payment terms + slabs | ✗ for now (U8) |
| MST-07 | Vendor rating | 2 |

### 2.6 Reports

| ID | Report | Phase |
|---|---|---|
| RPT-01 | PO pending (item / supplier) | **1** |
| RPT-02 | GRN not billed (= GRNI balance) | **1** |
| RPT-03 | Billed not received / shortage pending ageing | **1** |
| RPT-04 | Claims pending (supplier, age, value) | **1** |
| RPT-05 | Purchase register (+ HSN in / out) | **1** |
| RPT-06 | Supplier outstanding / ageing (exists in accounts — check purchase bills appear) | **1** |
| RPT-07 | Last purchase rate history / item purchase history | **1** |
| RPT-08 | Loadman coolie due (by loadman, by document) | **1** |
| RPT-09 | Duplicate supplier bill detection (offline) | **1** |
| RPT-10 | Three-way match exceptions / bills on payment hold | **1** |
| RPT-11 | Purchase margin (cost vs selling at entry) | **1** |
| RPT-12 | IMS status, ITC ledger, MSME dues, vendor rating | 2 |

---

## 3. Phase 1 — the documents

### 3.1 Purchase Order (DOC-01, 02)

- **Header:** supplier, PO no (device series), date, expected date, godown, agent, remarks, charges (estimate only, posts nothing).
- **Lines:** item, unit, qty, free, rate, discounts (ENT-06), GST % (split shown), cess, MRP, expected selling, remarks.
- **Status per line** (derived through the link table, never stored as a counter): pending = ordered − Σ received on GRN / invoice lines linked to it − short-closed.
- **Short close:** per line or the whole PO, with a reason.
- **Amendment:** allowed while nothing is received. After that, only an increase in qty, or a rate change on the pending part; each save = a revision (snapshot).
- **Clone / merge:** a new PO from the pending qty of one or more POs (3.0).
- Reorder screen → PO. Picking a supplier on a GRN or invoice offers that supplier's pending POs.
- Posts nothing (no stock, no accounts).
- **Do not copy 3.0:** delivered qty re-added on every re-save.

### 3.2 GRN (DOC-03)

- **Header:**
  - supplier, GRN no, date-time;
  - supplier invoice no / date **if known** (optional — goods may come before the bill);
  - godown, transport (shared band), crew (loadmen = employees, dd 72);
  - remarks, attachments.
- **Lines** (from PO lines or typed):
  - billed qty (if the challan / bill states it), received qty, damaged qty (→ DAMAGED bucket), free;
  - rate (PO rate, else last purchase rate — **provisional**);
  - alloc popup (batch / MRP / expiry / godown / size / colour);
  - selling price / markup (ENT-18, applied at GRN or at invoice — company setting).
- **Charges:** off-bill only (own labour, own freight) — CHG-03 / CHG-05.
- **Shelf-life check** (ENT-15).
- **Posts:**
  - **Stock:** received + free, into SALEABLE; damaged into DAMAGED.
  - **Accounts:** Dr INVENTORY / Cr **GRNI** at provisional value. Off-bill charges: Dr INVENTORY / Cr loadman / party.
  - **Shortage:** a challan qty > received qty opens a shortage row (§3.4).
- Labels printed from GRN lines (ENT-21).

### 3.3 Purchase Invoice (DOC-04, 08, 09)

**Sources:**
- direct (typed; stock comes in here);
- from GRN(s): lines carried, stock already in;
- from PO(s): lines carried, stock comes in here;
- from a sister company's sale bill (DOC-08).

**Header:**
- supplier, supplier invoice no + date (required), our PUR no;
- **Invoice Amount as printed** (ENT-22);
- credit days → due date; cash / credit (cash = tender rows, DOC-09);
- reverse charge (auto from supplier + item / service);
- place of supply; agent; transport; remarks; attachments.

**Lines:**
- billed qty, received qty (= billed unless short, direct mode), free, rate;
- discounts, scheme, taxes, cess;
- ITC state (LAW-01); price setting; alloc (direct mode).
- **Three-way match:** against the PO rate and the GRN qty → tolerance → "payment hold" flag + reason.

**Charges:**
- on-bill (supplier freight, supplier CD): into the payable;
- off-bill (if not already on the GRN).

**Inter-company (DOC-08):**
- pick the sister company + its sale bill. The lines come with item, batch, MRP, expiry and rate.
- The supplier = the ledger that has the seller's GSTIN.
- One sale bill → at most one purchase (link table + refusal).
- Items must be shared, or mapped by item code.

**Posts:** see §5.

### 3.4 Shortage register + Shortage Receipt (DOC-05)

- **A shortage row opens when:**
  - an invoice bills more than was received; or
  - a GRN challan states more than arrived; or
  - a return row's outcome is Replace.
- Each row holds supplier, source document / line, item, qty, value, GST pending, reason, status.
- **Outcome**, chosen on the invoice or later:
  - Supplier sends later → **Shortage Receipt**;
  - Supplier credit note → close against the supplier;
  - Claim → transporter / insurer (claim receivable);
  - Write-off.
- **Shortage Receipt:** pick the open rows of a supplier → receive qty (alloc popup) → stock in at the original invoice cost. Clears SHORTAGE_RECEIVABLE and releases the pending GST.
- The open qty is derived (opened − received − closed), never incremented.

### 3.5 Purchase Return / Claim / Replace (DOC-06, 07) — U14, D17

**One document. Kind:**
- **against invoice**: normal; or
- **against GRN**: rejection before the bill; no GST; clears GRNI.

Rows come from the source bill or GRN. **The batch is exact** (the lot it came in as). Qty ≤ billed − already returned.

**Per-row outcome:**

| Outcome | Goods | Money | ITC |
|---|---|---|---|
| **Return** (debit note) | Out now | Supplier balance reduced now | Reversed now |
| **Claim — goods sent** | Out now | Claim receivable; credit expected later | Reversed now (D18) |
| **Claim — destroyed here** (breakage, expiry destroyed on proof) | Written off here | Claim receivable | Reversed (17(5)(h)) |
| **Replace** | Out now | None | Reversed now; re-claimed when the replacement arrives |

A Replace row opens a shortage row → Shortage Receipt.

**Reasons** (from Stock Reasons 293, purchase-return set): expiry, breakage, damage, wrong item, excess, quality, rate.

**Settlement (when not cash):**
- adjust in the next bill;
- adjust in Payment (PURCHASE_RETURN is already a held debit type);
- cash refund;
- advance.

**Claim settlement (DOC-07):** a claim row shows supplier, value, age. Actions:
- **Credit received** (amount, CN no / date) → Dr supplier / Cr CLAIM_RECEIVABLE;
- **Short-settled** → the difference to WRITE_OFF;
- **Rejected** → write-off, with a reason.

Part settlements are allowed. Each settlement is a row (open amount derived).

### 3.6 Charges (CHG-01…07)

**Charge master, purchase rows.** Existing fields:
- method (FIXED / PERCENT / QTY / NET_QTY / KG / QTL / TON), type ADD / DEDUCT, apply-on;
- default rate, before tax, tax applicable + rate;
- ledger, separate posting, party mandatory;
- landing cost + cost allocation (VALUE / QTY / WEIGHT), auto apply.

**New: `chg_on_bill` boolean default true.**

| On bill | Ledger means | Into cost | Payable | GST |
|---|---|---|---|---|
| yes | where it posts (if not absorbed) | if Landing Cost | supplier (in the bill total) | per the existing before / after-tax rules |
| no | **who we owe** when no party is picked (e.g. "Coolie payable") | **always** (D13) | the party on the row (Party Mandatory), else the ledger | none in phase 1 (D14) |

**Coolie per loadman (CHG-05):**
- the charge row (rate from the master, method QTY / KG / QTL / TON / FIXED) gives the amount;
- the loadmen picked on the document are the payees, split **equally by default and editable per loadman on the row**;
- stored as one charge-detail row per payee (`cd_party_id`);
- posted Cr each loadman's ledger (D15) with a bill-wise reference per document;
- report RPT-08; paid in Payment 100.

**Spreading to lines:** by the charge's cost allocation (VALUE default). The rounding residue goes to the last line. Never qty-only for mixed items.

### 3.7 Pricing from purchase (ENT-18…21, U5, U6, U10, D10, D16)

- **Company setting `purchase.price_update`** (3.0 `price_fixing`):
  1. UPDATE_PRICES — selling prices and cost;
  2. COST_ONLY;
  3. OPEN_CHANGE_SELLING after save;
  4. NONE.
- **Per line, two ways only:**
  - **Manual**: type the selling price / MRP;
  - **Markup %**: selling = cost **without tax** × (1 + markup %), rounded by the item's round-off step.
  - Margin % is shown beside it.
- Price cells are editable only with the **price-change right** (or on a brand-new item with no price).
- **Price ≥ cost** unless `inventory.below_cost_price` allows it; **≤ MRP** unless `inventory.skip_mrp`.
- **D10:**
  - a new MRP = a new price bucket / lot;
  - existing lots and buckets are never repriced by a purchase.
- **D16 — the old-edit guard:**
  - before writing a price row, read its Price Change Log;
  - if any change to that row is newer in business time than this purchase (a later purchase's doc time, or any manual / Change Selling / Quick Rate / item card change time), **skip it**;
  - the line shows "Price not updated — changed later on 25-10 by Change Selling (VKPOS)";
  - the purchase line keeps its own markup / price as history;
  - the same check runs for a backdated new purchase and for an edit.
- **Every price write** goes through the Price Change Log, with source PURCHASE (P5).
- **Labels:** after save, the changed or new price rows → label print (qty = received + free).

### 3.8 Amend / cancel (U6, D11)

| State of the document | What may change |
|---|---|
| Nothing paid / adjusted / returned / sold from it | everything (a new revision snapshot) |
| Paid or adjusted (any allocation on its bill-wise row) | **only** remarks, transport, attachments, supplier invoice date typo. Money fields locked — undo the payment allocation first, or use a note. |
| Returned, or stock sold from its lots | qty not below returned + sold; a rate change → rate-difference note (phase 2; until then refuse) |
| GRN already invoiced | GRN locked; amend the invoice |
| Cancel | Same checks as qty-to-zero. Reversing rows; nothing deleted. |

- Each save writes a `log_rev_no` snapshot (LAW-15).
- The price guard (D16) runs on every save.
- The edit reason is required when `accounts.allow_posted_amend` is on (the existing setting).

### 3.9 Textile in phase 1 (ENT-13, U15, D9)

- **Auto batch** (`stock.auto_batch_no`) + `stock_batch_detail` (size / colour / model), the path already live on Opening Stock (notes-149).
- **Ratio-pack entry:** design + sizes + ratio + packs → lines / allocations generated.
- **Labels:** one per piece, with batch = barcode, design / size / colour / MRP / price.
- **Selling price and MRP live on the lot** (resolveLots tracks MRP + sale price for the textile preset).
- The 09-27 tag-table plan is **superseded** for the batch / label part.
- notes-149's open items (a cancelled draft leaving its batch-detail row live) are fixed first (P10).

---

## 4. Calculation (one engine for PO, GRN, invoice, return)

**Per line, in order:**
1. **Gross** = qty × rate.
   - Typing Cost back-solves the rate: rate = (cost − cess/unit) / (1 + (GST% + cess%)/100).
   - Typing Gross back-solves rate = gross / qty.
2. **Discounts:** item (% | per-qty × qty | amount) → scheme % on the remainder → extra discount amount = **net gross**.
3. **On-bill before-tax charges** (spread) → **taxable**.
4. **GST:** the rate is chosen by date and, for value-slab items, by value per piece. CGST + SGST or IGST by place of supply. Cess % + cess per unit + additional cess.
5. **Line total** = taxable + taxes + on-bill after-tax charge share.

**Bill:**
- Σ line totals + on-bill after-tax charges (with their own GST if tax-applicable) ± round off = **Net Payable**;
- compare with the **Invoice Amount as printed** (ENT-22).

**Landed cost per base unit** (stock value, D4):
- numerator: taxable + on-bill landing-cost charges + off-bill charges + **ineligible GST only**;
- divided by (received qty + free) in base units;
- the short qty is excluded (its value sits in SHORTAGE_RECEIVABLE).

**Markup / margin:**
- markup = (sell − cost wot) / cost wot;
- margin = (sell − cost wot) / sell (sell without tax).

**Money:** Decimal end to end. No float, no epsilon (the sale-order lesson).

---

## 5. Posting (perpetual — `accounts.cogs_mode`)

**Stock** = a purchase shadow voucher (D3; P4 builds the writer):
- RECEIPT `['PURCHASE']` from a GRN or direct invoice;
- ISSUE `['PURCHASE_RETURN']` from a return or claim;
- RECEIPT from a Shortage Receipt.

All carry `svh_supplier_id`, `svi_supplier_id`, `svi_landed_rate`, the barcode, the lot identity (batch / MRP / **selling price** / expiry), and the auto batch + batch detail.

### GRN

| Dr | Cr |
|---|---|
| INVENTORY (provisional) | GRNI |
| INVENTORY (off-bill charges) | each loadman / party ledger |

### Direct invoice

| Dr | Cr |
|---|---|
| INVENTORY (landed value of received + free) | Supplier (Net Payable) — bill-wise PURCHASE |
| INPUT_CGST/SGST/IGST/CESS (eligible, received qty) | each loadman / party (off-bill charges) |
| INPUT_TAX_PENDING (GST on short qty) | RCM_*_PAYABLE (RCM lines) |
| SHORTAGE_RECEIVABLE (value of short qty) | ROUND_OFF (if Cr) |
| on-bill charge ledgers (not landing cost, posted separately) | |
| ROUND_OFF (if Dr) | |

Ineligible / blocked GST is inside INVENTORY.

### Invoice from GRN(s)

| Dr | Cr |
|---|---|
| GRNI (provisional value of the linked GRN qty) | Supplier |
| INPUT_* | |
| ± rate difference: for qty **still on hand** → INVENTORY via revaluation rows (P13); for qty **already sold** → PURCHASE_PRICE_VARIANCE | |

### Shortage Receipt

| Dr | Cr |
|---|---|
| INVENTORY | SHORTAGE_RECEIVABLE |
| INPUT_* | INPUT_TAX_PENDING |

### Shortage closed by a supplier CN

| Dr | Cr |
|---|---|
| Supplier | SHORTAGE_RECEIVABLE + INPUT_TAX_PENDING |

### Return row

| Dr | Cr |
|---|---|
| Supplier (bill-wise PURCHASE_RETURN) or Cash | INVENTORY (lot cost) |
| | INPUT_* (bill rate) |
| | difference bill-rate value vs lot cost → PURCHASE_PRICE_VARIANCE |

### Claim rows

| Row | Dr | Cr |
|---|---|---|
| Claim — goods sent | CLAIM_RECEIVABLE (value + GST) | INVENTORY, INPUT_* |
| Claim — destroyed | CLAIM_RECEIVABLE (value) | INVENTORY |
| Claim — destroyed, its GST | ITC_REVERSAL (expense) | INPUT_* |
| Settlement | Supplier | CLAIM_RECEIVABLE |
| Short-settled | WRITE_OFF | CLAIM_RECEIVABLE |

### Replace row

| Dr | Cr |
|---|---|
| SHORTAGE_RECEIVABLE | INVENTORY |
| INPUT_TAX_PENDING | INPUT_* |

The replacement arrives through a Shortage Receipt.

### Return against GRN (rejection)

| Dr | Cr |
|---|---|
| GRNI | INVENTORY |

**Other registers:**
- **GST register:** `acc_voucher_doc_register` INWARD, nature PURCHASE / PURCHASE_RETURN; `vtx_itc_eligibility` per line. A supplier debit note maps INWARD sign −1 (P7).
- **Bill-wise:** `acc_bill_balance` — invoice = PURCHASE (CR); return = PURCHASE_RETURN (DR); loadman rows = PURCHASE-type payables on the employee ledgers.
- **New roles (P8):** GRNI, SHORTAGE_RECEIVABLE, CLAIM_RECEIVABLE, INPUT_TAX_PENDING, ITC_REVERSAL, PURCHASE_PRICE_VARIANCE, RCM_CESS_PAYABLE, TCS_RECEIVABLE.

---

## 6. Schema (purchase schema) — outline for the DDL

Every transaction table follows the house rules:
- **partitioned BY LIST (acc_year)**, PK (id, acc_year), uuidv7;
- company / branch / device / session ids; `_sync_date`, created / modified, is_deleted;
- varchar + CHECK (no enums);
- **no CHECK or UNIQUE over anything two offline branches can both create**;
- caches only as rebuilt roll-ups.

| Table | Prefix | Holds |
|---|---|---|
| `purchase.pur_order` / `pur_order_item` | po_ / poi_ | PO header / lines (+ short close qty + reason per line) |
| `purchase.pur_grn` / `pur_grn_item` | grn_ / gri_ | GRN header / lines (challan qty, received, damaged, free, provisional rate) |
| `purchase.pur_invoice` / `pur_invoice_item` | pi_ / pii_ | Invoice header (supplier inv no / date, printed amount, payment hold + reason, source kind) / lines as printed (HSN, rate, tax, ITC state, billed / received / short) |
| `purchase.pur_item_alloc` | pia_ | Stock side of a GRN / direct-invoice / return line: godown, batch, mfg, expiry, MRP, selling, size / colour / model, qty, free, barcode, `pia_lot_id` after post. Polymorphic parent (doc type + line id). |
| `purchase.pur_return` / `pur_return_item` | pr_ / pri_ | Return header (kind: invoice / GRN, settlement mode) / rows (outcome, reason, source line, alloc) |
| `purchase.pur_claim_settlement` | pcs_ | Settlement rows against claim rows (amount, CN no / date, write-off) |
| `purchase.pur_shortage` | psh_ | Shortage rows (source, item, qty, value, GST pending, outcome, status) |
| `purchase.pur_shortage_receipt` / `_item` | psr_ / psri_ | Receipts against shortage rows |
| `purchase.pur_doc_link` | pdl_ | Source line → target line with base qty: PO→GRN, PO→inv, GRN→inv, inv→return, GRN→return, sale-bill→inv (cross-company) |
| `purchase.pur_supplier_item` | psi_ | Supplier–item map (MST-02) |
| `purchase.pur_agent` | pag_ | Agent master (MST-05) |

**Reused, not created:**
- `txn_charge_detail` (+ `cd_party_id`, + `cd_on_bill`);
- `txn_transport_detail`; `txn_hold`; `txn_status_log`;
- `acc_*` posting; `stock_*` engine;
- `item_price_change_log`; `print_*`; `audit_log`.

**Columns on existing tables:**
- `charge_master.chg_on_bill`;
- supplier / ledger MSME + licence fields (MST-01);
- item master fields (MST-03).

---

## 7. Routes (own family — new module = own URLs)

```
/purchase/orders            save | get | list-pending | short-close | clone | cancel
/purchase/grns              save | get | post | cancel | from-po
/purchase/invoices          save | get | post | cancel | from-grn | from-po | from-sale-bill | validate
/purchase/returns           save | get | post | cancel | source-lines
/purchase/claims            open | settle
/purchase/shortages         open | receipt save/get/post/cancel | close
/purchase/supplier-items    save | get | list
/purchase/lookup            item (last rates, PO pending, supplier map) | supplier-context (notes, pending POs, open bills)
/purchase/labels            from-doc
```

- Save with `post: true` = save + post in one transaction (house standard).
- `/txn/adjacent` registry entries: PURCHASE_ORDER, GRN, PURCHASE_BILL, PURCHASE_RETURN, SHORTAGE_RECEIPT.
- Menu rights: 23 PO, 24 Invoice, 25 Return, 195 Shortage Receipt. **New menus** for GRN and Claims.

---

## 8. Screens (Qt) — mockups next

Every entry screen follows Opening Stock's layout + TxnKeys, NexTable + delegate, popups with EnterNavigation.

| Screen | Notes |
|---|---|
| Purchase Order | lines + pending column; clone / merge dialog |
| GRN | lines + alloc popup F12 + labels; crew pickers (dd 72) |
| Purchase Invoice | source picker (GRN / PO / sister sale bill); printed-amount check strip; match exceptions; side panel (last rates, supplier bills); charges grid with payee split |
| Purchase Return | source bill / GRN rows; outcome per row; settlement |
| Claims | pending claims list + settle dialog |
| Shortage Receipt | open shortage rows of a supplier → receive |
| Supplier–Item map | master list + popup |

Lists use TxnMainView + TxnDocTypes (`txn_doc_types_purchase.h`).

---

## 9. Acceptance (what "phase 1 done" means)

1. **PO → GRN (part) → invoice (rate +2 %)**: stock in at GRN; GRNI cleared; the rate difference split on-hand / sold. **Stock-in-Hand = the stock valuation report.**
2. **Direct invoice 5 billed / 3 received**: stock 3; SHORTAGE_RECEIVABLE = 2 × cost; ITC only on 3. Shortage Receipt of 2 → ITC released.
3. **Coolie**: QTY method, 2 loadmen 60 / 40 → two payables; Payment 100 settles one.
4. **Return row + claim row + replace row** on one document; claim part-settled + write-off; replace received.
5. **Old-bill edit** with a later Change Selling → price untouched, note shown.
6. **Paid bill**: rate edit refused; remarks edit allowed.
7. **Offline**: two branches enter the same supplier bill offline → both sync; the duplicate report shows them. The voucher serial does not collide (P1).
8. **Inter-company**: company A sale bill → company B purchase, batches / MRP carried; a second pull is refused.
9. **Textile ratio pack** → one label per piece; the label scans on the sale bill.
10. **Every document**: Prev / Next, History, hold / resume, cancel with reversing rows.

---

## 10. Order of work

1. **notes-162 prerequisites** (P1–P16) — Prathap.
2. **Mockups** (§8) → user review.
3. DDL + routes + posting — Prathap, by document: masters → PO → GRN → invoice → shortage → return / claim.
4. Qt screens in the same order; live tests per document.

## 11. Open points (do not block the start)

- Loadman equal split: by person, or by hours / qty each handled? Phase 1 = equal + editable.
- Employee ledger name reads "<name> – Staff Advance"; with coolie dues it becomes a staff account. Rename the pattern to "<name> – Staff A/c"? (user)
- Inter-company items not shared between the companies: map by item code, or refuse? Phase 1 = refuse with a list.
