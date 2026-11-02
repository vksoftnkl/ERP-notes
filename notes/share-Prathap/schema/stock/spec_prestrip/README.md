# spec_prestrip — REFERENCE ONLY, DO NOT RUN

These are the stock engine files as they stood on **2026-09-04**, before the
2026-09-21 strip removed every function and trigger body. They were rebuilt on
2026-09-26 from Claude's file history (the share copies were also truncated on
2026-09-24).

| file | original | taken |
|---|---|---|
| `16_stock.prestrip.sql.txt` | `16_stock.sql` (263,828 B) | 2026-09-04 20:26 |
| `19_stock_posting.prestrip.sql.txt` | `19_stock_posting.sql` (41,745 B) | 2026-09-04 20:22 |
| `20_stock_transfer.prestrip.sql.txt` | `20_stock_transfer.sql` (32,039 B) | 2026-09-04 19:45 (may miss that evening's last edits) |

The rule is that the database holds **no stock functions** except
`fn_create_stock_partitions`. These bodies are the **behavioural spec** that
`../plan-nestjs-stock-engine-port.md` ports into NestJS. Every `file:line`
that plan cites points into these files, byte-identical to the originals.

The `.txt` extension is deliberate: nobody should `\i` these.
