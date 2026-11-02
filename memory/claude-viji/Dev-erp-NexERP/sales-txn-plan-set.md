---
name: sales-txn-plan-set
description: Where the six-sales-transactions plan lives (share/sales + share/gst + 29/30 SQL), the two decisions (COGS perpetual, one sale_bill with sb_bill_mode), D51/D52, and what is still unverified/owed
metadata:
  type: project
---

Written 2026-09-20 on "all defaults, write the plan". Everything is in
`/home/vk/Shared/Prathap`: `sales/README.md` (read order), `sales/
sales_posting_flow.md` (THE spec), `29_sales_documents.sql` + `30_sales_
alters.sql` (P0, Prathap runs), `sales/plan-backend-sales.md`, `sales/
plan-qt-sales.md`, `gst/` (GSP layer: README, gsp_flow, two plans),
`notes.txt` (40).

**Also:** `sales/fsales-audit.md` = 3.0's fsales flag by flag vs the plan (D53–D59:
any-godown FIFO, VERIFIED delivery event, auto unit-break = engine REPACK pair,
qty-slab rates deferred to pricing, update-remarks route, loyalty one-scheme-per-bill + walk-in never earns). Loyalty
contract in full = spec §5.8 (scheme rate wins over tender master; the tender
panel must read party-context, not tnd_conversion_rate).

User skipped (2026-09-20): mobile/web orders and contract rates — do not build
them into the sales-txn work.

**Decisions:** COGS PERPETUAL (`accounts.cogs_mode`; the DC posts DR COGS /
CR INVENTORY, bill-against-DC revenue only, direct bill both, returns
reverse); one `sale_bill` with `sb_bill_mode` POS|WHOLESALE; all 50
analysis questions = default; D51 one COGS pair per document; D52 a POS
bill writes the party pair + bill-balance row like wholesale.

**Why:** the user wanted one clean ERP for chain / single store / wholesale /
supermarket; the receipt module is the pattern (DRAFT→POSTED→CANCELLED,
reversal never delete, legs by role, R20 amend).

**How to apply:** build in the phase order P0 (SQL) → P1 (shared
StockPostingService lifted from the stock engine's phases, SalesPostingService
by role, DocRegisterService, StatutoryService) → P2 bill+order → P3 DC →
P4 return → P5 Qt. Never hard-code a law figure — `fn_statutory_limit`
([[law-params-per-company]]). No 3.0 reference for the challan
([[sales-txn-3-0-reference-scope]]). Ids in plan-qt-sales §0 / plan-qt-gsp §0 are CONFIRMED (2026-09-20). Build
rhythm the user set: ONE transaction at a time — mockup PNG → user confirms →
code → next transaction; bill first (sales/sale_bill_additions_ui_mockup.py,
shared helpers in sales/mock_lib.py). 29/30 dry-ran clean on the
`erp_dry` localhost clone 2026-09-20 (keep re-running there after any edit); not
run live — that is Prathap's P0.
