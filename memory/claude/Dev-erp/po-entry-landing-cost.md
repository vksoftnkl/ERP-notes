---
name: po-entry-landing-cost
description: "2026-11-02 PO entry (menu 23) — 6 user points; charges now spread on PO lines \"full like Sale Entry\" with landing cost; server DEV-v1 bbef3f41 + NexERP main bcd844a (both unpushed); open decisions"
metadata:
  node_type: memory
  type: project
  originSessionId: cfb5e46c-d2f9-46e0-8a3a-5376ed8c408a
  modified: 2026-11-02T07:48:47.732Z
---

**Asked on 2026-11-02 (Qt PO entry, menu 23):**
1. Credit days as wide as the other fields.
2. No input box inside the Rate / Disc cells, nor in the charges table.
3. A landing cost column.
4. Should charges go into landing cost, split before / after tax like Sale Entry?
5. Even quick icons.
6. Totals on the right of the charges table.

**User decisions:**
- Full, like Sale Entry: a before-tax charge raises the line's Taxable and GST.
- Show both landing costs, without GST and with GST.
- Qt only for now: the web PO has no new columns.

**Where it landed** (both fast-forwarded into the user's checkouts, neither pushed):
- **NexERP `main`:**
  - 721dabe: points 1, 2, 5 and 6. Adds `PurchaseTotalsStrip::setGridColumns`, and new `quick/whatsapp.svg` + `quick/print.svg`.
  - bcd844a: columns 23–26 (Chrg before tax, Chrg after tax, Landing cost, Landing cost incl. GST), plus a hidden 27. After load, a /validate call fills them.
- **ERP server `DEV-v1`, bbef3f41:**
  - The PO runs ONE `calcPurchase` with the real charge-master flags, via the shared `purchase-charges.ts` builder.
  - /validate gains these line keys: chrgBeforeTax, chrgAfterTax, offbillShare, amount, landedValue, landedCost, landedCostIncl. Both landing costs are per the line's unit, 6 dp.
  - poTotal = net + chargesEstimate.
  - Approval stays on goods only (`poGoodsValue`).
  - The invoice's rate check against the PO uses taxable − poi_chrg_before_tax.
- Tests: test_purchase_order_landing passed 98/98 on :3011 after the merge.

**Open / for the user:**
- **Item column is cramped:** with 23 columns and no sideways scroll at 1560 px, Item is about 104 px, so long names get "…". Offered fix: hide Received / Short-closed / Pending / Status on new and draft POs.
- **Not registered in ui_table 57:** columns 23–27, so "layout has 23 columns but the entity declares 28" is logged on every open.
- **Old POs:** saved before bbef3f41, they show blank split cells.
- **Data drift:** the `ZTP-CD` cash-discount master went from FIXED 50 to PERCENT 50 on 11-02 at 11:08, before this session. That gives 3 line-math failures and 12 invoice API failures.
- **Already failing:** `test_purchase_final` "Ctrl+G opens the GRN" fails on main too.
- **Share test:** fidelity P2 needs new expectations.
- **Web PO:** with a before-tax charge, its Taxable / Tax / Ordered value now include the share.

**Why:** the user's 11-02 PO list ([[erp-finished-work-must-land-in-checkout]]).
**How to apply:** check `git log` in both repos before redoing any of this, and keep the web PO in mind when it gets ported.
