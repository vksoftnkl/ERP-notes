---
name: erp-sale-bill-react-port
description: Sale Bill Entry ("Sales Entry") React port — all 8 phases done 2026-09-12; 4 plan corrections found against the server
metadata:
  type: project
---

Ported the Qt `SaleBillEntry` to React in `/home/vk/Dev/erp/ERP client` against a
plan document the user pastes in the prompt (it is NOT in the repo, nor are its
companions `plan-react.md`, `plan-react-sale-tender.md`, `plan-react-sale-order.md`,
`plan-react-printing.md`; no Qt sources are on disk either).

**All 8 phases of §20 shipped 2026-09-12** — `features/sales/salebill/`, route
`/sales/sale-bill`, menu 12 "Sales Entry", 183 own tests. Header+grid+engine,
charges/auto-apply/POS, validate+save+load, tender, adjustments, imports,
hold+autosave, cancel. Only print is absent (§18.1 — no pipeline exists).

**Four plan errors confirmed against the ERP server, corrected in code comments:**
1. §15 "the refno is the client's" — wrong. `sbBillSlno` AND `sbBillRefno` both come
   from voucher sequence type 22; client values are ignored.
2. §16 `POST /bills/delete` "cancels the bill" — wrong. It cancels the **sale order
   the bill was raised against**; the bill row is untouched. NO route cancels a bill.
3. §7.4 size→CFT has no cell: ui table 22 configures **no Size column** (AliasName at
   position 4 where the order's grid 24 has Size). Needs a seed decision.
4. §12 "autosave → IndexedDB" — `txn_hold` already has an **AUTOSAVE kind**
   (`ck_txh_kind`, unique index `ux_txh_autosave`, one row per year/device/operator/
   doctype). Implemented IndexedDB as the plan says; the server option is flagged.

**Server contracts worth remembering:**
- `sbCustId` and `sbiGodownId` are both REQUIRED (unlike the quotation's) — godown
  comes from the item-price lookup, which can answer null. Guarded in validate.ts.
- `adjustments[]` is the ONE save array where absent ≠ empty: omit to leave the
  stored settlement alone, `[]` to reverse it. `GET /bills/get` returns none.
- Open credits: `GET /transactions/party-balance`, takes NO acc year (credits are
  never carried forward); post the credit's own `billAccYear` beside its `billId`.
- `sbSalesmanId`/`sbLoadmanId`/`sbPackedId` are `uuid[]` — `[]` for nobody, never null.
- Item grid = ui table 22, device Desktop so widths are `qtPercent`; 94 cols, 89 visible.
- Bill list = configured grid 86 (binds iacc_year, unlike the order's grid 87).

See [[erp-server-http-testing-without-credentials]].
