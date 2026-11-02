---
name: erp-a2-posting-services-state
description: "ERP server A2 (shared posting services) — where it stands after 2026-09-21, what is verified, what the plan docs are and where they are NOT"
metadata: 
  node_type: memory
  type: project
  originSessionId: a3c34649-c386-479b-8227-02eda6fcbd55
  modified: 2026-09-21T14:15:25.766Z
---

A2 = the shared posting layer in `ERP server/src/modules/sales/posting/` (SalesPosting, DocRegister, LoyaltyLedger, PromotionUsage, Statutory, ChargeCarry, sales.guards, sales.settings) plus `src/modules/stocks/posting/` (StockPostingService, the one stock engine; stock-voucher posts through it). Done and verified on 2026-09-21; next is A3 (bill/order routes, HANDOVER §2).

**Why:** The plan docs it cites (`plan-backend-sales.md`, `offline-sync-invariants.md`, `sales_posting_flow.md`, `HANDOVER_endpoints.md`, `schema/stock/19_stock_posting.sql`, `notes.txt`) are not on local disk (a `find / -xdev` misses them) — they arrive pasted, but the `schema/stock/` share IS mounted via gvfs at `/run/user/1000/gvfs/smb-share:server=192.168.0.103,share=shared/Prathap/schema/stock` (16_stock.sql, 16q, 19q, plans, mockups; no VCS). This box (192.168.0.106, `localhost:5432/ERP`) IS the A1 dev DB, cleared to masters-only.

**How to apply:**
- Stock e2e suites `opening-stock`, `physical-stock`, `stock-transfer` skip 100% (they gate on plpgsql `fn_svh_post*` that was never deployed) — a PASS there means nothing ran. `stock-engine-ts.e2e-spec.ts` is the real TS-engine suite; it now gates on the ledger triggers being ABSENT (migration `20260922120000`).
- Rule: no functions/triggers in DB. `stock` schema must hold only `fn_create_stock_partitions`. Source tests enforce single writers: `test/sales/loyalty-single-writer` and `test/stock-ledger-single-writer`.
- Pre-existing unit failures at HEAD (not A2): `auth.service.spec`, `quotation.service.spec` (vchrTypeId 21 vs 2), `bill.service.spec` (order conversion), and the `*-exception.filter.spec` files (`httpContext.getRequest is not a function`).
- `dist/` is tracked; rebuild with `npm run build` before committing. A `nest start --watch` dev server is often running (seen 2026-09-30), so `dist/` changes under you as you save and the live :3011 server restarts onto your edits.
- Pre-existing unit failures seen 2026-09-30: the `*-exception.filter.spec` files, `auth.service.spec`, `quotation.service.spec`, `sale-order.service.spec`; `pdfkit.renderer.spec` only fails under full-suite load. `master-lookup.service.spec`'s 27 failures (mock gaps) were fixed that day. Related: [[erp-stock-transfer-module-state]], [[erp-server-http-testing-without-credentials]].
