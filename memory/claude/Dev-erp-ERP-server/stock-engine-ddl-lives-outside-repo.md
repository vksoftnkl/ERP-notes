---
name: stock-engine-ddl-lives-outside-repo
description: Stock posting is 100% TypeScript since 2026-09-28 — no SQL function or trigger exists or may be added; every voucher type (transfers included) posts through StockPostingService with a postShape
metadata:
  node_type: memory
  type: project
  originSessionId: 04f8789e-47a6-43d6-8027-632e2629b5c4
  modified: 2026-09-28T06:08:49.580Z
---

DECIDED 2026-09-08 and completed 2026-09-28: the stock engine is TypeScript
(`src/modules/stocks/stock-voucher/stock-voucher-posting.helper.ts`), with full
`fn_sml_apply` parity, post AND cancel in process. On 2026-09-28 the transfer
functions (`fn_svh_post_transfer` / `fn_svh_receive_transfer` / the cancel guard)
were ported too, so NOTHING remains in SQL: `StockVoucherTypeRules.postFunction`
is gone, replaced by `postShape: 'SIMPLE' | 'COUNT' | 'TRANSFER_OUT' | 'TRANSFER_IN'`.
The user's flat rule (2026-09-22): no SQL function or trigger in the database except
the partition DDL helper; GENERATED columns, CHECKs, indexes and FKs stay.

What the engine does beyond the seven phases now: picks lots for a lotless outward
line by the policy's `stp_issue_strategy` (FEFO/FIFO/LIFO, MANUAL refuses, split by
`svi_split_no`), derives `line_cost_rate_wot` from `svi_tax_perc` when nothing states
it, refuses a zero-cost inward (stock module's own documents only, sales shadows
exempt), recomputes the header totals from the ledger at post and cancel (a count
carries the NET variance; `svh_line_count` stays the document's line count), writes /
settles `stock_transit`, and owns `sbl_reserved_qty` / `sbl_transit_in_qty` through
`refreshReserved` / `refreshTransitIn` (recompute, never increment).

**Why:** two engines drifted apart once already; a single door (`StockPostingService`)
is the guard, and `test/stock-ledger-single-writer.e2e-spec.ts` fails the build on any
second INSERT site.

**How to apply:** a new document type is a rule record with a `postShape`, never a new
engine. Transfer preflights must not refuse a lot-bearing line for missing identity nor a
receipt for a zero cost (its cost is `stt_cost_rate`); `allowsRepeatHolding: true` on both
transfer rule records. See [[stock-to-accounts-posting]] for the accounts leg and
[[stock-e2e-suites-run-now]] for the suites that stopped skipping.
