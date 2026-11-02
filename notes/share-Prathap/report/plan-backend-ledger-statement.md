# Ledger Statement — backend plan (NestJS)

**For Prathap.** Written 2026-09-25 against the schema at 192.168.0.106/ERP and
the NestJS tree as it stands today. Where the schema does not decide something
it is marked **DECISION** and listed in §12.

That box is a TEST database: its catalog is authoritative, its rows are not.
Row values quoted below only prove a *shape*.

Companion files in this folder (`Prathap/report/`):

    ledger_report_ui_mockup.png   the screen — Vouchers / Daily / Monthly tabs,
                                  filter strip, three panels, the voucher grid
    ledger_report_ui_mockup.py    regenerates it

**The whole module in one line:** a read-only report over `acc_vouchers` legs of
ONE ledger for ONE period, starting from that year's `acc_opening_balance`,
counting POSTED **and** CANCELLED vouchers, and served from its own new routes —
nothing it needs comes from an existing URL.

---

## 1. What this is, and why it is first

`accounts/accounting_scope.md` §wave 1 item 3: *"today there is no screen in the
product that can show whether a single posting is correct."* The ledger
statement is that screen. Every posting engine written so far (sale bill,
receipt, cheques, re-tender) is accepted on the strength of a SQL query someone
ran by hand. This replaces the query.

It writes nothing. No table, no column, no DB function, no trigger.

---

## 2. Routes are NEW and SELF-CONTAINED — the rule for this module

User, 2026-09-25: *"write into a separate urls dont use any old urls because
its gonna change bite later"*.

So:

- Every route lives under **`/api/v1/reports/ledger-statement/…`** (§5).
- The report does **not** call, forward to, or share a DTO with
  `/account-ledger-masters/*`, `/opening-balances/*`, `/receipts/*`, the
  dropdown runner or the grid runner. The ledger facts, the ledger picker and
  the month summary each get a route of their own here, even where an existing
  route returns something similar today.
- It does **not** call `accounts.fn_ledger_book_balance()` or
  `fn_party_bill_reconcile()` — §3.1 is why, and "no logic in the database"
  (notes 42) is the other reason. The SQL lives in the service (§6).
- It may import Prisma models and shared decorators (`@RequiredUuid`,
  `@UpperMaxString`, the auth guard). Those are building blocks, not URLs.

When the voucher/ledger routes change shape later, this module does not move.

---

## 3. Two defects found while reading — fix BEFORE the report is trusted

Neither is in this module. Both make the report disagree with numbers the
system already shows, and the report will be right.

### 3.1 `fn_ledger_book_balance` reads every Dr opening as Cr

```sql
SUM(CASE WHEN o.op_dr_cr = 'DR' THEN o.op_amount ELSE -o.op_amount END)
```

`op_dr_cr` is `char(1)` with `ck_op_dr_cr CHECK (op_dr_cr IN ('D','C'))`. It is
never `'DR'`, so **every** opening lands in the ELSE branch as a credit.
Verified: a ledger whose only row is `89.00 D` with no vouchers returns
`-89.00`.

`fn_party_bill_reconcile` → `fn_books_reconcile` → the `reconcile_on_post`
guard (notes 47) all sit on it. So every bill-by-bill party with a Dr opening
shows a false mismatch of 2 × its opening, and with
`accounts.reconcile_on_post = true` its next post is refused. This is very
likely notes 47's **D2** ("opening sign differs between acc_opening_balance
and the opening bill") — the sign differs in the *function*, not the data.

**Fix:** `WHEN o.op_dr_cr = 'D'`. One character.

### 3.2 Opening-balance closing + carry-forward drop cancelled originals

`opening-balance.service.ts closingByLedger()` (≈ line 429) and
`carry-forward.service.ts` (≈ line 317) both filter
`h.avh_voucher_status = 'POSTED'`.

But a cancel does not remove legs. Verified on five cancelled headers today:
the CANCELLED original keeps all its legs, and its mirror
(`avh_reversal_voucher_id` → a POSTED header whose
`avh_against_voucher_id` points back) carries the same number of legs, opposite
side. Counting POSTED only keeps the **mirror** and drops the **original**, so
each cancelled voucher moves the closing by its full amount **in the reverse
direction**.

Consequence: the year-end closing that `/opening-balances/list` shows beside
the opening is wrong by every cancel in the year, and **carry-forward writes
that wrong figure into next year's opening**, which is then permanent.

**Fix:** `h.avh_voucher_status IN ('POSTED','CANCELLED')` in both places — the
same rule `fn_ledger_book_balance` already uses correctly. The comment in
`carry-forward.service.ts` ("POSTED only because `ck_avh_balanced` only bites
at POSTED") no longer holds either: a CANCELLED header was POSTED, and
balanced, before it was cancelled.

The ledger statement's closing on the last day of a year must equal next
year's carried opening. Until 3.2 is fixed, it will not, on any ledger that
had a cancel.

---

## 4. The one definition of a balance

Every figure on the screen comes from this, and only this:

    balance(ledger, company, year, branchScope, asOf)
      =  opening(ledger, company, year, branchScope)            -- §4.2
      +  Σ av_signed_amount
           over acc_vouchers v JOIN acc_voucher_header h (pk)
           where v.av_company_id = company
             and v.av_ledger_id  = ledger
             and v.av_acc_year   = year
             and v.av_voucher_date <= asOf
             and v.av_is_deleted = false  and h.avh_is_deleted = false
             and h.avh_voucher_status IN ('POSTED','CANCELLED')
             and (branchScope = ALL or v.av_branch_id = branchScope)

- **DRAFT never counts** — a draft has no legs anyway (R10), but filter it
  regardless.
- **CANCELLED counts** — §3.2. Its POSTED mirror nets it to zero.
- **APPROVED** is legal in `ck_avh_status` and nothing writes it
  (approval withdrawn 2026-09-15). Treat it as not counting; if one ever
  appears, it is a bug elsewhere.
- Sign: `av_signed_amount` is GENERATED (+DR / −CR). Positive = **Dr**,
  negative = **Cr**, shown as `1,76,300.00 Dr` — never a minus sign on screen.
  `acc_opening_balance` has no signed twin: sign it in the query
  (`'D'` → +, `'C'` → −).

### 4.1 Opening at the From date

    openingAtFrom = opening(year set) + Σ legs with date in [year begin, From − 1]

Year begin = `public.fiscal_years.fy_begin_date` for (`comp_id`,
`fy_year_name = accYear`) — or `fy_books_begin_date` when that is later
(a company that started mid-year).

### 4.2 Which opening rows — the branch scope

`ux_op_scope` is `(company, COALESCE(branch, 0-uuid), year, ledger)`: the
company-level set (`op_branch_id IS NULL`) and each branch's set are separate
rows.

| Branch filter | Opening = | Legs = |
|---|---|---|
| one branch | that branch's row only | `av_branch_id = branch` |
| All branches | **Σ every set** (company-level + every branch) | every branch |

"All branches = Σ every set" is what `fn_ledger_book_balance` does today, so
the statement and the reconcile guard agree. The opening-balance README calls
the two sets "different sets of rows" — if a company-level row can ever
*duplicate* a branch row rather than *complement* it, this double-counts.
**DECISION L2.**

When one branch is picked and the ledger has a company-level opening only,
the branch view opens at 0 and the response says so
(`openingNote: "COMPANY_LEVEL_ONLY"`), so the screen can show it rather than
look wrong.

### 4.3 Company scope

`led_company_id` is nullable **on purpose** — NULL = shared by every company.
The ledger is valid for the request when `led_company_id IS NULL OR
led_company_id = companyId`. Balances always come from the company-scoped
tables (`av_company_id`, `op_company_id`); a shared ledger shows only this
company's movement.

---

## 5. The routes

Controller: `@Controller('reports/ledger-statement')`, `@Version(API_VERSION)`,
bearer auth, all `GET`, all read-only. **Seven routes**, nothing else.

**Common query keys** (one DTO, `LedgerStatementScopeDto`, reused by every
route below):

| key | type | rule |
|---|---|---|
| `companyId` | uuid | required |
| `accYear` | `YYYY-YYYY` | required, `@UpperMaxString(9)` + the second-half check |
| `branchId` | uuid | optional; absent = All branches |
| `ledgerId` | uuid | required (except on `/ledgers`) |
| `fromDate` | `YYYY-MM-DD` | required on `/header`, `/vouchers`, `/daily`, `/export` |
| `toDate` | `YYYY-MM-DD` | same; `from ≤ to`, both inside the fiscal year (§9) |

Amounts go out as **strings with 2 decimals** (the house Decimal rule — no
float64, see the sale-order float leak), sides as `"DR"` / `"CR"`.

### 5.1 `GET /reports/ledger-statement/ledgers` — the picker

The ledger combo on the filter strip. Search-as-you-type.

Query: `companyId`, `search` (optional, ≥ 1 char, matches `led_name`,
`led_alias`, `led_short`, `led_gstin_no`, `led_phone1`), `groupId` (optional),
`limit` (default 30, max 100).

```json
{ "items": [ { "ledgerId": "…", "name": "Sri Krishna Traders",
               "groupId": 12, "groupName": "Sundry Debtors",
               "isBillByBill": true, "isShared": true } ] }
```

Also serves PgUp / PgDn ("previous / next ledger in the group"): the client
asks with `groupId` and walks the list sorted by `led_name`.

### 5.2 `GET /reports/ledger-statement/header` — the three top panels

LEDGER DETAIL + PERIOD SUMMARY in one call.

```json
{
  "ledger": {
    "ledgerId": "…", "name": "Sri Krishna Traders",
    "groupName": "Sundry Debtors", "nature": "ASSET",
    "isBillByBill": true, "gstin": "33AAKFS4410M1ZQ",
    "mobile": "98430 21177", "creditDays": 30, "creditLimit": null
  },
  "period": {
    "fromDate": "2026-08-01", "toDate": "2026-09-25",
    "opening":  { "amount": "185200.00", "side": "DR" },
    "debit":    { "amount": "129000.00", "vouchers": 7 },
    "credit":   { "amount": "137900.00", "vouchers": 7 },
    "closing":  { "amount": "176300.00", "side": "DR" },
    "cancelledPairs": 1,
    "openingNote": null
  }
}
```

Sources: `acc_ledger_master` + `acc_group_master` (`acc_group_nature`);
`mobile` = `led_phone1`, falling back to `led_whatsapp_no`;
`creditDays` = `sales.customers.cus_credit_days` when the ledger is a customer
(`cus_id = led_id`), `purchase.suppliers.sup_credit_days` when a supplier,
else null.

**`creditLimit` has no column anywhere** (`accounts`, `sales`, `purchase` all
checked). It is returned as null and the mockup's "Credit limit" fact shows
"—". Adding one belongs to the customer master, not this report —
**DECISION L6**.

### 5.3 `GET /reports/ledger-statement/vouchers` — the grid

Extra query keys: `includeCancelled` (bool, default **true**),
`page` (≥ 1, default 1), `pageSize` (default 200, max 1000).

```json
{
  "broughtForward": { "amount": "185200.00", "side": "DR" },
  "rows": [
    { "voucherId": "…", "accYear": "2026-2027", "branchId": "…",
      "branchName": "Main Store",
      "date": "2026-09-16",
      "voucherTypeId": 12, "voucherTypeName": "Receipt", "voucherTypeShort": "Rct",
      "voucherNo": "rct00017",
      "status": "POSTED", "rowKind": "NORMAL",
      "particulars": null, "asPerDetails": true, "legCount": 5,
      "narration": "Sept collection — Trichy route",
      "billRefs": ["bil00412", "bil00501"],
      "debit": "0.00", "credit": "21200.00",
      "balance": { "amount": "176300.00", "side": "DR" },
      "createdBy": "ramesh",
      "src": { "module": "ACCOUNTS", "docType": "RECEIPT", "docId": "…" } }
  ],
  "carriedForward": { "amount": "176300.00", "side": "DR" },
  "page": { "page": 1, "pageSize": 200, "totalRows": 14 }
}
```

- **Ordering:** `av_voucher_date, avh_voucher_slno, avh_voucher_id` — slno is
  the per-company posting sequence and is set on every non-DRAFT header
  (`ck_avh_no`), so the order is total and stable across pages.
- **One row per voucher**, not per leg. `debit` = Σ this ledger's DR legs in it,
  `credit` = Σ its CR legs; both can be non-zero (a receipt with a discount pair
  has CR party + DR discount / CR party). Balance moves by the net.
  **DECISION L3** (gross both vs net one).
- **`broughtForward` / `carriedForward`** are computed server-side for the
  page, so the running balance is right on page 7 without the client summing
  pages 1–6 (§6).
- `rowKind`: `NORMAL`, `CANCELLED` (header status CANCELLED), `REVERSAL`
  (a POSTED header that some other header names in `avh_reversal_voucher_id`).
  The mockup's chips come from this, never from parsing a voucher number.
- `src` is the drill-down key: the client opens the source screen with
  `companyId · branchId · accYear · docId` (house rule 3). The report never
  proxies the source document.

**Opening, Total and Closing rows are NOT rows in `rows[]`.** The client draws
them from `broughtForward` (page 1) / `/header`. That keeps paging honest.

### 5.4 `GET /reports/ledger-statement/voucher-legs` — Alt+F1 / the ▾ toggle

Query: `companyId`, `accYear`, `voucherId`, `ledgerId` (so the response can
mark which legs are "this ledger").

```json
{ "voucherNo": "rct00017", "date": "2026-09-16", "status": "POSTED",
  "legs": [ { "rowNo": 1, "side": "DR", "ledgerId": "…", "ledgerName": "Cash",
              "amount": "5000.00", "role": null, "isThisLedger": false,
              "remarks": null } ] }
```

Every non-deleted leg of the voucher in `av_row_no` order. `role` = `av_role`
(TDS_RECEIVABLE, BANK_CHARGES, …) so the popover can label a role leg.
With the **Contra legs** checkbox on, the client calls this for every
"as per details" row on the page — or **DECISION L4** adds `withLegs=true` to
`/vouchers` to inline them.

### 5.5 `GET /reports/ledger-statement/daily` — the Daily tab

Same scope keys. One row per date **that has movement**, plus the opening:

```json
{ "opening": {…}, "days": [ { "date": "2026-09-02", "debit": "9800.00",
  "credit": "25000.00", "vouchers": 2, "closing": { "amount": "…", "side": "DR" } } ],
  "closing": {…} }
```

Not paged — a fiscal year is at most 366 rows.

### 5.6 `GET /reports/ledger-statement/monthly` — MONTH-WISE panel + Monthly tab

Scope keys **without** from/to: always the whole fiscal year, month 1 = the
month of `fy_begin_date`. Returns all twelve months, including months with no
movement (debit/credit 0, closing carried), and months after today
(`isFuture: true`) so the bar chart has a fixed axis.

```json
{ "opening": { "amount": "…", "side": "DR" },
  "months": [ { "month": "2026-04", "debit": "62300.00", "credit": "41000.00",
                "closing": { "amount": "192750.00", "side": "DR" },
                "isFuture": false } ] }
```

This is also what menu 144 "Ledger Monthly Summary" needs — **DECISION L1**.

### 5.7 `GET /reports/ledger-statement/export` — Print / PDF / Excel / WhatsApp

Same keys as `/vouchers` minus paging. Returns `/header` + all rows +
the same shape, in one response, hard-capped at **20,000 rows** (422
`RANGE_TOO_LARGE` beyond that, with the count, so the screen can say "narrow the
dates"). The client renders the print / file from it. No server-side PDF in
this plan.

---

## 6. The main query (for `/vouchers`)

Everything else is a simpler cut of it.

```sql
WITH scope AS (
  SELECT $company::uuid AS co, $ledger::uuid AS led, $year::bpchar AS yr,
         $branch::uuid AS br, $from::date AS d_from, $to::date AS d_to
),
op AS (                                   -- §4.2
  SELECT COALESCE(SUM(CASE o.op_dr_cr WHEN 'D' THEN o.op_amount
                                      ELSE -o.op_amount END), 0) AS amt
    FROM accounts.acc_opening_balance o, scope s
   WHERE o.op_company_id = s.co AND o.op_ledger_id = s.led
     AND o.op_acc_year = s.yr AND o.op_is_deleted = false
     AND (s.br IS NULL OR o.op_branch_id = s.br)
),
legs AS (
  SELECT v.av_voucher_id, v.av_acc_year, v.av_voucher_date, v.av_branch_id,
         v.av_dr_cr, v.av_amount, v.av_signed_amount
    FROM accounts.acc_vouchers v
    JOIN accounts.acc_voucher_header h
      ON h.avh_voucher_id = v.av_voucher_id AND h.avh_acc_year = v.av_acc_year
    , scope s
   WHERE v.av_company_id = s.co AND v.av_ledger_id = s.led
     AND v.av_acc_year = s.yr
     AND v.av_voucher_date <= s.d_to
     AND v.av_is_deleted = false AND h.avh_is_deleted = false
     AND h.avh_voucher_status IN ('POSTED','CANCELLED')
     AND (s.br IS NULL OR v.av_branch_id = s.br)
),
before AS (SELECT COALESCE(SUM(av_signed_amount),0) AS amt
             FROM legs, scope s WHERE av_voucher_date < s.d_from),
per_voucher AS (
  SELECT av_voucher_id, av_acc_year,
         SUM(av_amount) FILTER (WHERE av_dr_cr = 'DR') AS dr,
         SUM(av_amount) FILTER (WHERE av_dr_cr = 'CR') AS cr,
         SUM(av_signed_amount)                         AS net
    FROM legs, scope s WHERE av_voucher_date >= s.d_from
   GROUP BY 1, 2
)
SELECT h.*, pv.dr, pv.cr,
       (SELECT amt FROM op) + (SELECT amt FROM before)
       + SUM(pv.net) OVER (ORDER BY h.avh_voucher_date, h.avh_voucher_slno,
                                    h.avh_voucher_id
                           ROWS UNBOUNDED PRECEDING) AS running
  FROM per_voucher pv
  JOIN accounts.acc_voucher_header h
    ON h.avh_voucher_id = pv.av_voucher_id AND h.avh_acc_year = pv.av_acc_year
 ORDER BY h.avh_voucher_date, h.avh_voucher_slno, h.avh_voucher_id
 LIMIT $pageSize OFFSET ($page - 1) * $pageSize;
```

- The window runs over the **whole range** before `LIMIT`, so `running` on any
  page is exact. `broughtForward` = the first row's running − its net;
  `carriedForward` = the last row's running.
- `$queryRaw` with tagged-template parameters, exactly as
  `opening-balance.service.ts` does — never string-built SQL.
- **The cancelled-pair filter (§7) is applied in `per_voucher`, not after the
  window** — a hidden pair must also be out of the running balance's inputs, or
  the balances stop agreeing with the rows around them. (It nets to zero, so
  the final balance is the same either way; the rows between the two halves
  are what differ.)

### 6.1 Indexes — what exists, and the one gap

- `ix_av_ledger (av_company_id, av_ledger_id, av_voucher_date) INCLUDE
  (av_dr_cr, av_amount, av_signed_amount, av_voucher_no) WHERE NOT deleted` —
  fits `legs` exactly, per partition. Nothing to add for the main path.
- Header join is by `pk_acc_voucher_header (avh_voucher_id, avh_acc_year)`.
- `ix_avh_party` / `ix_avh_daybook` are `WHERE status = 'POSTED'` — they
  **do not** serve this report (it needs CANCELLED too). They are not needed:
  the report drives from legs, not headers. Do not "fix" them.
- `ix_op_lookup (company, year, ledger) INCLUDE (amount, dr_cr, stale)` serves
  `op`.
- **Gap:** `REVERSAL` detection needs "is some header's
  `avh_reversal_voucher_id` = this id". There is no index on
  `avh_reversal_voucher_id`. Either add
  `CREATE INDEX ix_avh_reversal ON accounts.acc_voucher_header
   (avh_reversal_voucher_id, avh_reversal_acc_year)
   WHERE avh_reversal_voucher_id IS NOT NULL;`
  or detect from the mirror's side (`avh_against_voucher_id` has
  `ix_avh_against`) — but `avh_against_voucher_id` is ALSO used by a PDC
  voucher pointing at its receipt, so from that side a mirror is
  "against_voucher_id set **and** that voucher is CANCELLED **and** its
  `avh_reversal_voucher_id` = me". Both work; the index is simpler.

---

## 7. Cancelled vouchers — the pair rule

- A cancel = the ORIGINAL header goes CANCELLED (legs kept) + a MIRROR header
  POSTED with the opposite legs, `original.avh_reversal_voucher_id = mirror`,
  `mirror.avh_against_voucher_id = original`. Verified live today.
- **Both count** in every total. Net effect zero.
- `includeCancelled=true` (default): both rows returned,
  `rowKind = CANCELLED` / `REVERSAL`.
- `includeCancelled=false`: **both halves hidden, never one** — only when BOTH
  fall inside [from, to]. If one half is outside the range (cancelled in the
  next month, say), the half inside is shown regardless, because it moves the
  balance in this period. The response counts these in
  `period.cancelledPairs` and flags the lone half with
  `pairOutsideRange: true`.
- Period summary counts (`debit.vouchers`, `credit.vouchers`) include both
  halves; the mockup's "14 vouchers · 1 cancelled pair" is exactly that.

Known cancel defect this report will *show*, not cause: notes 47 **D3** — the
bill-cancel mirror takes the next sale-bill number. The REVERSAL row's
`voucherNo` will be e.g. `bil00603`. That is the data, not a report bug.

---

## 8. Particulars and bill refs

- **Particulars** — the other side of this ledger in the voucher:
  1. if this ledger's legs carry `av_opp_ledger_id` and they all agree →
     that ledger's name;
  2. else the distinct ledgers on the **opposite side** of the voucher's legs;
     exactly one → its name;
  3. else `particulars: null, asPerDetails: true` → the screen shows
     "(as per details)" and offers Alt+F1.
  `legCount` = number of legs in the whole voucher.
- **Narration** = `avh_remarks`; if empty, `avh_usr_refno`; else null.
- **Bill refs** (`billRefs[]`, only when the Bill refs checkbox asks —
  `withBillRefs=true`, default true): for a voucher that SETTLES bills —
  `acc_bill_adjustment` rows with `abj_voucher_id = voucher`,
  `abj_voucher_acc_year = year`, `abj_party_id = ledger`, not deleted →
  `abl_voucher_refno` of each `abj_bill_id` (join on `abl_id` +
  `abj_bill_acc_year`, the bill can be in an older year). For a voucher that
  RAISES a bill — `acc_bill_balance` whose voucher is this one → its own
  refno + due date (`"due 10-09"` in the mockup).
  Negative counter-rows (a retracted allocation, `bill-adjustment reverses`)
  net out: list a bill only if its net `abj_amount` for this voucher ≠ 0.

---

## 9. Rules only the service can enforce

All refuse with 400 and a code the client can show:

| code | when |
|---|---|
| `LEDGER_NOT_IN_COMPANY` | `led_company_id` is neither NULL nor `companyId` |
| `BRANCH_NOT_IN_COMPANY` | `branch_master.br_comp_id ≠ companyId` |
| `YEAR_UNKNOWN` | no `fiscal_years` row for (company, accYear) |
| `RANGE_OUTSIDE_YEAR` | from/to not inside `fy_begin_date … fy_end_date` — **DECISION L5** |
| `RANGE_REVERSED` | `from > to` |
| `RANGE_TOO_LARGE` | `/export` over 20,000 rows (422) |

Menu rights: the same check every other menu-backed route makes for the
calling user on this report's menu id (§10). A user without the menu gets 403
on all seven routes — including `/ledgers`, because the picker lists every
ledger's name.

---

## 10. Menu and registration

- `fixed.menu_master` has **no** Ledger Statement row. Under 137
  "Financial Statements" (parent 6 "Reports", both `menu_visiblity = false`
  today) the siblings are 138 Trial Balance · 139 Balance Sheet · 140 P&L ·
  143 Account Group Summary · **144 Ledger Monthly Summary** · 154 P&L w/o
  Stock · 181 Day Book Detail.
- Needs: a new row "Ledger Statement" under 137, 137 and 6 made visible, and
  the rights rows for the testers. **Remember `user_menus` save is a FULL
  REPLACE** — granting this one by the API with a partial list revokes the
  user's other menus.
- No `ui_tables` / grid / dropdown registration: the screen reads only the
  routes in §5 (that is the point of §2).

---

## 11. Test checklist

Seed with a throwaway ledger (prefix `ZT-LS-`) in one company; clean up after.

1. Ledger with only an opening `D 1000` and no vouchers → `/header` opening
   and closing = `1000.00 DR`. (Catches §3.1 if anyone swaps in the DB
   function.)
2. Same with `C 1000` → `1000.00 CR`.
3. Post a sale bill → one row, `debit` = bill total, balance moves Dr.
4. Receipt against it → one row, particulars = the cash/bank ledger,
   `billRefs` = the bill.
5. Receipt with cash + UPI + TDS + discount → `asPerDetails: true`,
   `/voucher-legs` returns every leg, `isThisLedger` on the party leg.
6. Cancel the sale bill → two rows (CANCELLED + REVERSAL), closing unchanged
   from before the bill. `includeCancelled=false` → both hidden, closing
   still the same.
7. Cancel dated next month, range = this month only → the original shown
   alone with `pairOutsideRange: true`.
8. A DRAFT voucher on the ledger → never appears.
9. `branchId` = branch A → only A's legs and A's opening set; ledger with a
   company-level opening only → opening 0, `openingNote = COMPANY_LEVEL_ONLY`.
10. Paging: 450 rows, pageSize 200 → page 3's `broughtForward` = page 2's
    `carriedForward`; page 3's last `balance` = `/header` closing.
11. `/monthly` → 12 months, Σ debit − Σ credit + opening = the last closing;
    the month containing `toDate` agrees with `/daily` for that month.
12. `/header` closing at `fy_end_date` = next year's carried opening
    **after** §3.2 is fixed (fails today on any ledger with a cancel — that
    failure is the proof of §3.2).
13. Shared ledger (`led_company_id` NULL) used in two companies → each company
    sees only its own legs.
14. Every §9 code, and 403 without the menu.
15. All amounts arrive as strings with two decimals; no `1e+5`, no
    `176300.00000001`.

---

## 12. Decisions

| # | question | recommendation |
|---|---|---|
| **L1** | Menu: new "Ledger Statement" row under 137, and what happens to 144 "Ledger Monthly Summary"? | New row; 144 opens the same screen on the Monthly tab (one screen, two menu entries), no separate route |
| **L2** | All branches opening = Σ every `acc_opening_balance` set (company-level + branches)? | Yes — it matches `fn_ledger_book_balance` and the reconcile guard; confirm a company-level row never duplicates a branch row |
| **L3** | Per-voucher row shows gross Dr **and** gross Cr, or one net figure? | Gross both (the mockup); balance moves by the net |
| **L4** | Contra legs: a call per row (`/voucher-legs`) or inline `withLegs=true` on `/vouchers`? | Inline flag — one round trip per page |
| **L5** | A range crossing a year end (e.g. Mar–May) — refuse, or stitch two years? | Refuse (`RANGE_OUTSIDE_YEAR`); the year IS the partition and the opening set |
| **L6** | "Credit limit" on the ledger panel has no column anywhere | Show "—" now; a customer-master column is a separate ask |
| **L7** | Index `ix_avh_reversal` (§6.1) or detect mirrors via `avh_against_voucher_id`? | Add the index |

---

## 13. Do not build

- No write of any kind; no new table; no DB function (notes 42).
- No call to `fn_ledger_book_balance` / `fn_party_bill_reconcile` /
  `fn_books_reconcile` — §3.1.
- No reuse of any existing route or response DTO — §2.
- No `/list`-style document list: this is a report over legs, not a list of
  documents, and it is not registered as a grid (the running balance and the
  cross-page b/f cannot be expressed through the grid runner).
- No server-side PDF / Excel / WhatsApp sending — `/export` returns data only.
- No Day Book, Trial Balance or Cash/Bank Book here. They reuse §4's balance
  definition and should get their own `reports/*` modules, same pattern.
