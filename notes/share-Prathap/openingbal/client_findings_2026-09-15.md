# Opening balances — what the Qt client found when it was wired up

> **ALL THREE CLOSED, same day.** Prathap shipped the fixes and the client was
> re-tested against them on 2026-09-15: `opRemarks` is on `OpeningBalanceRowDto`
> and round-trips; an absent `includeZero` now answers with the whole chart
> (`false` correctly narrows to 5 rows here); the `replace:true` + `ablId` rule
> is written into the route description. The Qt side needed **no code change**
> for any of them — only one test assertion flipped, from "the remark is stored
> but never returned" to "the remark round-trips".
>
> Re-probed at the same time and unchanged: `/create` still refuses a bill-wise
> row at amount 0 (message reworded, behaviour identical), `replace:true` still
> retains an omitted bill-wise party under `retainedWithBills`, `op_source` is
> still server-owned, and a live refno still needs its `ablId`. The sections
> below are kept as the record of what was wrong and why it mattered.

**2026-09-15.** The screen (menu 55) is built, registered and tested against
the live server on 192.168.0.106. Every one of the six routes answers, and the
contract in `plan-backend-opening-balance.md` held up — these are the three
places where the live behaviour is not what a client would assume from the
Swagger, plus one field that is missing. Each was reproduced against
`https://192.168.0.106:3011/api/v1` as VKPOS; none is a design argument.

The client works around all three today. Two of them should be fixed anyway,
because the next client (the React port) will hit exactly the same three.

---

## 1. `/list` and `/trial-balance` never return `opRemarks` — FIX THIS ONE

`op_remarks` is accepted by `POST /create`, written correctly, and readable in
the table:

```
select l.led_name, o.op_remarks
  from accounts.acc_opening_balance o
  join accounts.acc_ledger_master l on l.led_id = o.op_ledger_id
 where o.op_acc_year = '2026-2027' and not o.op_is_deleted;

 Cash In Hand | counted at go-live      <- stored
```

but `OpeningBalanceRowDto` has no `opRemarks` field, so `GET /list` cannot
hand it back. The screen has a Remarks column, sends what is typed in it, and
then **paints it blank on the next reload** — the operator's note is on the
server and invisible.

That is the worst shape a field can have: not lost, not shown. A remark on an
opening is exactly the thing somebody needs six months later ("why is Cash In
Hand 4,17,000 when the sheet says 4,00,000") and it is being written into a
hole.

**Add `opRemarks` (nullable string) to `OpeningBalanceRowDto`.** One column on
a select. The client already has the column, the enum slot and the payload
field; nothing on this side has to change when it arrives.

## 2. An ABSENT `includeZero` is not the same as `includeZero=false`

```
GET /opening-balances/list?companyId=…&accYear=2026-2027                  -> rows: []      (0)
GET /opening-balances/list?companyId=…&accYear=2026-2027&includeZero=false -> rows: [ … ]  (21)
GET /opening-balances/list?companyId=…&accYear=2026-2027&includeZero=true  -> rows: [ … ]  (21)
```

The documented job of the flag is only to decide whether a ledger with no
opening AND no prior closing is listed. Whatever the intent, an omitted
`@Query` is arriving as `undefined` and taking a branch that filters
everything out. An empty chart is not a slow answer to a review screen — it is
a wrong trial balance that looks like a correct one, because 0 = 0 balances.

The client always sends it explicitly, so it is not blocked. But the default
should be the safe one: **treat absent as `true`**, or give the DTO an explicit
default. A route whose safest parameter value is the one you must remember to
type is a route that will eventually be called without it.

## 3. A refno already on the party may only be re-sent WITH its `ablId`

`POST /opening-balances/bills` with `replace: true` and a bill carrying a
reference that already exists live on that party is refused:

```
400  bills.0.ablDocRefno
     "ADV/2025/118" is already an opening bill of this party for this year.
     Send its ablId to update it, or use a different reference.
```

This is CORRECT and the message is a good one — `ux_abl_doc_refno` is the
constraint and the alternative would be silently creating a duplicate
invoice. It is written down here only because it is not obvious from the
Swagger, and because it makes `replace: true` **not** a "here is the whole
breakup, sort it out" call: a client that rebuilds the array from a grid
without carrying each row's `ablId` through will be refused on every bill it
loaded. The Qt screen keeps `ablId` in a hidden column for exactly this.

No change asked for. Worth one line in the route description.

---

## What was verified working, so nobody re-tests it

All against the live server, through the screen itself (`tests/openingbalance/`
in the Qt repo, 37 assertions, seeds its own fixture and leaves it balanced):

- `/list` returns the whole balance-sheet chart (21 ledgers here), income and
  expense correctly absent, `unclassified[]` carrying the one NULL-nature
  ledger.
- `POST /create` **refuses a bill-wise row outright, even at amount 0**, with a
  message naming the ledger. Exactly right, and the client now omits those rows
  rather than sending zeros.
- `replace: true` does not destroy a bill-wise party that was omitted — it
  comes back under `retainedWithBills`. This is the thing that makes the
  omit-bill-wise-rows rule safe, and it works.
- `POST /bills` owns the party's figure and rewrites `op_amount` in the same
  transaction; the party opens at the **net** of its bills (5,000 DR and
  3,000 CR opens it at 2,000 D, verified), not their sum.
- Validation messages are specific and in the operator's language throughout —
  negative amount, due date before doc date, duplicate refno within one
  payload, zero bill amount, missing branchId. The client shows them as they
  come; none needed rewording.
- `skippedZero` / `deleted` / `flippedToManual` / `staledAccYears` all report.
- `op_source` is server-owned: a client echoing `CARRY_FORWARD` onto a
  `MIGRATION` row does NOT move it. Good — the client treats the echo as
  advisory and repaints from the answer.
- `carry-forward` runs and reports honestly on an empty prior year
  (`created: 0`, and it names the missing RETAINED_EARNINGS mapping).

## The one thing that is a DESIGN consequence, not a bug

`ux_op_scope` is `(company, COALESCE(branch, 0-uuid), year, ledger)`, so the
company-level set and a branch's set are **two different sets of rows** and
neither trial balance can see the other. Verified: three ledgers saved at
company level and a bill-wise party saved under a branch produced two
separately-balanced sets sharing nothing.

Combined with `abl_branch_id` NOT NULL, that means **a bill-wise party's
opening can only ever live in a branch set.** The company-level screen
therefore cannot show it, and any company-level trial balance is structurally
incomplete the moment one debtor opens with bills.

The client handles it the only honest way: "This branch" / "All branches" are
presented as two sets rather than a filter, the breakup panel is closed under
"All branches", and a banner names how many parties are not in those totals.
Nothing to change server-side — but if anyone later asks why the company-level
opening "does not add up", this is the answer.

## Registered while building (already on the server)

- `fixed.ui_tables` **30** "OPENING BALANCE - LEDGERS" (15 columns) and
  **31** "OPENING BALANCE - BILLS" (13 columns), via `/ui-table-masters/create`.
- Menu **55** `menu_visiblity` flipped to true via
  `PATCH /menu-masters/visibility`.
