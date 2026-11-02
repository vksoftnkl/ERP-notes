# Purchase phase 1 — addendum 2 (cold review, 27-Oct)

A fresh reviewer read the analysis, plan, addendum 1, notes-162 and all mockups. It found 4 blockers, 14 major and 10 minor findings. The live-DB facts were re-checked. Below: each finding and its decision. Business questions are at the end (B1–B7).

Read the plan files in this order: plan rev 1 → addendum 1 → this file.

## Blockers

**R1. Supplier bill number vs existing UNIQUE indexes.** Live:
- `ux_avh_doc_refno` (company, party, voucher type, year, doc_refno) and `ux_abl_doc_refno` (company, party, bill type, year, doc_refno; no cancelled exclusion) are both company-wide;
- `uq_acc_voucher_doc_register_doc_in` includes the branch.

**Decision:**
- `avh_doc_refno` / `abl_doc_refno` hold **our** number (PUR/…/D1/00012). It is unique per device, so offline branches never collide, and a cancel + re-enter gets a new number.
- The supplier's invoice no lives in `pi_supplier_inv_no` and `gdr_doc_no`.
- Duplicate guard: the online check in the service is **company-wide** (not only `liveInwardHolderOf`, which is per branch). Offline duplicates are caught by report RPT-09.
- No index change needed. Recorded as notes-162 P17 (information).

**R2. GRN → invoice rate difference.**
- **The invoice must be at the GRN's branch** (refused otherwise). GRNI therefore nets per branch.
- **Split of the difference** (computed at the centre at post):
  - on-hand share = min(GRN qty, today's on-hand at the GRN branch) ÷ GRN qty;
  - for WAVG items "on-hand" is the item's on-hand at that branch; for lot-cost items it is that lot's.
  - The on-hand share → INVENTORY (revaluation rows, P13). The rest (sold, transferred, returned, adjusted) → PURCHASE_PRICE_VARIANCE.
- The same split applies to on-bill landing-cost charges that arrive with the invoice.
- HO GRN → Branch Outward → bill later: the transferred qty takes the variance (no chase across branches).

**R3. Shortage opened twice.** One shortage row per supplier line, ever.
- A GRN challan short opens the row with **qty only** (no value).
- The invoice for that GRN **adopts** the row: it sets value + pending GST. It never creates a second row.
- PO pending: a qty on an open shortage row is **not** PO pending (the PO shows "2 on shortage"). Only short-close returns it to pending.

**R4. Invoice mockup "From GRN" drawn like a direct bill.** The mockup now shows **Source = Direct (goods came with the bill)**, which is what its editable cells mean.

Rules for an invoice **from a GRN**:
- Recd / Dmg / Short, batches, serials and lots come from the GRN, **locked**;
- no off-bill charges (they stay on the GRN);
- only Billed, Rate, discounts, tax, on-bill charges and prices are keyed.

## Major

**R5. Invoice-from-GRN legs, completed.**

| Dr | Cr |
|---|---|
| GRNI (provisional value of linked GRN qty) | Supplier (Net payable) |
| INPUT_* (eligible) | |
| on-bill charges: INVENTORY / PPV split (R2) or the charge ledger | |
| TCS_RECEIVABLE (supplier TCS, included in Net payable) | |
| ROUND_OFF Dr / Cr | |
| rate difference per R2 | |

- Billed qty > GRN qty → the excess is short → R3 row.

**R6. Claim and Replace rows** credit INVENTORY at **lot cost**, like Return rows; the difference to bill value goes to PURCHASE_PRICE_VARIANCE.

**R7. ITC reversal not doubled.** Each return / claim row carries `itc_reversed_in_books` + `supplier_cn_status` (PENDING / MATCHED).
- The GSTR-3B / ITC view must not reverse again when the supplier's CN reaches 2B / IMS; it matches the CN instead.
- **Rejected claim** (no CN ever): **re-avail the GST** (Dr Input GST / Cr CLAIM_RECEIVABLE for the tax part) and write off the value only.
- **P7 restated:**
  - our debit note is a commercial document, not a GST document, so it creates no register row of its own;
  - the **supplier's credit note** is recorded as an INWARD credit note (sign −1) when it arrives — on the Claims screen or the return's "Supplier CN" field.

**R8. ITC as amounts, not one state per line.** `pii_itc_eligible_amt`, `pii_itc_pending_amt`, `pii_itc_ineligible_amt` (+ class). A partly short line shows "8 Elig. / 2 Pend.".
- **Whether ITC on the received part waits for the last lot (sec 16(2) 1st proviso) = B1.**

**R9. Damage defined once.**
- Damage seen at receipt = A1: stock in the DAMAGED bucket, ITC taken. D6's "damage like a shortage" is **retired**.
- Excess (received > billed) = B5.

**R10. Cash discount in cost or not = B2.**
- Until answered: the CD charge row's "Into cost" follows the charge master (`chg_landing_cost`).
- The mockup's "Yes" is one possible master setting, not a rule.

**R11. Supplier's freight on the bill is part of the value of supply** (sec 15(2)(c)). On-bill supplier charges default to **before tax** (taxed at the line's rate). The mockup's freight row is now "Before tax".

**R12. GTA freight.** An off-bill freight charge to a party whose ledger is reverse-charge (`led_is_reverse_charge`) or GTA is **refused** in phase 1, with the message "Enter it in Expense Voucher (RCM)". RCM on GTA stays with the Expense Voucher.

**R13. RCM self-invoice (DOC-10) → phase 2.**
- Phase 1 refuses an unregistered supplier's line whose HSN is in the notified RCM list (metal scrap, …), with a message.
- Self-invoice numbers must be ≤ 16 characters (Rule 46(b)) → a separate short series later.

**R14. Price rows under offline sync.**
- The D16 check runs **at the centre at post**, comparing **business times** (doc date-time of each purchase, change time of manual changes) from the Price Change Log. Sync order does not matter; only business time does.
- **D10 restated for MRP-bucket items:** a new MRP = a new bucket (old stock untouched); the **same MRP** = the bucket's price row updates (all stock at that MRP — the FMCG norm).
- **Which branch's price row a purchase writes = B3.**

**R15. No in-place counters in purchase tables.**
- `pur_supplier_item`: deterministic id (uuid v5 of company + supplier + item), so two branches creating it converge. Last rate / date / bill count are **derived** (query / rebuild), not updated per bill.
- **New `pur_shortage_close`** rows (CN / claim / write-off) = the closing documents. Shortage status + open qty are derived. The same pattern already exists for `pur_claim_settlement`.

**R16. Offline-unsafe uniques that purchase touches.**
- **`uq_ean_code_ci`** is global UNIQUE: "save as new EAN" (ENT-16) is **online only**.
- **Serial duplicates** checked online; offline duplicates → a detect report.
- Added to notes-162 as **P18**.

**R17. Textile vs "printed line".**
- Storage: `pur_invoice_item` = the supplier's printed line (HSN, rate, tax — for 2B / IMS); size / colour / model / batch are on its **`pur_item_alloc` rows**.
- **Screen:** the A2 grid shows one row per size × colour as the user approved. They are the alloc rows, grouped under their printed line (column "Bill line"). Ratio pack makes alloc rows under one printed line.

**R18. Return against a GRN that later gets billed.**
- When the invoice bills qty that was rejected at the GRN, the invoice line shows it as **"Rejected"**, and Post creates the Return row (debit note) for it, with GST at the bill's rate.
- A rejection against a **challan** stays without GST.

**R19. Previously undecided:**
- **Provisional rate** = PO **net** rate per unit (after PO discounts, before tax); else the last purchase net rate. With neither, the rate is **required** (no zero-rate GRN).
- **Payment hold:**
  - release needs a new right on menu 24, "Release payment hold";
  - "override (logged)" = the user, time and reason go into the status log;
  - Payment 100 refuses a held bill without that right.
- **Cash purchase = B6.**
- **Supplier TCS:** a line in the totals, included in Net payable; Dr TCS_RECEIVABLE.
- **Bill-from GSTINs:** new small table `purchase.pur_supplier_gstin` (supplier, GSTIN, state, address, default flag), deterministic id.
- **Item shortage %** (transit-loss tolerance) = B4.
- **Return cap** = received + free + damaged − already returned (not billed).
- **Partly invoiced GRN:** invoiced lines are locked; un-invoiced lines stay editable; cancel only after the invoice is cancelled.
- **Year ends:** `pur_doc_link`, shortage, claim and close rows carry **both** source and target `acc_year` and are partitioned by their own (target) year; open amounts are derived across years.
- **Inter-company:**
  - a sale bill pulled into a sister purchase cannot be amended or cancelled online until the purchase is cancelled (refused with the purchase no);
  - the supplier's e-way bill no is stored on the purchase (`pi_ewb_no`), **not** in `acc_voucher_doc_ewaybill` (that table is for EWBs we generate — its number is unique DB-wide).

## Minor (mockups fixed or noted)

- **R20. Cost w/o tax figures** are illustrative. The spread rule (chg_cost_alloc VALUE default; coolie included) is in the plan. Line 2 now shows cost and selling per the same unit.
- **R21. GRN mockup:** 5 lines; the new-MRP line prints labels; the F12 header says Σ = received + damaged + free.
- **R22. Return / claims mockups:** Atta is SALEABLE (breakage found in the shop, destroyed here); the Surf batch matches the invoice (SX1025).
- **R23. PO mockup:** the approval field reads "Off (setting 0)".
- **R24. Ratio pack** 1:2:2:1 × 3 = S3 / M6 / L6 / XL3 — the grid example now matches.
- **R25. Silk saree:** sarees are fabric (chapter 50) → 5 % at any value; the example now uses a **readymade shirt above ₹2,500** for the 18 % slab.
  - The slab test uses **taxable value per piece after discount**.
  - ENT-09 **validates and warns** (the supplier's printed rate is kept); it does not overwrite.
- **R26. Edit rule:** §3.8 wins (qty may not go below returned + sold). D11's "nothing consumed" is relaxed to that.
- **R27. MSME due date earlier than credit days** → the due date is the MSME date, and the chip warns.
- **R28. Three-way match basis** = net landed rate per base unit, without tax. The GRN qty tolerance is removed (short / excess have their own flow); only the rate tolerance remains.

## Business questions (owner / CA)

| # | Question |
|---|---|
| B1 | Goods short on a bill: take ITC now on what arrived, or only when the last lot arrives (sec 16(2) 1st proviso)? (CA) |
| B2 | Supplier's cash discount: reduce item cost, or book as Discount Received? |
| B3 | A purchase that sets a selling price: update every branch, or only the receiving branch? |
| B4 | Loss within the item's shortage % (normal transit loss): absorb into cost with no shortage row, or still chase the supplier? |
| B5 | Over-delivery (received more than billed): keep as free, return, or ask the supplier to bill? |
| B6 | Cash purchase at a branch: pay from the branch cash ledger, or from a till (counter)? |
| B7 | The lorries / transporters you pay yourselves — are any GTA (they issue a consignment note and GST applies under RCM)? |

## Owner answers (27-Oct)

| # | Answer → rule |
|---|---|
| B1 | **ITC on what the purchase entry received.** ITC is taken for the received (+ damaged) qty at post; only the short qty is pending until it arrives. (This deliberately does not wait for the last lot; the CA may revisit — it is one rule in the ITC builder.) |
| B2 | **Cash discount reduces item cost.** The CD charge row is landing cost, always (no DISCOUNT_RECEIVED leg on purchase bills). |
| B3 | **New setting `purchase.allow_price_update`** (per company, default **off**). Chains keep it off — a purchase never writes selling prices / MRP rows; the bill only shows markup / margin for information, and pricing stays with Change Selling. A single shop turns it on; then `purchase.price_update` (UPDATE_PRICES / COST_ONLY / OPEN_CHANGE_SELLING / NONE) and the price-change right apply. Cost rows (`ipm_cost_*`) follow the stock engine either way. **B3's branch question disappears**: when on, a single shop has one branch. |
| B4 | **Use the item's shortage %.** A short within the item's shortage % (of received qty) is absorbed: no shortage row; its value spreads into the cost of what arrived (like 3.0). Beyond it, the whole short qty opens a shortage row. |
| B5 | **Over-delivery: the user decides per line**, default "ask supplier to bill". Choices on the line: **Hold for supplier bill** (stock in, value on GRNI until a bill / debit note arrives — it appears in "GRN not billed"), **Return** (opens a return row), **Keep as free** (stock in at zero cost, spread). |
| B6 | **Cash purchase pays from the branch cash ledger** (cash-in-hand of the branch, as Expense Voucher does), not a till. Tender strip: Cash / UPI / Bank. |
| B7 | **GTA / transporter GST bills: not now.** Phase 2 Landed Cost Voucher takes the transporter's service bill with GST (and RCM when applicable). Phase 1 keeps R12 (refuse off-bill freight to an RCM / GTA ledger). |
