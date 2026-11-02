---
name: erp-sales-endpoints-implemented
description: "State of the sales-transactions HANDOVER (2026-09-20) implementation as of 2026-09-22 — what exists, the deliberate gaps, and the assumptions to verify with the user"
metadata: 
  node_type: memory
  type: project
  originSessionId: 830b4a7c-7118-43ba-8c5f-c1d6b3c31c9e
  modified: 2026-09-22T06:04:22.793Z
---

On 2026-09-22 the sales-transactions endpoint contract (HANDOVER for Prathap, dated 2026-09-20) was
implemented in `ERP server/src/modules/sales/`: bills (validate/post/cancel/amend/delete-narrowed,
open-sources, party-context, delivery-status, update-remarks, transport, tender-context, retender),
sale-orders (post/cancel/amend, delete DRAFT-only), and four new modules `delivery-challan`,
`dc-return`, `sale-return`, `temp-credit`. Shared plumbing lives in `sales/posting/` (context,
blocks, transport band, sales-stock shadow voucher, DC fulfilment, stock reservation, GST gateway
stub, generic `SalesDocStore`).

**Why:** the handover's companion docs (`plan-backend-sales.md`, `sales_posting_flow.md`, numbered
SQL 11–31) are NOT in the repo; the tables already existed as Prisma migrations, so everything was
built against the live DB.

**How to apply / open points to raise with the user:**
- Stock for sales docs moves through a SHADOW `stock.stock_voucher` (ISSUE/RECEIPT) posted by the
  existing engine, with ledger rows labelled `SALES / <doc type>` via a new `ledgerSource` option.
- No GST module exists: `GstGatewayService` refuses IRN/EWB cancels honestly (`GST_IRN_CANCEL_FAILED`)
  and only logs after-post generation. Wire it when the GST module lands.
- No day-close table exists; `SALES_DAY_CLOSED` never fires (guard probes `accounts.acc_day_close`).
- Menu ids assumed: Sale Bill 12, Sales Order 11, Sale Return 13, Challan/DC return 182 ("Deliver
  Note"). `um_menu_id` is INTEGER (the A2 guard had cast it to uuid — fixed).
- Draft `adjustments[]` are only validated; `/bills/post` applies FIFO up to `sbAdvanceAmt` unless the
  post body names them. TEMP_CR "who" rides on the tender row (td_bank_name/td_ref_no/td_payer_vpa/
  td_notes) until post creates `acc_temp_credit`.
- New error code not in the handover: `SALES_AMOUNT_MISMATCH` (bill parts ≠ declared total).
- Pre-existing failing specs (not from this work): quotation numbering (type 21 vs 2), loyalty
  exception filter 500, sale-order cheque register.
