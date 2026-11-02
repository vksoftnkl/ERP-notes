---
name: pre-existing-test-failures
description: unit suite ALL GREEN since 2026-10-23 (495/9254 after notes 160, 2026-10-27) + tsc clean — any unit failure is new; older failure lists kept as history
metadata: 
  node_type: memory
  type: project
  originSessionId: 46b78f79-1f67-4281-bb21-c6013dcdfa37
  modified: 2026-10-26T04:55:44.110Z
---

**E2E on dev data, 2026-10-24:** `test/physical-stock-http` (post 422 STOCK_LINE_COST_UNRESOLVED) and
`test/physical-save-and-post-http` (400) both fail because the FIRST holding on the Coimbatore
count sheet (godown 019daae6-…) is a leftover e2e item `E2E-QTR-MUF90PCV` with on-hand −4 and avg
cost 0 (created 2026-09-24): an overage on it cannot be valued / countedQty goes negative. Data, not
code — clear that holding and both pass. `test/physical-create-sql-trace` is a DEBUG spec that needs
`TRACE_OUT=<file>` or it TypeErrors. Validate says clean while post refuses the 0-cost overage —
a pre-existing validate/post discrepancy on cost, not touched by notes 150.

**Latest baseline, 2026-10-23 (transpile-only jest, see [[unit-jest-is-transpile-only]]):**
`npx jest` = **ALL GREEN — 478 suites / 8924 tests, ~11 s**, and `tsc --noEmit` clean. Any unit
failure now is new. Re-run 2026-10-23 12:30 after notes 142 (txn-adjacent, +3 suites / +65 tests): 481 / 8989, all green. 2026-10-26 after notes 153 (rev 2): 491 / 9180, all green. 2026-10-26 19:20 after notes 157/159: 495 / 9231, all green. The last six (notes 140 price periods) were spec-side mock gaps, fixed
the same day: anything reading prices "today" calls `databaseToday()` → `$queryRaw` (give the
mock `[{ today }]`, or jest.mock the helper when `$queryRaw` also serves other rows);
selling-price-bulk saves call `gateway.calendar()` first; an item-price UPDATE derives TWO
bucket keys (deriveBuckets mock must map rows → keys). Still uncovered by unit specs: the
gateway's schedule methods (calendar, insertSplitPrice, listScheduled, …) and the service's
split / listSchedule / cancelSchedule paths. The older lists below are history — all fixed.

As of 2026-09-16 on branch DEV-v1, `npx jest` reports **13 failed suites / 15 failed
tests** (1889 pass, 1904 total) on a pristine tree. Re-verified by stash-and-rerun on
2026-09-16; the count was 12 suites / 14 tests on 2026-08-27, so it drifts upward as
other work lands — re-establish it rather than trusting the number here.

- ~~`modules/auth/auth.service.spec.ts` (2)~~ FIXED 2026-10-23. The `userMaster.update is not a function` lines are swallowed WARNs, not the cause; the spec was stale on three deliberate service changes: response `user_name` = login name (6eff6a52), the channel `OR` filter when no device_type is given (a2b1aec0), and `branch_id`/`device_id` claims in the token
- nine `*-exception.filter.spec.ts` files under `Inventory/` and `sales/loyalty/`
  (all the same `httpContext.getRequest is not a function`)
- `modules/sales/bill/bill.service.spec.ts` (sale-order line status/pending-qty expectations)
- ~~`modules/sales/quotation/quotation.service.spec.ts` (2 — quotation numbering)~~ FIXED 2026-10-21: spec still said voucher type 21; the service moved to 2 in 9d73417b (2026-08-08)
- ~~`modules/sales/sale-order/sale-order.service.spec.ts` (cheque register)~~ FIXED 2026-10-21 (see below)

**Why:** a full-suite run looks alarming after any change, and the reflex is to assume you
broke something. These fail on their own.

**How to apply:** before investigating a failure, confirm it is not in this list. To be
certain, snapshot your edited files to the scratchpad first, then
`git stash push -- <just the files you touched>`, re-run, and `git stash pop` — a
targeted stash beats `--include-untracked -- src/` because other sessions may be editing
this repo at the same time ([[concurrent-claude-sessions]]). Note a full `npx jest` run
rebuilds tracked `dist/`, so check `git status` afterwards.

Re-verified 2026-09-17 while adding `POST /receipts/delete`: still exactly **13 suites /
15 tests** (1889 pass, 1904 total). The count has held for a day.

**Stock engine e2e, 2026-10-15:** `stock-engine-ts` (cases 3–7 + notes 77) and
`stock-lot-cost-notes-92` (cases 2–6, 8) — 12 tests — fail identically on a copy of the
working tree with the posting helper reverted to HEAD. Every one dies at the count
sheet (`countSheet` returns no rows for the fixture item), before any costing runs.
A rsync copy of src/test + symlinked node_modules in the scratchpad is a safe
baseline that never touches the shared tree.

## `npm run lint` is not clean either, and that is also pre-existing

The script is `eslint "{src,test}/**/*.ts" --fix`, so it covers `test/` — and **every
e2e spec under `test/` fails it**, with dozens of
`@typescript-eslint/no-unsafe-member-access` / `no-unsafe-argument` errors on
`res.body.…` and `app.getHttpServer()`. Two existing files alone
(`receipt-open-items-credits`, `bill-draft-http`) account for 60. They are not
auto-fixable — `--fix` does not touch them.

**How to apply:** lint `src/` only when checking your own work
(`ESLINT_USE_FLAT_CONFIG=false npx eslint --ext .ts src/<path>` — ESLint 9 finds no flat config and refuses without the flag). `src/` is NOT fully clean any more (seen 2026-10-23): ~170 `prettier/prettier`
errors in sales + Inventory specs, all one pattern — `code: 'VALIDATION_FAILED', message: …` jammed on one
line by an old bulk edit. Compare against `git show HEAD:<file>` before claiming an error; prettier --write
only the files you touch.

**Quiet runs:** a unit run prints NO `ERROR [ModuleExceptionFilter]` stacks since 2026-10-23. A filter
spec's 500 case must `jest.spyOn(Logger.prototype, 'error').mockImplementation(() => undefined)` (and
`afterEach(() => jest.restoreAllMocks())`), else the deliberate "boom" prints a stack that reads like a
failure. LOG/WARN lines are dropped for every unit spec by test/unit-logger.setup.ts
(`Logger.overrideLogger(['error'])`, package.json jest `setupFilesAfterEnv`; `TEST_LOGS=1` prints them
again) — so a unit run prints ZERO `[Nest]` lines, and any that appears is an unexpected error. If
you add an e2e spec, match the surrounding style rather than typing `res.body` to
silence the rule — a single clean file among twenty would be the odd one out, and
nothing in the repo enforces it.

Related: [[prisma-schema-is-generated]], [[concurrent-claude-sessions]]

## Two receipt e2e suites fail on their own too (seen 2026-09-29)

`test/receipt-represented-cheque-guards.e2e-spec.ts` (5 tests, 422 ACC_PARTY_OUT_OF_BALANCE — its
fixture writes opening bills without the matching `acc_opening_balance` row, so the books check
refuses the post) and `test/receipt-open-items-credits.e2e-spec.ts` (2 tests, read-only against
live dev data that has moved on). Verified failing identically on HEAD code by swapping the files
back. Every other receipt-*/payment-* suite passes.

## cheques-bounce-flow fails 17 of 19 (seen 2026-09-29)

`test/cheques-bounce-flow.e2e-spec.ts` (RECEIVED cheques): 11× 422 and 2× 409 from the first
step (`/cheques/deposit`) on. Identical on a clean `git worktree add --detach <scratch> HEAD`
at 65e413e7 (symlink node_modules, copy .env, run jest there, then `unlink` the symlink BEFORE
`git worktree remove --force`), so not caused by notes 63/64. Not investigated further.

## Two opening HTTP suites and a trace spec fail on their own too (seen 2026-09-30)

- `test/opening-cancel-http.e2e-spec.ts` "409 when the target is a DRAFT" gets 201 — stale:
  `POST /stock/opening/cancel` cancels a DRAFT by design now (opening README route table).
- `test/opening-balance-http.e2e-spec.ts` "null differenceLedgerId" — accounts opening balances, live
  data now maps OPENING_DIFFERENCE; also logs a unique-constraint prisma:error on acc_bill_balance.
- `test/physical-create-sql-trace.e2e-spec.ts` is a debug tracer: needs `TRACE_OUT=<file>`.
- A multi-suite `test:e2e` run can die with ioredis "Connection is closed" (bullmq teardown) on
  whichever suite starts next — rerun that suite alone before believing it.

## app-theme token count fails on its own too (seen 2026-10-02)

`test/app-theme.e2e-spec.ts` "/effective: a company with a live theme gets it" expects 33 token
keys and gets 38 (the theme token set grew with 20261001140000_app_theme_template; the spec was
not updated). Verified identical with the notes 78 edits stashed. The other 6 cases pass.

## Opening / physical stock e2e: the branch balance assertion (seen 2026-10-03)

`test/opening-stock.e2e-spec.ts` (1) and `test/physical-stock.e2e-spec.ts` (2) fail at
`expect(await rebuild()).toBe(0)` with 1 differing holding. `assertStockBalances` checks the WHOLE
branch, and live item "001" (019fa296-42f8-…) has `ITEM_COST_QTY` stored -106.945 vs derived -107.945
— dev data, not the fixtures. Everything else in those suites passes.

## loyalty-concurrency fixture collides with live data (seen 2026-10-05)

`test/sales/loyalty-concurrency.e2e-spec.ts` picks the first customer of the company and
inserts a wallet for it, but that customer (LISTED_CUSTOMER, "new customer") has held a wallet
since 2026-09-22 — `ux_lmb_cust` refuses (23505). Pre-existing data, not code. The rolled-back
`loyalty-ledger.e2e-spec.ts` and the single-writer spec pass. A new loyalty fixture should pick a
customer with NO wallet (see loyalty-status-http.e2e-spec.ts).

## bill-tender-http first case fails on HEAD too (seen 2026-10-05)

`test/sales/bill-tender-http.e2e-spec.ts` "a walk-in owing 500 needs a TEMP_CR tender … 422 without
one" gets 201: the walk-in shortfall rule (`bill-lifecycle.service.ts` ~687) fires only when
`snap.custId === settings.defaultCustomerId`, and the live default-customer setting no longer names
the harness WALK_IN. Verified by stashing the three services notes 90 touched and rerunning that
single case. The other 8 cases pass.

**2026-10-08:** `test/sales/bill-tender-http.e2e-spec.ts` › "a walk-in owing 500 needs a TEMP_CR tender … 422 without one" fails on HEAD. The post answers 201, not 422 SALES_TEMP_CREDIT_DETAILS_MISSING, and it fails with the till changes reverted too, so it is likely from commit 273a8aa4 ("silence bill guard warnings"). The other 8 tests of that suite pass.

2026-10-12 additions (seen while building notes 106, independent of it): `test/bill-draft-http` 2/12 fail (queries a dropped `sb_posted_on` column; expects `sbiItemName` null on create, now resolved); `sale-order.service.spec` 4/… fail in advance-receipt posting (expects `avhTotalDebit/Credit` and POSTED on the mocked create — the totals are DERIVED now). Every sales e2e suite died at `postOpeningStock` with "Device not found" until the harness got its own device row (see [[save-and-post-one-call-notes-106]]).

**Till day-close e2e, 2026-10-15:** `till-day-close-http` fails 5/8 on a clean HEAD worktree
(4f1ea5db). The first failure is the notes 121 line `cash_expected − cash_expected_counted`
= 0 ≠ 500: the test calls `TillDayHeaderProvider.resolve()` outside a request, so the live
session's expected figure is hidden (no caller → no 273 OVERRIDE); the other 4 cascade from
the live session the first test never closes. A `git worktree add --detach <scratch>/wt HEAD`
with node_modules symlinked and .env copied runs a suite against HEAD without touching the tree.

**sale-order.service.spec FIXED 2026-10-21** (9 failures → 0, 261/261 in sale-order): stale
DRAFT/derived-totals expectations rewritten, `$executeRaw` added to the mock (re-sync calls
`retireMoneyLineRows`), and a date time bomb: the cheque fixture is dated 2026-09-08 while
`soOrderDate` defaults to TODAY, so once the calendar passed it the cheque read as backdated
(CHEQUE_BACKDATED). Expect the same class of failure in other sales specs whose fixtures
(2026-08-08 / 2026-09-xx) meet a doc date that defaults to now — pin the doc date in the dto.

**2026-10-21, sales + reports fully green (125 suites / 2792 tests)** after fixing 4 stale specs
(all spec-side, no service change): loyalty-ledger lot order (row order, EARN reversal first),
loyalty exception filter plain 4xx → HTTP_404 (notes 127), ledger-statement DTO (class-validator
lists a subclass's own field first; `{}` is refused whole under `request`, so "missing" = keys set
to undefined), loyalty-status (summary's `red` CTE also reads `FROM filtered f`; base's gift
lateral holds a legit `LIMIT 1`).

**2026-10-23, accountsModule fully green (45 suites / 709 tests)** via `./scripts/test-modules.sh accountsModule`.
Spec-side fixes only: `voucher-derive.spec` predated notes 126 (typed cash/bank lines on RcpV/PmtV/Jrl, Contra
lines without `tenderId`) — its RcpV/PmtV fixtures now mirror live (`instruments: true`, MANY) and pay by
instrument; `books-reconcile.guard.spec` TS7006 fixed by typing the `statements()` rest as `unknown[]`. The
"worker failed to exit gracefully" line at the end of that run is a teardown warning, not a failure.
