---
name: ledger-map-screen-built
description: "menu 250 built and live-tested 2026-09-19: the file set, grid 105's missing filter flags, the almId-is-not-optional rule, and the one backend row still owed"
metadata:
  type: project
---

**Built, registered and live-tested 2026-09-19** — commits `ed5f75e` + `7cc5a73`. The plan
in [[ledger-map-screen-plan]] is done; Prathap's backend landed first and all
three routes answer as its plan said.

**Files.** `src/modules/accounts/ledger_map/` — `ledger_map_entity.h`
(columns + `LedgerMapRow`), `ledger_map_delegate.*`, `ledger_map_entry.*`,
plus `namespace LedgerMap` in `accounts_end_points.h`. `tests/ledgermap/run.sh`,
47 checks, all passing.

**Menu 250 exists** (added by the user 2026-09-19) — "Ledger mapping" under
**CONFIGURATION**, not `&5 Accounts`, position 8.10. VKPOS has view/create/
edit/delete on it in `user_menus`. `fm.reg<LedgerMapEntry>(250)` stays in
`form_accounts_menu.h`: FormManager keys on the menu id alone.

**`ui_table 37` "LEDGER MAP - POSTING ROLES" is registered** (2026-09-19,
`POST /ui-table-masters/create`, `replaceColumns: true`). Before it existed
the screen had headers and no widths — every column at `minimumSectionSize`
48 with the last swallowing the rest, which is what a NexTable looks like
with no layout row, not a screen that forgot to size itself.
- widths are **percentages of the table width**; the seven visible ones sum
  to **95, not 100**, because `stretchLastSection` is on and the remainder
  should land in Remarks rather than make a horizontal scrollbar.
- **`uiTblClmColumnFocus` is the Enter-stop flag**, and only the two EDITABLE
  columns (Ledger, Remarks) carry it. With no layout NexTable falls back to
  *every visible column*, so Enter walked through five read-only cells before
  the next row.
- `GET /ui-table-masters/get` takes **`uiTableId`** (not `uiTblId`) and its
  `data` is a **LIST** holding the one table. Both mistakes fail as an empty
  column list, which reads as "no layout".
- Adding a column is now NOT local: `fixed.ui_table_columns` is keyed by
  column NUMBER, so a new one goes on the END or the layout is renumbered in
  the same breath ([[receipt-screen-revision]]).

**What the live probes settled, and is not in either plan:**
- **`almId` is not optional once a role has one.** Create-or-update selects on
  its PRESENCE, and re-pointing a mapped role without it is a **409** naming
  the role. A delete-then-create issues a **NEW almId**, so the screen reloads
  from `/roles` after every save rather than keeping the ids it held.
- **The expectation is THREE fields, not one.** `expectedLedgerType` /
  `expectedDutyHead` / `expectedGroupNature`, any of them null. OUTPUT_CGST is
  TAX + Central Tax with no nature; ADVANCE_RECEIVED is Liabilities and
  nothing else. The local backstop checks only the TYPE — the other two are
  not on the picked row.
- **`isActive` is echoed, never offered.** A parked mapping does not resolve,
  which is a silent unmap that walks past `/delete`'s in-use guard. The screen
  shows it ("Off") and counts it as broken; there is no toggle.

**Grid 105 had SQL, live columns and ZERO filterable columns** — it answered
200 and found nothing the moment you typed, and NexTable passes the first
keystroke through as the search. Fixed by flagging `led_name`, `led_short`,
`acc_group_name` via `PUT /grid-details/filter-settings`. That is a THIRD way
a registered grid looks dead, beside the ones in [[nextable-grid-traps]], and
the suite now asserts it. Grid 106 stays registered and unused: it carries the
INTRA/INTER pair but not the label, the expectations or usedBy.

**The test writes to PURCHASE_RETURN only** (usedBy is empty, so nothing posts
it), captures where it started and puts it back; DISCOUNT_ALLOWED is exercised
through refusals alone. The box ends at 28 live rows, all resolving.

**Two test-harness traps, both cost a run:**
- The modal closer rejects every visible modal that is not the screen under
  test, so nulling that pointer before a `close()` that raises "Discard
  changes?" makes the two bounce off each other forever at 120ms a turn —
  a silent hang, not a crash. Set `modalAnswer = true` first.
- **`AppMessage` sets no window title.** Reading `windowTitle()` alone records
  an empty string for every modal; gather the child `QLabel`s too, the way
  `tests/shipaddr` does. Four assertions passed on behaviour and failed on the
  title until it did.

**Follow-up, not done:** `ReceiptRoles` in `receipt_entity.h` is still a second
hand-maintained copy of the role list. Pointing the receipt's Type combo at
`GET /ledger-map/roles` is why that route exists at all.
