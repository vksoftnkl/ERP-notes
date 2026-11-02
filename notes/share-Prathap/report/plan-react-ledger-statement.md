# Ledger Statement — React plan

Scope: **one routed report screen**, Reports › Financial Statements › Ledger
Statement. It covers one ledger over one date range inside one fiscal year,
with three tabs (Vouchers / Daily / Monthly), a drill-down into the source
voucher, and Excel export. It writes nothing.

**This is not a port.** There is no Qt screen for it yet, and 3.0's
`pos/Reports/ledgerreport.*` is only a reference for the gestures (Search,
Reset, Print, Export, WhatsApp). Nothing has to be un-learned. The job is to
avoid inventing anything the server already decides.

Companion documents that this plan does **not** repeat:

- **`plan-backend-ledger-statement.md`** (this folder): the seven routes, the
  balance definition (§4), the cancelled-pair rule (§7), how particulars and
  bill refs are derived (§8), error codes (§9), and decisions L1–L7.
- **`ledger_report_ui_mockup.png`** (this folder): the screen. Section
  numbers below refer to its bands.
- `../react-plans/plan-react-dropdown.md`: the searchable popup picker. The
  ledger combo reuses the component, **not** its grid-runner data source (§3.2).
- `../react-plans/plan-react-printing.md`: Print / PDF / WhatsApp wait for it
  (§11).

> **The whole screen in one line:** the client asks, formats and navigates.
> **It does no money arithmetic at all.** Every balance, total, running figure
> and b/f arrives computed from the server, and the client's only job with an
> amount is to print it.

---

## 1. What this screen is, and what it is not

It is a **read-only report**, so it has no draft, no dirty state, no save and
no lifecycle. That makes it the smallest accounts screen in the plan set. It
is also the one whose numbers get quoted back to Prathap as bug reports, so
every figure on it must be the server's own.

Three facts shape everything below:

1. **Its routes are its own.** Every call goes to
   `/api/v1/reports/ledger-statement/*`. The user's rule (2026-09-25) was
   *"separate urls, don't use any old urls because it's gonna change"*. So the
   ledger picker does **not** use dropdown 39 or the grid runner, and the
   ledger facts do **not** come from `/account-ledger-masters/get`, even
   though both would work today.
2. **CANCELLED vouchers count.** A cancel leaves the original voucher
   (CANCELLED, legs intact) and adds a POSTED mirror. Both are real rows, and
   both move the balance. The client never filters them out itself.
   `includeCancelled=false` is a server parameter (§6.2), because hiding one
   half of a pair must also keep the running balance consistent, and only the
   server can do that.
3. **The contract is a plan, not yet a live API.** The routes in the backend
   plan are not built. `wire/parse.ts` is written against that plan's
   examples. **When the routes land, read the live Swagger and one real
   response per route before trusting a single field name.** Four bugs on
   2026-09-22 came from coding against a spec rather than the artefact.

---

## 2. Module layout

```
src/features/ledgerStatement/
  LedgerStatementScreen.tsx      // layout + wiring; reads/writes the URL query (§3.3)
  components/
    FilterStrip.tsx              // ledger, from, to, branch, show, 5 checkboxes, Show F5
    LedgerPicker.tsx             // the dropdown component over /ledgers (§3.2)
    LedgerDetailPanel.tsx        // header.ledger — six facts
    PeriodSummaryPanel.tsx       // header.period — opening / Dr / Cr / closing
    MonthWisePanel.tsx           // /monthly — table + bars, click a month (§7.3)
    VoucherGrid.tsx              // virtualised; synthetic Opening/Total/Closing rows (§8)
    LegsPopover.tsx              // Alt+F1 on the focused row (§8.5)
    DailyTable.tsx               // Daily tab (§9)
    MonthlyTable.tsx             // Monthly tab (§9)
    TypeChip.tsx                 // rowKind + voucherTypeShort → chip (§8.4)
    ReportFooter.tsx             // hint bar + Close / Excel / PDF / WhatsApp / Print
  api/
    ledgerStatement.ts           // the seven GETs, one function each, AbortSignal-aware
  wire/                          // no React import, no fetch
    types.ts                     // response shapes, straight from the backend plan
    parse.ts                     // raw JSON → typed; tolerant of missing optionals
    money.ts                     // "176300.00" + "DR" → "1,76,300.00 Dr" (§4.2)
    dates.ts                     // ISO on the wire, dd-MM-yyyy on screen
    errors.ts                    // server code → operator sentence (§12)
  query/
    params.ts                    // Filters ⇄ URLSearchParams ⇄ request query (§3.3)
    useStatement.ts              // header + monthly + first vouchers page, one generation
    useVoucherPages.ts           // page cache keyed by (filters, page) (§8.2)
    useVoucherLegs.ts            // per-voucher cache for the ▾ / Alt+F1 legs
  nav/
    drillTarget.ts               // row.src → route + key (§10)
  export/
    toXlsx.ts                    // /export response → workbook (§11)
```

`wire/`, `query/params.ts`, `nav/` and `export/` import no React. The tests go
there (§14).

---

## 3. Scope, inputs and the URL

### 3.1 What comes from the session, and what is a filter

| Input | Source | Why |
|---|---|---|
| `companyId` | session, captured on mount | not a picker; the report is always one company |
| `accYear` | session, captured on mount | the year **is** the partition and the opening set; L5 refuses ranges that cross it |
| `branchId` | the Branch combo | `All branches` = omit the key |
| `ledgerId` | the Ledger picker | required |
| `fromDate` / `toDate` | two date edits | both must fall inside the fiscal year |

The fiscal year's begin and end dates come from the session (the same place
the other accounts screens get them). The date edits clamp to that range, so
an out-of-year date cannot be typed. The server still refuses one
(`RANGE_OUTSIDE_YEAR`), and §12 shows that refusal. The client check is a
convenience, not the rule.

**Defaults on first open:** from = the fiscal year begin, to = today (or the
year end if today is past it), branch = the session's branch. The ledger is
empty, and the screen shows only the filter strip until one is picked.

### 3.1a "All branches" does NOT mean what it means on Opening Balances

On the Opening Balances screen, "All branches" is the **company-level set**
(`op_branch_id IS NULL`), which is a separate set and not a merge. Here it
means **every branch combined**: the Σ of every opening set plus every
branch's legs (backend §4.2, decision L2). Same words, opposite meanings.
Label the option **"All branches (combined)"** and put the rule in its
tooltip. An operator who has just used the opening screen will otherwise read
the two numbers as a contradiction.

When one branch is picked and the server answers
`openingNote: "COMPANY_LEVEL_ONLY"`, show a one-line banner under the panels:
*"This ledger's opening is held at company level, so this branch starts at
0.00. Choose All branches (combined) to include it."*

### 3.2 The ledger picker

This is the house dropdown component (`plan-react-dropdown.md`: popup grid,
keyboard matrix, loading, empty and error states) with a **different data
source**: `GET /reports/ledger-statement/ledgers?companyId&search&limit=30`,
debounced at 200 ms, with the in-flight request aborted on the next keystroke.
Columns are Name · Group · a Bill-wise chip · a Shared chip (`isShared`
means `led_company_id IS NULL`, the same ledger in every company).

- **Ctrl+L** focuses the picker from anywhere on the screen.
- **PgUp / PgDn** go to the previous or next ledger **in the same group**. The
  client keeps the last `/ledgers?groupId=<current>` result (sorted by name,
  as the server returns it), moves one step, and runs the report at once. At
  either end it stays put and the hint bar says so. It does not wrap.

### 3.3 The filters live in the URL

`/reports/ledger-statement?ledger=<uuid>&from=2026-08-01&to=2026-09-25&branch=<uuid>&tab=vouchers&cancelled=1&refs=1`

`query/params.ts` converts in both directions. This gives three things the Qt
screens could not have:

- The browser Back button returns to the previous ledger or range. Drilling
  into a voucher and pressing Back lands on the same scroll position (§10).
- A link pasted to a colleague opens the same report.
- A refresh keeps the report.

`companyId` and `accYear` are **not** in the URL. They come from the session,
so a link opened under another company re-scopes rather than leaking.
Changing a filter does **not** fetch. **Show (F5)** writes the URL, and the
URL change fetches. There is one path, so the Back button and the Show button
cannot disagree. The exceptions that fetch at once are PgUp/PgDn (§3.2) and a
month click (§7.3). Both go through the same URL write.

---

## 4. The wire contract

### 4.1 Routes (backend §5)

| Function in `api/ledgerStatement.ts` | Route | Used by |
|---|---|---|
| `searchLedgers` | `GET …/ledgers` | picker, PgUp/PgDn |
| `getHeader` | `GET …/header` | Ledger detail + Period summary, Total + Closing rows |
| `getVouchers` | `GET …/vouchers` | the grid, one call per page |
| `getVoucherLegs` | `GET …/voucher-legs` | ▾ expand, Alt+F1 |
| `getDaily` | `GET …/daily` | Daily tab |
| `getMonthly` | `GET …/monthly` | Month-wise panel + Monthly tab |
| `getExport` | `GET …/export` | Excel (and Print later) |

All of these are `GET` with a bearer token, and every one accepts an
`AbortSignal`. There is no route outside that list. If a field seems to be
missing, the fix is on the server (§13), not a call to an old route.

### 4.2 Amounts are strings, and they stay strings

Every amount arrives as a **string with two decimals**, with a separate side
where a side applies (`{ "amount": "176300.00", "side": "DR" }`).
`wire/money.ts` formats **from the string**:

- split on `.`, group the integer part Indian-style (`1,76,300`), and keep the
  two decimals exactly as sent;
- append ` Dr` / ` Cr` from `side`, never from a minus sign. The screen never
  shows a negative balance;
- a zero balance is shown as `0.00` with no side.

**Never** `Number(amount)`, **never** `parseFloat`, and **never**
`Intl.NumberFormat` on a parsed float. The client has no reason to do
arithmetic (§1), so it has no reason to leave the string domain. That removes
the float and locale class of bug entirely: the `176300.00000001` leak
from sale-order, and `2400,50` parsing as `240050`. The one place a number is
needed is the month bar heights (§7.3). There `Number()` is fine, because a
bar a pixel off is not an accounting fact.

### 4.3 Dates

ISO `YYYY-MM-DD` on the wire and `dd-MM-yyyy` on screen, converted only in
`wire/dates.ts`. Do not build a `Date` from an ISO date string without a time:
the browser reads it as UTC midnight, and in IST that is still the right day,
but the habit breaks the first time someone formats with `toISOString()`.
Keep dates as strings end to end, like amounts.

### 4.4 Types (`wire/types.ts`)

Mirror the backend plan §5.2–5.7 exactly. The ones the screen leans on:

```ts
type Side = 'DR' | 'CR';
type Bal = { amount: string; side: Side | null };        // null only on 0.00
type RowKind = 'NORMAL' | 'CANCELLED' | 'REVERSAL';

type VoucherRow = {
  voucherId: string; accYear: string; branchId: string; branchName: string;
  date: string; voucherTypeId: number; voucherTypeName: string; voucherTypeShort: string;
  voucherNo: string; status: 'POSTED' | 'CANCELLED'; rowKind: RowKind;
  pairOutsideRange?: boolean;
  particulars: string | null; asPerDetails: boolean; legCount: number;
  narration: string | null; billRefs: string[];
  debit: string; credit: string; balance: Bal; createdBy: string | null;
  src: { module: string; docType: string; docId: string } | null;
};

type VoucherPage = {
  broughtForward: Bal; rows: VoucherRow[]; carriedForward: Bal;
  page: { page: number; pageSize: number; totalRows: number };
};
```

`parse.ts` treats every optional as optional, and it **fails loudly** (a
visible "the server's answer was not understood" state, with the route name)
on a missing required field. It never defaults a missing amount to `"0.00"`: a
silent zero in a ledger is worse than an error.

---

## 5. Loading: one generation at a time

Show fires **three** requests in parallel: `/header`, `/monthly` and
`/vouchers?page=1`. Tab switches add `/daily`. They belong to one
**generation**: a counter bumped on every URL change, captured in each
request, and checked before the result is applied. The previous generation's
requests are aborted.

Why this matters: an operator on PgDn who presses it four times fast must
never see ledger 3's panels over ledger 4's grid. Aborting alone is not enough,
because a response can already be in the parser when the abort fires. Aborting
**and** comparing the generation is. If the app uses TanStack Query, the query
key `['ledgerStatement', 'header', filters]` gives the same guarantee. Either
way, one rule: **nothing is rendered from a response whose filters are not the
current URL's.**

While a generation loads, the previous one stays on screen dimmed, with a thin
progress bar. It is not blanked. Blanking on every PgDn makes it impossible
to compare two ledgers by eye, and that is exactly what PgDn is for.

---

## 6. The filter strip

### 6.1 Fields

Ledger\* (§3.2) · From · To · Branch (§3.1a) · Show (`Detailed` only in phase
1, and the combo exists for the Tally-style `Condensed` later) · **Show F5**.

### 6.2 The five checkboxes: server parameters vs display switches

| Checkbox | Kind | Effect |
|---|---|---|
| Narration | display | shows/hides the narration text inside Ref / narration; no refetch |
| Bill refs | **server** `withBillRefs` | refetch; off saves the server a join |
| Contra legs (Alt+F1) | **server** `withLegs` (L4) | refetch; every "as per details" row arrives expanded |
| Include cancelled | **server** `includeCancelled` | refetch; the pair rule is the server's (§1 fact 2) |
| Running balance | display | shows/hides the Balance column; no refetch |

A server checkbox takes effect on the next Show, like any filter. It lives in
the URL. A display checkbox acts at once and is remembered per user in
`localStorage` (wrapped in try/catch, and it must work without it). If L4
lands as "a call per row" instead, Contra legs becomes a display switch that
triggers `useVoucherLegs` for each visible "as per details" row.

---

## 7. The three panels

### 7.1 Ledger detail (`header.ledger`)

Group · Nature · Maintain (bill by bill / balance only) · GSTIN · Credit ·
Mobile. **Credit shows `limit —  ·  30 days`.** No credit-limit column
exists anywhere (backend L6). Do not hide the fact. The dash says "not
recorded", and hiding it says "not relevant". `creditDays: null` shows `—`.

### 7.2 Period summary (`header.period`)

Opening at From · Debits (n vouchers) · Credits (n vouchers) · **Closing at
To**, the last one large and in the accent colour. These four figures are
the reference for everything else on the screen: the grid's Total and Closing
rows are drawn **from here** (§8.3), not from the grid.

### 7.3 Month-wise (`/monthly`)

This panel covers the whole fiscal year and ignores From/To. Months inside the
current range are tinted. Future months (`isFuture`) are listed greyed with
`—` rather than `0.00`, so "no movement" and "not yet" look different.

The bars show the closing of each month (the one place `Number()` is allowed,
§4.2). A Cr closing is drawn below the axis. A debtor that has gone into
credit should look different at a glance.

**Clicking a month** sets From/To to that month, clipped to the fiscal year
(the first month may start on `fy_books_begin_date`), and runs the report at
once. That is the Tally "month → vouchers" drill. This is the same data the
Monthly tab shows in full (§9).

---

## 8. The voucher grid

### 8.1 Columns

| # | Column | From | Notes |
|---|---|---|---|
| 0 | ▾ | `asPerDetails` | toggle; blank when the voucher has one contra |
| 1 | Date | `date` | dd-MM-yyyy |
| 2 | Type | `rowKind` + `voucherTypeShort` | chip (§8.4) |
| 3 | Voucher | `voucherNo` | |
| 4 | Particulars | `particulars` ?? "(as per details)" | |
| 5 | Ref / narration | `billRefs` + `narration` | `agst bil00355, bil00361 · Sept route`; narration part obeys the Narration checkbox |
| 6 | Branch | `branchName` | hidden when one branch is selected (it would be one value repeated) |
| 7 | Debit | `debit` | blank when `"0.00"` |
| 8 | Credit | `credit` | blank when `"0.00"` |
| 9 | Balance | `balance` | `1,76,300.00 Dr`; obeys the Running balance checkbox |
| 10 | By | `createdBy` | |

Debit and Credit **can both be filled on one row** (backend L3: a receipt with
a discount pair). Do not "fix" that by netting in the client. If L3 lands as
net, the server will send one side as `"0.00"` and the rendering rule above
already handles it.

### 8.2 Rows, pages and virtualisation

A year of a busy debtor is thousands of rows. Render with a virtual list
(TanStack Virtual, as the quotation grid does). Fetch `/vouchers` in pages of
200 as the viewport approaches the unloaded range. `totalRows` from page 1
sizes the scrollbar from the start, so the scrollbar never jumps.

**Each page is self-sufficient.** It carries its own `broughtForward`, and
every row carries its own `balance`. **Never compute a running balance by
adding a row's debit and credit to the previous row's balance**, not even to
"fill in" an unloaded gap. If page 5 is not loaded, its rows are
placeholders, and a jump to the end loads the last page directly. That is
what the server-side window function (backend §6) is for.

### 8.3 The synthetic rows

These are not in `rows[]`. The grid draws them:

| Row | Position | Figures from |
|---|---|---|
| **Opening balance**, `b/f from <From − 1>` | top | `header.period.opening` |
| **Total for the period** | after the last row | `header.period.debit.amount`, `header.period.credit.amount` |
| **Closing balance**, `c/f` | last | `header.period.closing` |

**The Total row is never Σ of the loaded rows.** Unloaded pages make that sum
wrong, and a wrong total at the foot of a ledger is exactly the kind of number
that gets reported as a posting bug. The Total and Closing rows only appear
once the last page is loaded. Until then the foot shows
*"… 1,240 more rows — End to jump"*, so the operator never sees a total that
could be misread as a partial one.

**Consistency check (dev builds only):** when the last page is loaded, assert
that its last `balance` equals `header.period.closing`. Also assert that page
1's `broughtForward` equals `header.period.opening`. On a mismatch, log both
with the filters. It is a server bug to report, and it must not be patched
over in the client.

### 8.4 Chips (`TypeChip.tsx`)

| rowKind | Chip text | Style |
|---|---|---|
| NORMAL | `voucherTypeShort` upper-cased (SALES, RECEIPT, CR NOTE, JOURNAL, CHQ BOUNCE …) | colour per voucher **type id**, from one small map with a neutral fallback |
| CANCELLED | `<TYPE> · CANCELLED` | red chip; the whole row in muted text; **not** struck through |
| REVERSAL | `REVERSAL` | red chip; muted row |

The row is not struck through because a struck-through figure reads as "does
not count", and it **does** count (§1 fact 2). Muted text says "look at this
together with its pair". Hovering either half highlights the other when both
are loaded (match by `voucherNo` in `narration`, or better by a pair id if
the server adds one, see §13).

`pairOutsideRange: true` adds a small ⇢ glyph with the tooltip *"The other
half of this cancel is outside the dates shown."*

The colour map is keyed by **voucher type id**, and unknown ids get neutral
grey. It is presentation only. **No behaviour may branch on voucher type**
anywhere in this feature. If one ever needs to, the server should send the
fact instead.

### 8.5 Legs: ▾ inline, Alt+F1 popover

- **▾** on an "as per details" row expands contra-leg sub-rows beneath it:
  `Dr Cash … 15,000.00`, indented and muted, with the amount in the Debit or
  Credit column of the leg's own side. The sub-rows come from `withLegs` when
  it was on, or else from `getVoucherLegs` fetched on first expand and cached
  by `voucherId` for the session.
- **Alt+F1** on the focused row opens `LegsPopover`, anchored to the row.
  Title: voucher no · date · type · status. It lists **every** leg, with this
  ledger's leg(s) (`isThisLedger`) in bold at the foot, like the mockup. It
  also lists role legs (`role`: TDS_RECEIVABLE, BANK_CHARGES …) with the role
  as a small caption. Esc closes it, and Alt+F1 on another row moves it.
- Expanded sub-rows are **excluded** from the virtual row count's paging maths.
  They are client-side children of a row, not rows of the report.

---

## 9. Daily and Monthly tabs

Both are small, unpaged tables over `/daily` and `/monthly`, with the same
Opening / Total / Closing synthetic rows fed from the same `header.period`
(Daily) or from `/monthly.opening` and the last month's closing (Monthly).

- **Daily:** Date · Vouchers · Debit · Credit · Closing. Enter on a day switches
  to the Vouchers tab with From = To = that day.
- **Monthly:** Month · Debit · Credit · Closing, all twelve months. Enter on a
  month does what a click on the panel does (§7.3).

The tab is in the URL (`tab=`). Only the active tab's route is fetched.
`/monthly` is always fetched because the panel needs it.

---

## 10. Drill-down: Enter opens the voucher

Enter (or a double-click) on a real row navigates to the source document's own
screen, using `row.src`:

```ts
// nav/drillTarget.ts
const ROUTES: Record<string, (k: DocKey) => string> = {
  'SALES/SALE_BILL':  k => `/sales/bills/${…}`,
  'ACCOUNTS/RECEIPT': k => `/accounts/receipts/${…}`,
  // one entry per screen that exists; nothing else
};
```

- **The key is the row's, never the session's.** `companyId` is the report's
  own. `branchId` and `accYear` come **from the row**. A voucher written under
  another branch or dated in another partition must open in its own scope.
  The cheques plan (§10 there) found Qt getting this wrong, and the fix is
  the same here.
- **Unknown `src`**, or `src: null` (a voucher with no source document, such
  as a future journal), shows a toast: *"This voucher has no screen to open
  yet."* It must never be a dead Enter.
- This lookup table is the **only** type-keyed map in the feature, and it maps
  to routes, not behaviour. The day a screen exists, it gets one line here.
- Back returns to the report with the same URL (§3.3) and the **same scroll
  position**. Store `{ generation filters, scrollTop, focusedVoucherId }` in
  history state before navigating, and restore it after the pages reload.

---

## 11. Export, print, WhatsApp

| Button | Phase 1 | How |
|---|---|---|
| **Excel** | yes | `getExport` → `toXlsx` (SheetJS). Sheet 1: the Period summary as a header block, then the grid **with** Opening / Total / Closing rows, amounts written as numbers with 2 dp and a Dr/Cr column (a spreadsheet user wants to sum). Sheet 2: month-wise. File name `Ledger_<ledger>_<from>_<to>.xlsx` |
| **Print / PDF** | no | waits for the printing pipeline (`plan-react-printing.md`); needs a print purpose `LEDGER_STATEMENT` seeded with `/export` as its dataset (§13). Until then the button is disabled with the tooltip *"Arrives with the printing module"* |
| **WhatsApp** | no | depends on Print (it sends the PDF) |

Excel is the **one** place an amount string becomes a number, and it happens
inside `toXlsx`, at the sheet boundary. The cells get
`Number(amount)` and a `#,##,##0.00` format. That is safe because the
workbook is the end of the line: nothing reads the number back into the app.

`RANGE_TOO_LARGE` (422) on export: *"This range has 24,310 vouchers, and the
export limit is 20,000. Narrow the dates."* The count comes from the error
body.

Do not build a one-off print path through `window.print()` of the grid. The
cheques plan refused the same shortcut for the deposit slip, for the same
reason: a second print path is a second layout to keep right.

---

## 12. Errors → sentences (`wire/errors.ts`)

| Server | Shown |
|---|---|
| `LEDGER_NOT_IN_COMPANY` | "This ledger belongs to another company." |
| `BRANCH_NOT_IN_COMPANY` | "That branch is not part of this company." |
| `YEAR_UNKNOWN` | "The year <accYear> is not set up for this company." |
| `RANGE_OUTSIDE_YEAR` | "Both dates must fall inside <begin> – <end>." |
| `RANGE_REVERSED` | "From is after To." |
| `RANGE_TOO_LARGE` | §11 |
| 403 | "You do not have access to the Ledger Statement." (the whole screen, not a toast) |
| network / 5xx | "The report could not be loaded." + Retry, with the previous result kept on screen dimmed |

The error shows in one line under the filter strip, and the field it names
gets the accent outline. There are no modal dialogs for a report error.

---

## 13. Backend gaps this screen inherits

1. **None of the seven routes exist yet.** Phase 1 runs against MSW fixtures
   built from the backend plan's examples (§14).
2. **Backend §3.1:** `fn_ledger_book_balance` reads every Dr opening as Cr.
   The report does not call it. But the reconcile guard does, so until it is
   fixed the report and a "reconcile refused" message on a bill will disagree,
   **and the report will be the one that is right.** Say so to testers.
3. **Backend §3.2:** the opening-balance closing and carry-forward drop
   cancelled originals. Until that is fixed, this screen's closing at year end
   will not equal next year's opening on any ledger that had a cancel. Test 12
   in the backend plan is that check.
4. **Decisions L1–L7 open.** L3 (gross vs net) and L4 (inline legs) change
   this plan's §8.1 and §6.2. Both are written to survive either answer.
5. **No pair id.** Matching a CANCELLED row to its REVERSAL for hover
   highlighting (§8.4) needs `pairVoucherId` on each row. The server already
   knows it (`avh_reversal_voucher_id` / `avh_against_voucher_id`). Ask for
   it; it is one column.
6. **No credit limit** anywhere (L6).
7. **Print purpose `LEDGER_STATEMENT`** is not seeded, and its dataset would be
   the `/export` shape.
8. **Menu row** (L1): the screen needs its own `menu_master` row under 137,
   and 144 "Ledger Monthly Summary" should open this screen with `tab=monthly`.

---

## 14. Tests

- **Vitest, `wire/money.ts`, written first.** `"0.00"` → `0.00`; `"5.00"`;
  `"100000.00"` → `1,00,000.00`; `"176300.00"` + DR → `1,76,300.00 Dr`;
  `"12345678.90"` + CR → `1,23,45,678.90 Cr`. The decimals must survive
  byte-for-byte, and the function must be proven not to call `Number`.
- **Vitest, `query/params.ts`:** round trip `parse(build(f)) ≡ f` for every
  filter combination. The URL never carries company or year.
- **Vitest, `nav/drillTarget.ts`:** the key uses the row's branch and year, and
  the session's are never used. An unknown `src` returns `null`, not a throw.
- **Vitest, `export/toXlsx.ts`:** the synthetic rows are present, amounts are
  numeric cells, and Dr/Cr is its own column.
- **MSW fixtures** from the backend plan's worked example (Sri Krishna
  Traders, 01-08 → 25-09): 14 rows incl. one cancelled pair, two "as per
  details" rows, opening 1,85,200.00 Dr, closing 1,76,300.00 Dr. The screen
  test asserts the Opening, Total and Closing rows equal `header.period` to
  the character.
- **Race test:** PgDn ×4 with the fixtures delayed in reverse order. The
  final screen shows ledger 5 in every panel.
- **Paging test:** 450 fixture rows. Scroll to the end without loading the
  middle. The last balance is correct, and no balance is computed client-side
  (spy on `money` helpers receiving only server strings).
- **Live, once the routes land:** backend §11 tests 1, 6, 7, 9 and 10 driven
  through the screen, not the API.

---

## 15. Phases

| Phase | Builds | Done when |
|---|---|---|
| 1 | `wire/`, `query/params.ts`, `nav/`, MSW fixtures | the Vitest suites above pass |
| 2 | Filter strip + picker + the three panels + the Vouchers grid (paging, chips, synthetic rows, ▾, Alt+F1) | the mockup reproduces from fixtures; race + paging tests pass |
| 3 | Daily + Monthly tabs, month click, drill-down + Back restore, Excel | Enter on each supported `src` opens the right screen in the row's scope |
| 4 | Switch fixtures → live routes | Swagger + one real response per route read; `parse.ts` corrected; live tests pass |
| later | Print / PDF / WhatsApp | printing module exists and `LEDGER_STATEMENT` is seeded |

---

## 16. Do not build

- **No money arithmetic.** No running balance, no total, no b/f, no
  "fill the gap" sum. If a figure is not in a response, it is a server ask
  (§13), not a client computation.
- **No old routes.** Not dropdown 39, not the grid runner, not
  `/account-ledger-masters/*`, not `/opening-balances/*`. The user's rule, and
  the reason the backend plan gives the picker its own route.
- **No client-side filtering of cancelled rows.** `includeCancelled` is a
  server parameter. Hiding one half locally breaks the running balance of
  every row between the halves.
- **No strike-through on cancelled rows**; they count (§8.4).
- **No behaviour keyed on voucher type.** Chips and the drill route map are
  presentation and navigation. Anything more should be a field from the
  server.
- **No session scope in the drill key** (§10).
- **No `Number()` / `parseFloat` / `Intl.NumberFormat` on amounts** outside
  `toXlsx` and the month bars.
- **No report data in `localStorage`.** Only the two display checkboxes live
  there. A cached ledger is a stale ledger.
- **No `window.print()` print path** (§11).
- **Do not reuse the Opening Balances screen's "All branches" semantics**
  (§3.1a).
