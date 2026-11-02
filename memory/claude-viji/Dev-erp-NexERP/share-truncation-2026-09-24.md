---
name: share-truncation-2026-09-24
description: "18 share files were cut to 8K/32K/64K on 2026-09-24 19:16 (Samba client copy); 14 rebuilt in scratchpad, 13/15q/17 not fully — check file size before trusting any share file"
metadata:
  node_type: memory
  type: project
  originSessionId: 0dbdc790-b14e-4a2b-8980-56a32216f95e
  modified: 2026-09-26T12:51:59.097Z
---

On 2026-09-24 19:16:48 eighteen files in /home/vk/Shared/Prathap were truncated to exactly 8192/32768/65536 bytes plus an added `\n` (sizes 8193/32769/65537). Found 2026-09-26. The likely cause is a Samba client (.106 or .102). No Claude session was running, and the share has no backup or versions.

Rebuilt by replaying the session transcripts, with every candidate prefix-verified. They are in scratchpad `0dbdc790.../scratchpad/recovered/` (session-scoped; copy them out before it goes):
- 14 full: stock/16_stock.sql, 19_stock_posting.sql, 16s_stock_seed.sql, 19_physical_stock_flow.md, 21, 22, 26, 29, 33, receipt/HANDOVER_receipt_backend.md + plan-backend-receipt.md, sales/plan-backend-sales.md, react-plans opening-balance/receipt/received-cheques.
- 17_printing.sql: partial (last ~15 KB of the settings seed lost).
- 13_loyalty_ledger.sql and 15q_promotion_grid_queries.sql: NOT recovered. Ask the user for a copy on the client machines.

Pre-strip (2026-09-04) copies of 16/19/20 WITH their function bodies are in ~/.claude/file-history/e4646e35-.../ (42c03feb706da33c@v7 = 16, d65aba2b533c16bc@v4 = 19, fe248e1bf6ff45fc@v2 = 20).

**How to apply:** before reading a share file, check its size isn't 8193/32769/65537. Restoring into the share needs the user's go-ahead.

**RESTORED 2026-09-26** with the user's OK: the 14 full files went back into the share, the cut copies are kept beside them as `*.trunc`, and 17_printing.sql was left as it is (the printing module is already built). 13 and 15q are still lost.
