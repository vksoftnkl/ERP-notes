# notes-162 — platform fixes needed before the purchase module

Date: 2026-10-27. From: Viji / Claude. To: Prathap.

The purchase plan is `purchase/plan-purchase-phase1.md`; the evidence is in `purchase/analysis-purchase-base-2026-10-27.md` §6. These fixes come first: purchase posts through the same stock, accounts and log machinery as sales. P1 and P2 also matter **today** for sales and receipts at offline branches.

Each item below was checked on .106 or in the source on 2026-10-27.

## P1. Accounts voucher serial is company-wide

`ux_avh_voucher_slno` is UNIQUE on `(avh_company_id, avh_acc_year, avh_voucher_slno)` (non-draft, not deleted). Two offline branches posting while apart both take the same slno, and the second sync aborts.

**Asked:** number per branch + device (as stock vouchers do, with `dev_code`), or mint the slno only at the centre. Either way the index must include what makes it unique offline.

## P2. CHECK on a contended bill balance

`acc_bill_balance` has `ck_abl_settled`: alloc + disc + writeoff ≤ bill_amount. If a supplier bill is paid at HO and also adjusted by a return at an offline branch, the correct data fails at sync. Same rule as the loyalty CHECKs we converted (offline-sync-invariants).

**Asked:**
- drop it → `ix_abl_overdrawn` partial index + report;
- the posting service refuses while online.

## P3. "Last purchase rate" comes from any inward

`stock_item_cost.sic_last_purchase_rate/_date` take any unreversed inward: opening, TRANSFER_IN, SALE_RETURN, ADJUST_PLUS (`stock-voucher-posting.helper.ts` ~2647).

**Asked:** only `sml_txn_type = 'PURCHASE'`.

## P4. A purchase shadow stock writer

`sales-stock.service.ts` cannot be reused as is:
- it never sets `svh_supplier_id`, `svi_supplier_id`, `svi_landed_rate`, `svi_barcode`;
- it writes the document rate into `svi_sale_price`, which is part of the lot key;
- it never calls `assignAutoBatchNumbers` / `syncBatchDetails`.

**Asked:** a sibling writer for PURCHASE that:
- sets those columns, with sale price = the selling price;
- runs auto batch + batch detail;
- posts RECEIPT `['PURCHASE']` / ISSUE `['PURCHASE_RETURN']`;
- links `svh_link_src_*`.

## P5. Price Change Log source

`item_price_change_log.ipcl_source` CHECK lacks `PURCHASE`.

**Asked:** add `PURCHASE`. Also confirm `PriceChangeLogService` can answer "latest change to this price row, with its business time" (plan D16).

## P6. Doc-type CHECKs

`txn_status_log` and `txn_transport_detail.ttd_doc_type` lack GRN, SHORTAGE_RECEIPT, PURCHASE_RETURN (against GRN), CLAIM.

**Asked:** extend both.

## P7. Voucher types + GST direction

- No purchase-bill voucher type that is accounts + inventory; no purchase-return type.
- `gstDoc()` maps DEBIT_NOTE to OUTWARD.

**Asked:**
- types for GRN, purchase bill, purchase return / debit note, shortage receipt, claim settlement;
- a supplier-side debit note = INWARD, sign −1.

## P8. Ledger roles

**Missing:** GRNI, SHORTAGE_RECEIVABLE, CLAIM_RECEIVABLE, INPUT_TAX_PENDING, ITC_REVERSAL, PURCHASE_PRICE_VARIANCE, RCM_CESS_PAYABLE (voucher-derive refuses cess under RCM), TCS_RECEIVABLE.

**Asked:** seed the roles + ledger-map rows (the same way as the LOADING_* roles in notes-153).

## P9. Stock-in-Hand includes recoverable GST

INVENTORY is posted at `sml_cost_value` (with tax). With ITC, the recoverable GST would sit in Stock-in-Hand **and** in Input GST.

**Asked:** confirm which cost the INVENTORY leg reads. Purchase will state `svi_cost_rate` = landed **net of eligible ITC** (with tax only on ineligible lines). The INVENTORY leg must use that figure, and the sales COGS leg must stay consistent with it.

## P10. Batch detail left live by a cancelled draft (notes-149 open item)

A cancelled or deleted draft leaves its `stock_batch_detail` row live. Purchase will be the main writer of batch detail.

**Asked:** fix as noted in notes-149.

## P11. Sync of first-writer and incremented columns

- `stock_lot` first-receipt cost / source columns are overwritten by the cloud-sync PK upsert;
- `txn_charge_detail.cd_carried_amt` is incremented (charge carry).

**Asked:** rebuild both at the centre / derive them. Purchase carries charges PO → GRN → invoice.

## P12. Item preferred supplier has no FK

`item_master.item_supplier_id` has no FK. Purchase brings a supplier–item map (`purchase.pur_supplier_item`).

**Asked:** add the FK (or retire the column once the map exists).

## P13. Stock revaluation (value-only rows)

A GRN posts stock at a provisional rate. When the bill arrives at another rate, the qty still on hand must be revalued. `ck_sml_qty_nonzero` forbids a 0-qty ledger row, and `ck_svh_voucher_type` has no REVALUE. notes-92 Rev 2 proposed `stock_cost_revaluation` (append-only, read by the cost rebuild); it was never created.

**Asked:**
- create it, or the equivalent;
- make the rebuild include it;
- expose an internal service the purchase posting can call: (lot / item, branch, value delta, source doc).

## P14. Charge master + charge detail

- `charge_master.chg_on_bill boolean NOT NULL DEFAULT true` — "On supplier bill" (purchase rows only; sales ignore it).
- `txn_charge_detail.cd_on_bill` (snapshot) and **`cd_party_id uuid`** (the payee ledger).
  - `chg_man_party` ("Party Mandatory") is saved today, but no column holds the party and no code reads the flag.
  - One detail row per payee (coolie split per loadman).
- Validation: an off-bill purchase charge must be landing cost and carry no GST (plan D13 / D14).
- Today's purchase rows ("New Reg", "purchase") are test data — no migration needed.

## P15. Value-based GST slab

`inventory.tax_rate_master` has no value band. Apparel ≤ ₹2,500 per piece → 5 %, above → 18 % (GST 2.0, 22-Sep-2025).

**Asked:** a value threshold + the rate above it (dated, like the rest of the rate master), and an item-lookup answer that takes the unit value.

## P16. Attachment store

There is no attachment table. Purchase needs the bill scan and damage photos.

**Asked:** one shared `public.txn_attachment` (src module / doc type / doc id / acc year, file name, mime, size, storage key, sync date). Store files outside the DB if you prefer.

---

The Qt side starts with mockups; nothing here blocks them. Please send back which of P1–P16 you take as is and which you want to discuss.

---

## Added 27-Oct after the cold review (plan addendum 2)

**P17. Supplier bill no is NOT stored in `doc_refno` (information only).**
- `ux_avh_doc_refno` / `ux_abl_doc_refno` are UNIQUE per company + party, without the branch. Purchase keeps **our** per-device number there, so offline branches never collide.
- The supplier's invoice no goes in `pi_supplier_inv_no` + `gdr_doc_no`.
- **Asked:** the online duplicate check must search the whole company, not only `liveInwardHolderOf` (per branch).

**P18. Offline-unsafe uniques purchase touches.**
- `inventory.item_ean_codes.uq_ean_code_ci` is a global UNIQUE → "save scanned barcode as a new EAN" from a purchase is **online only**.
- Serial-number duplicates are checked online.
- **Asked:** a detect report for EAN / serial duplicates that arrive by sync.

**P19. Inter-company lock.** A sale bill pulled into a sister company's purchase must refuse amend / cancel until that purchase is cancelled.

**P20. Supplier e-way bill numbers stay off `acc_voucher_doc_ewaybill`** (its `uq_acc_voucher_doc_ewaybill_no` is DB-wide). They are stored on the purchase header (`pi_ewb_no`).
