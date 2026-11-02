---
name: opening-balance-screen-built
description: "Menu 55 Opening Balances: the Qt screen is built, registered and tested live — the file set, the four rules the live API forced, and the three backend gaps found"
metadata:
  type: project
---

**BUILT AND VERIFIED 2026-09-15**, against the live server. Supersedes the
"no Qt screen" state in [[opening-balance-concept]]; the design in
`openingbal/plan-qt-opening-balance.md` held up and was followed.

**File set** (`src/modules/accounts/opening/`): `opening_balance_entity.h`
(both column enums + OpeningRow/OpeningBill + the Dr/Cr and date wire
helpers), `opening_balance_delegate.*` (ledger grid), `opening_bill_delegate.*`
(bill panel), `opening_balance_entry.h/.cpp/.ui`. Endpoints in a NEW
`src/core/api/endpoints/accounts_end_points.h` (`ApiEndpoints::OpeningBalances`)
— the house pattern for the four accounts screens still to come. Registered
`fm.reg<OpeningBalanceEntry>(55)` in `form_accounts_menu.h`; CMakeLists patched
by hand.

**Registered on the server while building:** `fixed.ui_tables` **30**
"OPENING BALANCE - LEDGERS" (15 cols) and **31** "OPENING BALANCE - BILLS"
(13 cols) via `/ui-table-masters/create`; menu 55 `menu_visiblity` flipped
true via `PATCH /menu-masters/visibility` (the DB role is read-only, so it
must go through the API).

**Test:** `tests/openingbalance/run.sh` — 38 assertions, the repo's existing
offscreen convention ([[qt-offscreen-test-harness]]). It NEEDS the server and
**seeds its own fixture** through the same API the screen uses (four ledgers +
one bill-wise party, Dr 350,000 = Cr 350,000) and leaves it balanced, so it is
repeatable. The binary must live in the build dir — AppConfig reads
appconfig.ini from `applicationDirPath()`. Private *methods* are not invokable
by `QMetaObject::invokeMethod` (it fails silently) — drive buttons instead;
only private *slots* work.

**Four rules the LIVE API forced, none of them guessable from the schema:**
1. `POST /create` **rejects a bill-wise row outright, even at amount 0**. So
   `buildLedgerPayload()` omits every bill-wise row — and `replace:true` does
   NOT lose them: they come back under `retainedWithBills`.
2. A party opens at the **NET** of its bills (5,000 DR + 3,000 CR = 2,000 D),
   not their sum.
3. `POST /bills` refuses a refno already live on the party unless its `ablId`
   comes too — so the hidden AblId column is load-bearing, not decoration.
4. `op_source` is **server-owned**: echoing CARRY_FORWARD onto a MIGRATION row
   does nothing. The client echoes it and repaints from the answer.

**The scope fact that shapes the whole screen:** `ux_op_scope` makes the
company-level set (`op_branch_id IS NULL`) and a branch's set two DIFFERENT
sets of rows — neither trial balance sees the other. With `abl_branch_id` NOT
NULL that means a bill-wise party's opening can ONLY live in a branch set, so
under "All branches" the breakup panel is closed and a banner names how many
parties are missing from the totals.

**Three backend gaps were reported and ALL THREE ARE CLOSED** (Prathap shipped
them the same day; re-tested green 2026-09-15) —
`openingbal/client_findings_2026-09-15.md`, notes.txt (11) then (12):
- `opRemarks` is on `OpeningBalanceRowDto` and round-trips. It had been stored
  but never returned; the client had deliberately NOT cached it to paper over
  that, **so the fix needed no client change at all** — one test assertion
  flipped. That is the argument for never keeping a client-side copy of a
  column the server owns.
- An absent `includeZero` now answers with the whole chart (it used to answer
  `rows: []`). The builders still send it explicitly and no longer depend on
  the default either way.
- The `replace:true` + `ablId` rule is now in the route description.

Re-probed after his commit and UNCHANGED, safe to rely on: all four rules
above, plus the reworded (behaviour-identical) bill-wise refusal message.

**THE PERMISSION TRAP (bug found and fixed 2026-09-15, worth never repeating):**
`FormManager::reg<T>()` builds the dialog with `new T(ctx.parent)` and
`openForm()` assigns `dialog->ctx.menuId` **only AFTER the factory returns**.
So a constructor that calls `setup()` ALWAYS runs with `menuId == 0`, `setup()`
returns before `loadPermissions()`, and every `ctx.can*` flag stays false.
Reading those flags as a refusal made the screen answer "You do not have
permission" to a user with every right on menu 55.

The house pattern is the `ctx.menuId == 0` escape hatch in
`applyPermissions()` (as `OpeningStockEntry` and `NexCrudForm` do) — an
unloaded permission is NOT a denied one. `AppSession::canCreate()` agrees: it
defaults to **true** for a menu with no entry. Never gate an action on
`ctx.canCreate` directly; gate on a flag set only when `menuId != 0`.

The test HID this by setting `ctx.canCreate = true` by hand. It now builds the
screen ungoverned and asserts it saves, then drives the real
`storeUserPermissions()` -> `setup()` path for denied and allowed. Grant keys
are **`umMenuId` / `umCanCreate`** (the flat user-menus shape), not the
`canCreate` names they are re-keyed into — wrong keys are ignored silently and,
with the default-true, look like a pass.

**THE CRASH: every ApiClient callback must be QPointer-guarded.** `NexDialog`
sets **`Qt::WA_DeleteOnClose`** (and `Qt::ApplicationModal`), so closing a
screen DELETES it — and an in-flight reply then calls back into freed memory.
Open menu 55, press Esc before the chart lands, segfault in
`QTableWidget::setRowCount`. Reproduced under gdb 2026-09-15. All six call
sites now capture `QPointer<T> self` and bail on `!self`;
`OpeningStockEntry::refreshLineSignatures()` already did this. Test section 8
clicks Reload then close() with no event loop between.

**WIDTH AND VISIBILITY BELONG TO `ui_tables`, NOT TO THE SCREEN** (user's call,
2026-09-15). `applyLayout()` writes visibility + header + width; a re-seed on
`layoutLoaded` puts hardcoded px and `QHeaderView::Fixed` back over the
operator's saved percentages, so widths "would not stick". Seed header text
and the hidden flag ONCE before the server answers, set no width and no resize
mode, and **do not reconnect `layoutLoaded`**. `NexTable::showEvent()` re-applies
the percentages against the real width, so a test must `show()` the screen or
every column measures `minimumSectionSize`.

**Tests must assert FAITHFULNESS to the layout, never a fixed visibility.**
Asserting "OpId is hidden" broke the moment someone unhid it in Layout
Settings — the layout doing its job.

**Double-click is how a cell is edited.** Wiring `cellDoubleClicked` to "open
the bill breakup" made an ordinary ledger answer "Not a bill-wise party"
instead of opening an editor. It now fires only on a bill-wise row and only on
Amount / DrCr / BillWise / BillCount. A plain ledger gets silence, not a
dialog, and selecting one closes the panel.

**Clear a detail panel BEFORE the async load**, not in the callback: the party
label changes immediately while the grid still holds the last party's rows.

**THE GRID LOADS THE OPENINGS THAT EXIST AND PICKS THE REST** (user's call,
2026-09-15, and he was right). `includeZero=false` — ledgers that already have
an opening plus prior-closing carry-forward candidates; everything else comes
from **grid 107 "POPUP - LEDGERS"** on the Ledger cell (`setPopupConfig`,
restricted to Assets/Liabilities, scoped by `iled_company_id`). Ins adds a
blank row, Ctrl+Del removes one.

The old full-chart design was defended with "a partial chart is a wrong trial
balance, not a slow one". **That is false** — the server answers the same year
with `includeZero` true/false as 31 rows and 5 rows and THE SAME TRIAL BALANCE,
because it totals the whole chart server-side regardless. A ledger with no
opening contributes zero to both sides. 1000+ debtors made the old way unusable.

**THE INVARIANT:** `replace:true` soft-deletes every opening absent from the
payload, so the grid must always hold EVERY row that has an opening.
`includeZero=false` guarantees it and the picker only ADDS. **Never page,
filter or trim the loaded set** — a search box would make Save delete whatever
scrolled out of view.

**Configured-grid SQL parameters are bare `i`-prefixed tokens, textually
substituted** (`NULLIF('iled_company_id','')::uuid`), as grids 47 and 87 use.
**`:name` is NOT supported** — tested: `:led_company_id` → "syntax error at or
near $1" (it rewrites to `$1` then fails to bind), `:iled_company_id` → "syntax
error at or near :". Reported to Prathap, notes.txt (14).

**NO ADD/REMOVE BUTTONS** (2026-09-15 evening). Both grids always end in ONE
blank row which IS the picker; picking into it grows the next. Removal is
`NexTable::setLineKeysEnabled(true)` + `lineCreateRequested`/`lineRemoveRequested`
(`+`/`-`), and NexTable refuses `-` on the last row itself. Every removal
CONFIRMS first. The trailing row is furniture: it does not dirty the screen,
the payload drops it, validation ignores it. **Nothing on the trailing row opens until a ledger is picked into it** — the
delegate refuses Opening/DrCr/Remarks while `ledId` is empty and paints them
grey, leaving the Ledger cell to NexTable's picker.

Footer is TWO stacked bands: `frameTotals` (a bordered card with the figures)
above hint · spacer · Save · Close. The totals keep their own band — crammed
onto the buttons' line they read as a caption, not a total. Save reads just
"Save".

**`AppSession::companyName()`** added — the login payload has only `company_id`,
so the name comes from `/company-masters/get?compId=` and is cached for the
other accounts screens.

**THE SCREEN FOLLOWS `opening_balance_ui_mockup.png`** (2026-09-15): title
"Opening Balances — <year>" + subtitle (company · scope · year) + a
BALANCED / OUT BY x pill top-right; "THE LEDGERS" / "THE BILLS" section
headings; figure bands with a small uppercase CAPTION over a large VALUE,
green when balanced and **the whole band** red when not; Save filled maroon
(#7B1113), everything else outlined.

**The FOCUSED cell is painted dark with a 2px border, by the DELEGATE.** It
cannot be a stylesheet: `app.qss` sets `QTableView::item { border: none }` and
`outline: 0`, and `::item:focus` does not follow the current index — Qt gives
the item no focus pseudo-state. `QStyle::State_HasFocus` on the style option IS
the current cell and only the delegate sees it. Force `State_Selected` on and
repoint `QPalette::Highlight`/`HighlightedText` so the text stays aligned and
elided as normal, then draw the border AFTER the base paint.

**Chips are PAINTED, not styled** — Source / Bill-wise / Stale / Status.
A `QTableWidgetItem` has no border-radius, so a stylesheet can colour a column
but cannot draw a pill. Clear `opt.text`, let the base paint the background,
then draw the rounded rect. Amber = the operator did it or must look
(MANUAL/STALE/PARTIAL/FROZEN), blue = the system derived it (CARRY FWD/
BILL-WISE/MIGRATION), green = settled (OPEN), grey = unrecognised.

**Do not assert an exact grid row count** — and do not assert WHICH column is
last visible, or that a data-driven banner is present. Both broke the same way:
the layout hid Remarks, and someone fixed the last NULL-nature group. Assert
the property (columns fill the viewport; the banner matches the data), never a
hardcoded expectation.

**The test fixture collides with a live session.** It seeds and leaves a
balanced set on saw mill / 2026-2027; if anyone is using menu 55 at the same
time, both sides' `replace:true` saves delete each other's rows. Three runs
failed this way before it was recognised — re-run before believing a failure. The chart grows: seeding the ledger
role map added ten balance-sheet ledgers and turned `rowCount() == 21` into 31
overnight.

**Still open:** DECISION 1 (`led_ob_*` vs `acc_opening_balance`) is untouched;
no OPENING_DIFFERENCE or RETAINED_EARNINGS ledger is mapped on the test
company, so the plug is named as unmapped rather than offered.

**2026-09-29 bill side default:** addBillRow() used to default every new bill to DR ("common case is a
debtor"). KL ELECTRICS (Suppliers, Liabilities) got 4 opening bills keyed DR, so Payment listed them as
held DEBITS, found no bill to pay and put the money on account. Now: Liabilities nature → CR, else DR
(test check added, 81/0). Payment shows a "No bill to pay" banner when a party has only DR rows.
