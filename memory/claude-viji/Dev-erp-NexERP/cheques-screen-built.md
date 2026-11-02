---
name: cheques-screen-built
description: Menu 51 Received Cheques Qt screen is coded, registered, building and tested offscreen (2026-09-17) — file set and the two traps it encodes
metadata:
  type: project
---

The received-cheques register (menu 51) is built, in `src/modules/accounts/cheques/`:
`cheque_register_entity.h` (ChequeRow + the status→allowed-actions map, shared with menu 52),
`cheque_action_panels.h/.cpp` (one `ChequeActionPanel` base and six subclasses answering
endpoint/payload/validate), `cheque_history_dialog.h/.cpp`, `received_cheques.h/.cpp/.ui`.
plus `cheque_row_delegate.h/.cpp`. Registered `fm.reg<ReceivedCheques>(51)`, added to
CMakeLists by hand. TWO test suites, both green offscreen against the live server, each
seeding its own receipt fixture: `tests/cheques/run.sh` (34 assertions — shape, the
status→action map, validation, no writes) and `tests/cheques/run-ops.sh` (34 — drives the real
buttons through deposit → clear, deposit → bounce → re-present, return, and a forced 409;
every result asserted against the SERVER's row, not the screen's labels).

The backend was already there and did not need asking for: **grid 109 "MAIN LIST - RECEIVED
CHEQUES"**, menu 51, dropdowns 25 (BANK LEDGERS) and 43 (LEDGERS - ALL) all exist.

**Two traps, both encoded in comments at the call sites:**

1. Grid 109's SQL names nine BARE parameters (`iapd_company_id`, `istatus`, `ifrom`, `ito`,
   `ibank_ledger_id`, `iparty_id`, `isearch`…). PostgreSQL resolves all of them at prepare
   time, so omitting one is a 400 (`column "istatus" does not exist`), not a filter ignored.
   `applyFilters()` sends all nine every time, `""` meaning no filter.
2. The search box must set **`isearch` inside grid_param** — NOT the `search` key that
   `NexApiTableView` lifts out into its own query item. `search=` with no `isearch` returns
   zero rows AND no meta: an empty register that looks filtered.

Design decisions worth keeping: the register is grid 109 (layout lives in `fixed.grid_columns`)
while `GET /cheques/list` is called once per refresh for its **summary only** (whole-register,
ignores filters — exactly what a strip wants); "what is being acted on" is *checked rows if any,
else the current row*, and the bar enables only what EVERY row allows; Esc closes the panel
before the screen.

Also fixed in passing: `ApiEndpoints::Receipts::get()` was sending bare `companyId/branchId/
accYear/voucherId` and the server refuses them ("property companyId should not exist"), so Enter
on a row of the receipt F8 list could never open it — now `avh`-prefixed. Added
`ReceiptEntry::openFor(key, parent)` for cross-screen drill-through.

**Three bugs the operations run found and fixed**, all in the client:
1. A reload leaves the current index where it was, so re-selecting "the same" row emits no
   `currentRowChanged` — the detail strip, cleared by the reload, stayed blank until the
   operator moved off the row and back. It is the only place the bills a cheque settled are
   named, so somebody about to bounce one was looking at an empty box. `restoreSelection()`
   now re-reads under the cursor itself instead of waiting to be told.
2. `uic` SILENTLY DROPS `<property name="stretch">` on a QVBoxLayout mixing nested layouts and
   widgets — the generated header had no `setStretch` at all. Opening a panel squeezed the
   register to one row. Set in code in `tuneLayout()`.
3. A QStackedWidget is as tall as its TALLEST page, so the four-field Deposit panel opened at
   the eight-field Replace panel's height. `sizeStackToCurrentPage()` gives every hidden page
   an Ignored size policy.

**UI pass for visibility:** the summary strip is four colour-coded tiles (amber in hand, blue
with the bank, red bounced, green cleared) plus a slate FIFTH tile carrying the Cheques In Hand
figure the other four must tie to — it was a cramped three-line sentence in the corner.
`ChequeRowDelegate` overrides `displayText()` so pills read "In hand" / "With bank" / "Matured"
while `TxnBadgePalette` still keys colour off the RAW value. Badge overrides fix two wrong
defaults: BOUNCED falls in the palette's Danger bucket, which dims AND strikes through the row —
wrong for the most actionable row on the screen (red yes, dead no); DEPOSITED and REPLACED were
in no bucket and got hashed arbitrary hues.

Still owed: the deposit-slip PRINT (waits on the printing module — F6 currently reports the
dataset), and the ON_CLEARING allocation grid in the Clear panel (waits on `ReceiptBillsBand`
being extracted from the receipt screen; empty allocations asks the server for auto-FIFO, which
is the same default). See [[cheques-api-test-results]] for the three server defects, one of
which — the tender-less replacement row — the screen deliberately does NOT try to predict.
