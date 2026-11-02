---
name: stock-e2e-suites-run-now
description: "opening-stock, physical-stock and stock-transfer e2e suites no longer skip (they checked for SQL functions); they are order-dependent and reset via afterEach cancels; the transfer suite raises a temporary second branch; four HTTP transfer suites are partly stale"
metadata:
  node_type: memory
  type: project
  originSessionId: 04f8789e-47a6-43d6-8027-632e2629b5c4
  modified: 2026-09-29T03:00:00.000Z
---

Until 2026-09-28 `test/opening-stock`, `physical-stock` and `stock-transfer` e2e suites
always SKIPPED (their `detectEngine` looked for `fn_svh_post` etc.), so their expectations
had never run. They run now against localhost and pass (11/15/14) after these harness
fixes: an `afterEach` that cancels every POSTED voucher the test created (each test
re-posts the same holdings, and only a reversal frees them — [[stock-ledger-cancel-leaves-forward-row]]);
the physical fixture inserts an ITEM batch policy for MILK so B-2604 shows on the sheet;
the opening fixture sends `lineCount/totalQty/totalValue` (a draft's totals are the
screen's); the transfer fixture opens 100 MILK (the flow's 55 was consumed by the later
cases) and, when no company has two branches with a device and a godown, INSERTs a
temporary branch + godown + device and removes them in cleanUp. `stock-engine-ts` (rolled
back) gained cases 11–14 (accounts legs, PERIODIC, lot pick) and calls
`accounts.fn_seed_ledger_map()` inside its transaction.

**Why:** a green suite that never ran is worse than no suite; the plan's §1.1 tests had to
actually execute.

**How to apply:** run `npx jest --config ./test/jest-e2e.json test/<name>` one suite at a
time (they share the DB). `-t` filters break order-dependent cases — case 14 needs its own
policy, cases 11/12 need MILK batch / TEA batch identity. The HTTP suites
(`stock-transfer-http`, `transfer-all-endpoints-tables`, `transfer-in-preflight`,
`transfer-sql-trace`, `transfer-receive-real-data`) need a running app + token and use
REAL data; their "engine is missing" despatch probes are skipped or widened, not
re-pointed, because a live despatch would move real stock on the dev DB.

`stock-transfer-http` fails its "setup: seed 10 units" step on the local DB since at least
2026-09-29: its hard-coded LOT `01a08502-d594-72c8-8e94-d79166ac46dc` no longer exists (no
stock_lot row, no ledger rows), so the count save 422s "names a holding this godown no longer
has". Stale fixture data, not an engine regression — re-point the constants before blaming a change.
