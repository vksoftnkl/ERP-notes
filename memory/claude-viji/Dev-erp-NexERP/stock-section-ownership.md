---
name: stock-section-ownership
description: The user gave me the WHOLE stock section — decide designs myself, check them against the standing rules before writing, never hand over a half-checked plan
metadata:
  type: feedback
---

The user said on 2026-10-06: "why this confusion, i gave stock section completely to you". That came after notes 92 went through Rev 1 → 2 → 3 in one hour:
- Rev 1 used an incrementing cache, against the offline rule already in [[offline-sync-invariants]].
- Rev 2 added a `stock_lot_cost` table that held nothing `stock_balance` couldn't.
- Each time I asked "shall I…?".

**Why:** the stock module is mine to design. Every revision and every "shall I" hands back a decision that was mine, and makes the plan look unreliable to the person passing it on to Prathap.

**How to apply:** before a stock plan reaches the user or the share, check it against:
1. offline sync: rebuild, never increment; no CHECK over a running total; deterministic ids;
2. [[sql-vs-nestjs-rule]]: no DB functions or triggers;
3. no new table unless it holds information no existing row can;
4. the real NestJS order and code paths, read, not assumed.

Then decide and write ONE version. Ask the user only for real business choices (what the shop wants), never "shall I write / revise it".
