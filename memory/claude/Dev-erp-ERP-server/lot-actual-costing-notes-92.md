---
name: lot-actual-costing-notes-92
description: "Since notes 92 (2026-10-06) tracked items are costed per lot on stock_balance rows, plain stock keeps the item average; every derived stock figure is a rebuild from the ledger; deploy needs migration + POST /stock/admin/rebuild-costs"
metadata:
  node_type: memory
  type: project
  originSessionId: 4730bfed-9079-4e8c-b567-93820fa6f3ba
  modified: 2026-10-06T06:05:37.627Z
---

Notes 92 (2026-10-06): the effective policy's `stp_valuation_method` decides costing and FOLLOWS the
track flags (tracks nothing → WAVG, tracks anything → LOT_ACTUAL; `valuationMethodFor()` in the policy
service, `ck_stp_valuation_tracks` / `ck_spt_valuation_tracks` in the DB; direct policy INSERTs must
state the column). A tracked lot's cost per branch lives on its `stock_balance` rows
(`sbl_avg_cost_rate`, same on every godown/bucket), written by `applyLotCost`; `slt_cost_rate` is only
the first receipt's cost. `postingCte` reads the lot rate (`lotted`/`lc`) for every outward, count
shortage, AVG_COST/LOT_COST inward and re-lot IN (inherits the OUT lots' weighted cost).

Every derived figure is now a REBUILD from the ledger (balances, lot cost, item cost + WAVG stamp,
lot totals): qty ≤ 0 → the LATEST ledger row's rate (not "last inward"; this keeps engine test 7's 17);
no clamp of a fresh row (engine test 5 now expects −85 / −18); stamps come from unreversed rows (a
cancelled receipt is no purchase). `rebuildStockDerivedFigures(tx, scope)` is exported and backs
`POST /stock/admin/rebuild-costs?dryRun=true|false`. New lots get a deterministic uuid v5 id
(`lotIdentityUuid`, pgcrypto digest, namespace `STOCK_LOT_ID_NAMESPACE`).

**Why:** MRP STOCK ITEM's four batches (25/250/400/500) all read 340.909; the user's rule is "only plain
stock keeps the total average". Rebuilds are for branch-wise offline sync (recompute, never increment).

**How to apply:** deploy = `prisma migrate deploy` (20261006100000) THEN the rebuild (admin route, or
`plan/notes-92-lot-cost-backfill.sql` for a BEGIN/ROLLBACK dry run) — until the rebuild runs tracked
rows still carry the item average. On the dev DB the rebuild also repairs two pre-existing drifts
(item 001 qty −106.945→−107.945 from the old fresh-row clamp; ZT-ITEM-A value +585.96 because the old
engine unwound cancelled 50-cost openings at the running average) — `physical-stock` and
`opening-stock` e2e suites fail on the branch-wide assertion until it runs. Acceptance spec:
`test/stock-lot-cost-notes-92.e2e-spec.ts`. The step-0 revaluation table of §4 was NOT created
(no tracked lot had an outward on 2026-10-06). See [[stock-engine-ddl-lives-outside-repo]],
[[stock-adjustment-family]], [[pre-existing-test-failures]].

**Local DB state (2026-10-06 17:14 IST):** migration 20261006100000 was applied to the local DB as a side
effect of a `migrate deploy` for 20261006120000 (charge-basis column widening). The rebuild
(`POST /stock/admin/rebuild-costs`) had NOT been run at that point — check before assuming tracked rows
carry lot costs.

**Notes 93 (same day):** `lotlessOutwardLine()` treats MRP 0 / sale price 0 as not stated and a tracked
outward line whose identity matches no lot as lotless (→ `pickIssueLots`, narrowed by `issueNarrowing`,
refused when nothing matches); `resolveLots` opens a lot from an outward only for signature `N`. The
preflight's `has_stock` narrows identically. Cases "notes 93.1–3" live in the notes-92 spec. The phantom
lot `d7ee6f25-…` (MRP ITEM, MRP 0, −103 from bil00964/bil00967) is cleared by cancelling those two bills.

**Dev DB:** the real rebuild (`rebuildStockDerivedFigures`, scope {}) WAS run and committed on 2026-10-06
after the migration; only ZT-ITEM-A's total moved (+585.96); the assertion was clean afterwards.
Notes 92 itself was committed by a sibling session as d40e67cc.
