# Account Day Summary (day book) — backend plan (NestJS)

**For Prathap.** Written 2026-10-12 against the schema at 192.168.0.106/ERP and
the NestJS tree (`nexerp-api-ro`) as it stands today. Where the schema does not
decide something it is marked **DECISION** and listed in §12.

That box is a TEST database: its catalog is authoritative, its rows are not.
Row values quoted below only prove a *shape*.

Companion files in this folder (`Prathap/report/`):

    account_day_summary_ui_mockup.png   the screen — Summary / Vouchers / Day-wise,
                                        filter strip, tiles, four panels
    account_day_summary_ui_mockup.py    regenerates it (worked example reconciles:
                                        Σ type amount = Σ group Dr = Σ group Cr)
    plan-react-account-day-summary.md   the client
    plan-backend-ledger-statement.md    the balance definition this one REUSES (§4)

**The whole module in one line:** a read-only report over the `acc_vouchers`
legs of ONE company, one branch (or all), ONE day (or a range inside one fiscal
year), classified by voucher type, by money ledger and by ledger group, counting
POSTED and CANCELLED headers, leaving out cancelled pairs that net to zero
inside the range, and served from its own new routes.

---

## 1. What this is

`accounts/accounting_scope.md` §8 lists **Day Book (summary + detail)** as a
MUST, and §10 wave 1 item 3 puts it next to the ledger statement: the two
screens that can *check* a posting. The ledger statement checks one ledger; the
day summary checks one day: did every voucher balance, where did the cash and
the bank go, what did each group move.

3.0 reference: `pos/Reports/daybook.*` (company + type + from/to, ◀ ▶ one-day
step, grid 649, Enter opens the voucher) and `daybookdetail.*`. The gestures
carry over, not the layout.

It writes nothing. No table, no column, no DB function, no trigger.

---

## 2. Routes are NEW and SELF-CONTAINED

Same rule as the ledger statement (user, 2026-09-25: *"write into a separate
urls dont use any old urls"*):

- Every route lives under **`/api/v1/reports/day-summary/…`** (§5).
- It does **not** call or forward to `/reports/ledger-statement/*`, even where
  the voucher-legs shape is identical. Copy the SQL fragments (`openingSql`,
  `legsSql`, `kindCtes`) into this module's service, or lift them into a shared
  **SQL helper** file under `src/modules/reports/common/` that both import —
  a helper is a building block, a route is not. **DECISION D1.**
- No grid runner, no dropdown runner, no `/vouchers/*`, no `/till/*` route.
  The voucher-type picker and the business-day status get routes of their own
  here.
- No DB function (notes 42). SQL lives in the service via `prisma.$queryRaw`
  tagged templates, as `ledger-statement.service.ts` does.

---

## 3. What was found while reading — read before trusting the figures

Verified on the box 2026-10-12 (3,610 vouchers with legs, year 2026-2027).

### 3.1 Clean: the books balance

Zero rows each for: unbalanced vouchers, header totals ≠ leg sums, leg
branch/date/type/number/company ≠ its header, single-leg vouchers, live legs
under deleted headers, drafts with legs, CANCELLED without a reversal. **The
Dr = Cr check (§8) passes today.** It is still worth having: it is the alarm for
the next posting engine.

### 3.2 `avh_doc_amount` is NOT the voucher amount

`avh_doc_amount ≠ avh_total_debit` on **1,106 of 2,624** POSTED vouchers (Bil
452, Rev 187, SRt 129, PurA 115, ARc 65, …). Doc amount is the source
document's figure (bill total before what the posting split off), not what was
posted. **This report's "Amount" = Σ DR legs of the voucher** (= `avh_total_debit`
when the header is POSTED). `ix_avh_daybook` INCLUDEs `avh_doc_amount` — do not
be tempted by it.

### 3.3 Cancel mirrors come in two shapes

A cancel = the original goes CANCELLED (legs kept) + a POSTED mirror with the
opposite legs, `original.avh_reversal_voucher_id = mirror`,
`mirror.avh_against_voucher_id = original`, same date, same totals.

- **Same-type mirror** — Bil, DCh, DCR, Pmt, Rct, SRt: a cancelled bill posts
  another **Bil**, which takes the next Bil number (notes 47 D3).
- **`Rev`-type mirror** (type 23, nature JOURNAL) — every other type.

So "POSTED Bil vouchers today" counts the mirrors unless the query knows which
POSTED rows are mirrors. Use the ledger-statement `kindCtes` rule:
**REVERSAL = some header names this id in `avh_reversal_voucher_id`**
(`ix_avh_reversal` exists). §6 builds on it.

### 3.4 There is no money-ledger role

`acc_ledger_role` has no CASH, BANK, CARD_CLEARING or UPI_CLEARING. The reliable
rule (§4.2) walks the group chain to `acc_ledger_profile` 'Cash' / 'Bank'. It
agrees with `led_ledger_type` CASH/BANK on every live ledger, but
`led_ledger_type` is NULL on 395 ledgers, so it is not the key.

### 3.5 Tender → ledger mapping is wrong for several tenders (for whoever owns tenders)

On the box: CARD, WALLET, BANK and TEMP_CR tenders post **straight to the bank
ledger** (Kvb Current A/c); LOYALTY posts to Cash In Hand; RRN and CREDIT post
to **"SALES FREIGHT COLLECTION"**; CHEQUE posts to "Cheques In Hannd". Only UPI
uses a clearing ledger (UPI Clearing → settles to Kvb). Card Settlement
Clearing and Wallet Clearing exist with 0 legs.

Effect on this report: the CARD/UPI clearing rows of the money grid will be
empty or tiny until the tender master is corrected, and "card in" shows as bank
in. The report is right; the data is not. Settlement (menu 278) assumes card
money sits in a clearing ledger until the slip is matched — so this is a real
defect, not a cosmetic one. **Listed for the tender owner; not this module's fix.**

### 3.6 Number gaps are everywhere

17 of 30 series have gaps (6,809 numbers missing: OPENING 3,158, Phy 2,798,
Pmt 281, Bil 277, Rct 87, Rev 38); 12 series do not start at 1; 41
soft-deleted CANCELLED headers still hold numbers. A naive "gap = error" check
would fire every day. §8.3 defines the check so it is useful: per series, *for
the day only*, and it tells a deleted number apart from a never-used one.

### 3.7 Smaller ones

- **12 legless Rct vouchers** (6 POSTED + 6 CANCELLED, zero amount,
  2026-09-29, rct00275–rct00333). Counted by header, invisible to legs. The
  report drives from legs (§6), so they are not counted as vouchers — but the
  `LEGLESS` check (§8) lists them so they do not hide.
- **24 non-draft vouchers dated after today** (up to 2026-11-08). The date
  edits allow the future (a range to year end is legitimate); the `FUTURE_DATED`
  check counts them for the company when the asked range includes today.
- **`acc_group_primary_name` is NULL on 23 groups** (Customers, Suppliers,
  every area group). Group roll-up uses a recursive CTE on
  `acc_group_parent_id` (§4.3), never that column.
- **INVENTORY types carry legs:** OPENING (408 legs) and Phy (301) post
  INVENTORY / OPENING_DIFFERENCE / STOCK_EXCESS / STOCK_SHORTAGE. They are
  vouchers and they appear — that is correct for a day book.
- DB triggers (`tr_av_refresh_totals`, `fn_acc_group_inherit_nature`) exist
  despite the no-DB-logic rule. FYI; nothing here depends on them.

---

## 4. Definitions — every figure comes from these

### 4.1 Which legs count

Exactly the ledger statement's set (its plan §4), widened from one ledger to
all:

    legs(company, year, branchScope, from, to)
      = acc_vouchers v JOIN acc_voucher_header h
          ON (h.avh_voucher_id, h.avh_acc_year) = (v.av_voucher_id, v.av_acc_year)
        where v.av_company_id = company and v.av_acc_year = year
          and v.av_voucher_date between from and to
          and not v.av_is_deleted and not h.avh_is_deleted
          and h.avh_voucher_status IN ('POSTED','CANCELLED')
          and (branchScope = ALL or v.av_branch_id = branchScope)

- **DRAFT** has no legs (verified: 0 of 31) — counted from headers (§6.3),
  never summed.
- **APPROVED** is legal in the CHECK and absent from data — not counted.
- Optional `voucherTypeId` filter → `v.av_voucher_type_id = type` (the leg
  carries it; verified equal to the header's).

**The pair rule (the one place this differs from the ledger statement).**
A cancelled pair whose **both halves** are inside [from, to] (and inside the
branch scope) is **left out of every figure** — counts, amounts, cash/bank in
and out, group Dr/Cr — and reported only as `cancelledPairs`. Reason: the pair
nets to zero per ledger, so leaving it out changes no balance and no closing,
but leaving it IN doubles the day's gross sales and gross cash in/out with
movement that never happened. A **lone half** (cancelled tomorrow, or a
yesterday's bill cancelled today) moves this range's balances and is counted,
on its own type row, flagged (`loneHalves`). **DECISION D2.**

### 4.2 Money ledgers — cash, bank, clearing

    kind(ledger) =
      CASH      if the ledger's group chain reaches acc_ledger_profile = 'Cash'
      BANK      if it reaches 'Bank'       (Bank Accounts, Bank OD A/c)
      CLEARING  else if the ledger is acc_tender_master.tnd_ledger_id of a live
                tender with tnd_settlement_ledger_id IS NOT NULL
      (none)    otherwise

- Chain = recursive CTE on `acc_group_parent_id` from `led_group_id`, first
  non-General profile wins.
- Live today: CASH = Cash In Hand, Store Safe Cash, Cashin Hand, Cash in
  Transit; BANK = Kvb Current A/c, Test Bank Account; CLEARING = UPI Clearing,
  Card Settlement Clearing, Wallet Clearing.
- The `Clearing ledgers as bank` checkbox (`clearingAsBank=true`, default)
  folds CLEARING into the type grid's Bank in / Bank out columns. The money
  grid always shows clearing rows separately, tagged `kind`.
- **A money ledger appears in the money grid if it has an opening ≠ 0 at
  `from` OR any leg in range** (`zeroRows=true` adds every money ledger of the
  company).
- Ledger scope: `led_company_id IS NULL OR led_company_id = companyId` (NULL =
  shared, on purpose); balances always from company-scoped tables.

### 4.3 Group level for the group summary

Root groups on the box are the Tally primaries: Current Assets, Current
Liabilities, Sales Accounts, Indirect Expenses, … Cash-in-Hand, Bank Accounts
and Sundry Debtors are **children of Current Assets**. Rolling to the root
would put cash, bank and debtors on one line — useless for a day.

    reportGroup(ledger) = the group on the ledger's chain whose parent is a ROOT,
                          or the root itself when the ledger sits directly in a root

That gives exactly the mockup's rows (Cash-in-Hand, Bank Accounts, Sundry
Debtors, Sundry Creditors, Duties & Taxes, Sales Accounts, Indirect Expenses).
`groupLevel=PRIMARY` (root) and `groupLevel=LEDGER_GROUP` (the ledger's own
group) are the other two settings. Nature: `acc_group_nature` of the report
group; NULL (the junk roots "Corporate Accounts" / "Office accounts") → `null`,
shown as "—". **DECISION D3.**

### 4.4 Opening and closing of a money ledger

Copied from the ledger statement, unchanged:

    opening(ledger, at from) = Σ acc_opening_balance (D +, C −) for (company, year, ledger,
                               branch rule) + Σ av_signed_amount of counted legs dated
                               [year begin, from − 1]
    closing = opening + Σ in − Σ out            (in = DR legs, out = CR legs)

- Branch rule = ledger statement §4.2: one branch → that branch's row; All
  branches → Σ every set. `openingNote: "COMPANY_LEVEL_ONLY"` when a branch is
  picked and only a company-level row exists.
- Pairs left out by §4.1 do not change closing (they net to 0). Pairs **before**
  `from` are inside the opening sum both halves, which also nets to 0.

### 4.5 Amount of a voucher

`amount = Σ av_amount where av_dr_cr = 'DR'` over its counted legs (§3.2). On a
type row: Σ over its vouchers. Totals: Σ type amount = Σ group Dr = Σ group Cr
— the service asserts all three equal before it answers (§8.1).

Money goes out as **strings with 2 decimals** (`money()` / `sided()` in the
ledger-statement service), sides as `"DR"` / `"CR"`, never a minus sign.

---

## 5. The routes

Controller: `@Controller('reports/day-summary')`, `@Version(API_VERSION)`,
`@CacheTTL(0)`, bearer auth, all `GET`, `{success, message, data}`.
**Seven routes**, nothing else.

**Common query DTO** `DaySummaryScopeDto`:

| key | type | rule |
|---|---|---|
| `companyId` | uuid | required |
| `accYear` | `YYYY-YYYY` | required, `@UpperMaxString(9)` |
| `branchId` | uuid | optional; absent = All branches |
| `fromDate` | `YYYY-MM-DD` | required |
| `toDate` | `YYYY-MM-DD` | optional; absent = `fromDate` (one day) |
| `voucherTypeId` | int | optional; absent = all types |
| `clearingAsBank` | bool | default `true` (§4.2) |
| `groupLevel` | `REPORT` \| `PRIMARY` \| `LEDGER_GROUP` | default `REPORT` (§4.3) |
| `zeroRows` | bool | default `false` |

### 5.1 `GET /reports/day-summary/voucher-types` — the Type combo

`[{ voucherTypeId, code, short, name, nature, category, sortOrder }]` from
`acc_voucher_types` where `vchr_is_active`, in `vchr_sort_order`. Includes the
INVENTORY types that post legs (OPENING, Phy, StkAdj). Omits types with no legs
ever possible (Quo, SOr) — **D4**.

### 5.2 `GET /reports/day-summary/summary` — the whole Summary tab, one snapshot

One call, one transaction (`REPEATABLE READ`, read only), so the tiles, the
type grid, the money grid, the group grid and the checks are the same moment's
numbers and tie to each other. Four separate routes (as the mockup's panel
captions first suggested) could straddle a posting and disagree by one bill.

```jsonc
{
  "scope": { "companyId", "companyName", "branchId": null, "branchName": "All branches",
             "accYear": "2026-2027", "fromDate": "2026-10-09", "toDate": "2026-10-09",
             "days": 1, "asOf": "2026-10-12T11:40:03+05:30" },
  "businessDay": { "status": "OPEN" | "CLOSING" | "CLOSED" | "NONE" | "MIXED",
                   "zNo": null, "sessions": { "total": 3, "closed": 2 } },   // §7; null for a range
  "tiles": [ { "code": "SALES", "amount": "186420.00", "vouchers": 41, "drafts": 0,
               "cancelledPairs": 1 }, … , { "code": "CASH_IN_HAND",
               "opening": {"amount":"48250.00","side":"DR"}, "closing": {…}, "net": {…} } ],
  "types": [ { "voucherTypeId": 3, "short": "Bil", "name": "Sale Bill", "nature": "SALES",
               "category": "BOTH", "vouchers": 41, "drafts": 0, "cancelledPairs": 1,
               "loneHalves": 0, "amount": "186420.00",
               "cashIn": "92300.00", "cashOut": "0.00", "bankIn": "71620.00", "bankOut": "0.00" } ],
  "typeTotals": { "vouchers": 63, "drafts": 2, "cancelledPairs": 1, "amount": "495270.00",
                  "cashIn", "cashOut", "bankIn", "bankOut" },
  "moneyLedgers": [ { "ledgerId", "ledgerName": "Cash in Hand", "kind": "CASH",
                      "opening": {…}, "in": "110300.00", "out": "88700.00", "closing": {…},
                      "openingNote": null } ],
  "moneyTotals": { "CASH": {…}, "BANK": {…}, "CLEARING": {…}, "ALL": {…} },
  "groups": [ { "groupId", "groupName": "Sundry Debtors", "nature": "Assets",
                "debit": "22500.00", "credit": "61750.00", "net": {"amount":"39250.00","side":"CR"},
                "hasChildren": true } ],
  "groupTotals": { "debit": "495270.00", "credit": "495270.00" },
  "checks": [ { "code": "BALANCED", "level": "OK", "facts": { "debit": "495270.00", "credit": "495270.00" } }, … ]
}
```

- `types` rows: only types with a counted voucher or a draft in range
  (`zeroRows` adds the rest). Order `vchr_sort_order`.
- `cashIn/cashOut/bankIn/bankOut` per type = Σ DR / Σ CR of that type's legs on
  CASH, and on BANK (+ CLEARING when `clearingAsBank`), §4.2. A contra between
  two money ledgers appears on both sides — that is the movement.
- Tiles are **server-defined** (codes, not client sums of rows), §6.4.
- `checks` carry codes and facts, never sentences (§8); the client words them.

### 5.3 `GET /reports/day-summary/vouchers` — the Vouchers tab (day book detail)

Extra keys: `page` (1), `pageSize` (≤ 500, default 200), `withLegs` (false),
`includeCancelledPairs` (default **true** here — the detail shows what the
summary left out), `natureFilter` (optional, from a tile/row drill),
`moneyKind` (optional `CASH|BANK|CLEARING` — "only vouchers touching cash").

Row: `{ voucherId, accYear, date, voucherTypeId, short, name, nature, voucherNo
(avh_voucher_refno), branchName, party (avh_party_id → led_name) | null,
particulars | null, asPerDetails, legCount, narration (avh_remarks), amount,
status, rowKind: NORMAL | CANCELLED | REVERSAL, pairInRange: bool, createdBy,
src: {module, docType, docId} | null, legs?: [{ ledgerId, ledgerName, side,
amount, role }] }`, ordered `avh_voucher_date, avh_voucher_slno,
avh_voucher_id`. Particulars = the ledger statement's §8 rule, with "this
ledger" = the party, else the first CR leg on a receipt-nature, first DR leg
otherwise. Page block `{ page, pageSize, totalRows }` + `totals { amount }` for
the whole filtered set.

Drafts are **not** rows here (no number, no legs); `draftsOnly=true` returns
the DRAFT headers instead (`avh_voucher_date` in range, not deleted): `{
voucherId, date, type, amount: avh_doc_amount, createdBy, src }` — the only
place `avh_doc_amount` is used, because a draft has nothing else.

### 5.4 `GET /reports/day-summary/daily` — the Day-wise tab

One row per calendar day in [from, to] (days with nothing → zero rows only when
`zeroRows`): `{ date, vouchers, drafts, cancelledPairs, amount, cashIn, cashOut,
cashClosing: {…}, bankIn, bankOut, bankClosing: {…}, businessDay: status | null }`.
Closing = running from the range opening (one window, like ledger statement
§6). Max 366 rows, no paging.

### 5.5 `GET /reports/day-summary/group-ledgers` — Enter on a group row

Extra key `groupId` (required). The ledgers rolled into that report group:
`[{ ledgerId, ledgerName, groupName, debit, credit, net }]`, same legs, same
pair rule. Enter on a ledger here opens the **Ledger Statement** for that
ledger and range (client navigation, not a call — §2).

### 5.6 `GET /reports/day-summary/checks/detail` — "show me" on a check

Extra key `code`. Returns the rows behind one check: the legless vouchers, the
gap numbers, the future-dated vouchers, the drafts. Kept off `/summary` so the
summary stays small. Capped at 500 rows.

### 5.7 `GET /reports/day-summary/export` — Print / PDF / Excel / WhatsApp

`/summary` + every page of `/vouchers` (with legs) in one body. Data only, no
rendering (ledger statement §5.7). Over 20,000 voucher rows → 422
`RANGE_TOO_LARGE` with the count.

---

## 6. The queries

### 6.1 The core CTE (shared by summary, vouchers, daily, group-ledgers)

```sql
WITH s AS (SELECT $co::uuid co, $yr::bpchar yr, $br::uuid br,
                  $from::date d_from, $to::date d_to, $vt::int vt),
hdr AS (                             -- counted headers in range
  SELECT h.avh_voucher_id id, h.avh_acc_year y, h.avh_voucher_type_id vt,
         h.avh_voucher_status st, h.avh_voucher_date dt, h.avh_branch_id br,
         h.avh_reversal_voucher_id rev_id, h.avh_reversal_acc_year rev_y,
         h.avh_against_voucher_id ag_id,  h.avh_against_acc_year ag_y
    FROM accounts.acc_voucher_header h, s
   WHERE h.avh_company_id = s.co AND h.avh_acc_year = s.yr
     AND h.avh_voucher_date BETWEEN s.d_from AND s.d_to
     AND NOT h.avh_is_deleted AND h.avh_voucher_status IN ('POSTED','CANCELLED')
     AND (s.br IS NULL OR h.avh_branch_id = s.br)
     AND (s.vt IS NULL OR h.avh_voucher_type_id = s.vt)
),
kind AS (
  SELECT x.*,
         CASE WHEN x.st = 'CANCELLED' THEN 'CANCELLED'
              WHEN EXISTS (SELECT 1 FROM accounts.acc_voucher_header o
                            WHERE o.avh_reversal_voucher_id = x.id
                              AND o.avh_reversal_acc_year  = x.y) THEN 'REVERSAL'
              ELSE 'NORMAL' END AS row_kind,
         -- pair fully in range: the other half is in hdr too
         CASE WHEN x.st = 'CANCELLED'
                   THEN EXISTS (SELECT 1 FROM hdr m WHERE m.id = x.rev_id AND m.y = x.rev_y)
              WHEN x.ag_id IS NOT NULL
                   THEN EXISTS (SELECT 1 FROM hdr o WHERE o.id = x.ag_id AND o.y = x.ag_y
                                  AND o.st = 'CANCELLED' AND o.rev_id = x.id)
              ELSE false END AS pair_in_range
    FROM hdr x
),
legs AS (                            -- counted legs: pairs-in-range dropped (§4.1)
  SELECT v.*, k.row_kind
    FROM kind k
    JOIN accounts.acc_vouchers v
      ON v.av_voucher_id = k.id AND v.av_acc_year = k.y AND NOT v.av_is_deleted
   WHERE NOT k.pair_in_range
)
```

Notes:
- `ag_id` alone is not proof of a mirror — PDC vouchers use
  `avh_against_voucher_id` to point at their receipt. The `o.st = 'CANCELLED'
  AND o.rev_id = x.id` test is what makes it a mirror (ledger statement §6.1).
- With a `voucherTypeId` filter, a `Rev`-type mirror of a filtered Bil is not
  in `hdr`, so the Bil's pair is "lone" and counted. That is wrong for a filter.
  **Fix: compute `kind` without the type filter and apply `vt` in `legs`.**
  The SQL above shows the filter in `hdr` for readability only; implement it in
  `legs`.
- Branch: header and leg branch are equal on every live voucher (§3.1), so
  filtering the header is enough; filter the leg too, defensively.

### 6.2 Classification joins

```sql
money AS (                           -- §4.2, once per request
  WITH RECURSIVE chain AS (
    SELECT l.led_id, g.acc_group_id gid, g.acc_group_parent_id pid, g.acc_ledger_profile prof, 0 d
      FROM accounts.acc_ledger_master l JOIN accounts.acc_group_master g ON g.acc_group_id = l.led_group_id
     WHERE (l.led_company_id IS NULL OR l.led_company_id = $co)
    UNION ALL
    SELECT c.led_id, g.acc_group_id, g.acc_group_parent_id, g.acc_ledger_profile, c.d + 1
      FROM chain c JOIN accounts.acc_group_master g ON g.acc_group_id = c.pid)
  SELECT led_id,
         (array_agg(CASE prof WHEN 'Cash' THEN 'CASH' WHEN 'Bank' THEN 'BANK' END ORDER BY d)
            FILTER (WHERE prof IN ('Cash','Bank')))[1] AS kind
    FROM chain GROUP BY led_id
)
-- CLEARING: led_id IN (SELECT tnd_ledger_id FROM accounts.acc_tender_master
--                       WHERE tnd_settlement_ledger_id IS NOT NULL AND NOT deleted)
--           and kind IS NULL
```

The group roll-up (§4.3) is the same recursive walk, keeping the node whose
parent's parent is NULL. Both walks are small (51 groups) — compute them per
request; no cache, no table.

### 6.3 Drafts

```sql
SELECT avh_voucher_type_id, count(*)
  FROM accounts.acc_voucher_header
 WHERE avh_company_id = $co AND avh_acc_year = $yr AND NOT avh_is_deleted
   AND avh_voucher_status = 'DRAFT' AND avh_voucher_date BETWEEN $from AND $to
   AND ($br IS NULL OR avh_branch_id = $br)
 GROUP BY 1;
```

`ix_avh_pending` (company, branch, status, date) WHERE DRAFT/APPROVED serves it.

### 6.4 Tiles

| code | = |
|---|---|
| `SALES` | types with `vchr_nature = 'SALES'` |
| `SALE_RETURNS` | `vchr_nature = 'CREDIT_NOTE'` (SRt, CrN) |
| `RECEIPTS` | `vchr_nature = 'RECEIPT'` (Rct, RcpV, ARc, TPIn) |
| `PAYMENTS` | `vchr_nature = 'PAYMENT'` **except** ExpV |
| `EXPENSES` | `vchr_type_code = 'ExpV'` |
| `CASH_IN_HAND` | Σ CASH money ledgers: opening, net, closing |

Amount, vouchers, drafts, cancelledPairs per tile. **DECISION D5** (whether
Purchase gets a tile; whether TPOut/TPIn — till paid-out/in — belong under
Payments/Receipts or under a Till tile).

### 6.5 Indexes

- `ix_av_daybook (av_company_id, av_branch_id, av_voucher_date) WHERE NOT deleted`
  drives legs for one branch. For All branches the planner skips the branch
  column; with ~2 branches per company that is fine. **Measure before adding**
  `(av_company_id, av_voucher_date)`.
- The `hdr` CTE has no header index covering CANCELLED (`ix_avh_daybook` is
  POSTED-only — do not "fix" it, other readers rely on its predicate). Either
  drive `hdr` from `legs → header pk` (preferred: legs are indexed, headers are
  joined by pk), or add
  `ix_avh_day (avh_company_id, avh_voucher_date) WHERE NOT avh_is_deleted AND
  avh_voucher_status IN ('POSTED','CANCELLED')`. **DECISION D6.**
- `ix_avh_reversal` serves `kind`; `ix_av_ledger` serves the opening sums.

---

## 7. Business day status

Only for **one branch and one day**: `accounts.till_business_day` where
(`tbd_company_id`, `tbd_branch_id`, `tbd_business_date` = from, `tbd_acc_year`),
not deleted → `tbd_status`, `tbd_z_no`; sessions = `till_session` by
`ix_tss_day (tss_day_id, tss_acc_year)` → total and `tss_status = 'CLOSED'`
count. No row → `NONE` (a branch that does not run tills). All branches →
`MIXED` when the branches differ, else the common status. A range → `null`
(the Day-wise tab carries it per day).

Do not join `av_session_id` to `till_session` — it is not proven to be a till
session id (118 distinct values on legs).

---

## 8. Checks

Each `{ code, level: OK | WARN | INFO | FAIL, facts }`. The client words them.

### 8.1 Integrity (FAIL when broken)

| code | rule |
|---|---|
| `BALANCED` | Σ DR = Σ CR over counted legs; and no voucher with Σ signed ≠ 0 (facts list the first 20 ids) |
| `AMOUNT_TIES` | Σ type amount = Σ group debit = Σ group credit; and money closing = opening + in − out per ledger. A mismatch is a bug in **this** service — it also logs an error |
| `LEGLESS` | POSTED/CANCELLED headers in range with no live legs (§3.7) — WARN, not FAIL |

### 8.2 Housekeeping (WARN / INFO)

| code | rule |
|---|---|
| `DRAFTS` | drafts in range: count + Σ `avh_doc_amount` (§5.3) — WARN |
| `CANCELLED_PAIRS` | pairs left out (§4.1) + lone halves — INFO |
| `CLEARING_PENDING` | CLEARING ledgers with closing ≠ 0 — WARN (settles T+1, menu 278) |
| `BUSINESS_DAY` | §7 status ≠ CLOSED — WARN; NONE → INFO |
| `FUTURE_DATED` | non-draft vouchers of the company dated > today, when to ≥ today — INFO |

### 8.3 `NUMBER_GAPS` (INFO) — scoped so it is not noise

Per series (company, branch, type, year — `ux_avh_voucher_no`), for numbers
**issued in the range only**:

    lo = min(avh_voucher_no) in range, hi = max in range
    missing = { n in [lo, hi] : no live non-draft header holds n }
    of which DELETED = held by a soft-deleted header (41 such today)
             UNUSED  = held by nobody (allocator skipped / failed transaction)

Facts: `[{ series: "Bil · Main Store", lo, hi, deleted: [..], unused: [..] }]`,
first 50 numbers per series. No check across day boundaries (a gap between
yesterday's last and today's first is yesterday's question, and with 6,809
historical gaps it would fire forever). **DECISION D7** — whether this check
exists at all before the allocator's skips are explained.

---

## 9. Rules only the service can enforce

400 via `throwAccountsBadRequest`, error family `DAY_SUMMARY_ERROR`:

| code | when |
|---|---|
| `BRANCH_NOT_IN_COMPANY` | `branch_master.br_comp_id ≠ companyId` |
| `YEAR_UNKNOWN` | no `fiscal_years` row for (company, accYear) |
| `RANGE_OUTSIDE_YEAR` | from/to not inside `fy_begin_date … fy_end_date` (books-begin when later) |
| `RANGE_REVERSED` | `from > to` |
| `TYPE_UNKNOWN` | `voucherTypeId` not in `acc_voucher_types` |
| `GROUP_UNKNOWN` | `/group-ledgers` with a group not visible to the company |
| `CHECK_UNKNOWN` | `/checks/detail` with an unknown code |
| `RANGE_TOO_LARGE` | `/export` > 20,000 rows (422) |
| `NO_MENU_RIGHT` | 403 — §10 |

Reuse `resolveScope()` (fiscal year + branch) by copying, per §2 / D1.

---

## 10. Menu and registration

- New row **284 "Account Day Summary"** under **137 Financial Statements**,
  `menu_position` 0.6 (after 258 Ledger Statement), visible, verbs
  `{VIEW,PRINT,EXPORT}` (`menu_master_menu_id_seq` last value = 283 on
  2026-10-12 — re-check before inserting).
- **181 "Day Book Detail"** (exists, hidden, 2 users hold view): opens the same
  screen on the Vouchers tab. The right check accepts either:
  `user_menus.um_menu_id IN (284, 181)`, `um_can_view`, not deleted — the
  ledger statement's `assertMenuRight()` pattern with (258, 144). **D8.**
- Remember **`user_menus` save is a FULL REPLACE** when granting testers.
- Module `src/modules/reports/day-summary/` (controller, module, service, dto,
  types, README), registered in `app.module.ts` next to `LedgerStatementModule`
  and in `swaggerDocs.ts`.
- No `ui_tables`, grid or dropdown registration.

---

## 11. Test checklist

Throwaway data prefix `ZT-DS-` in one company + branch, one day; clean up after.

1. Empty day → every list empty, all totals `0.00`, `BALANCED` OK, money grid
   lists only ledgers with an opening ≠ 0.
2. One cash bill → SALES 1 voucher; cash in = its cash tender; Sales Accounts
   Cr + Duties & Taxes Cr = amount; Σ group Dr = Σ group Cr = amount.
3. Cancel that bill the same day → `vouchers` 0, `cancelledPairs` 1, amounts 0,
   cash closing = opening. `/vouchers` lists both rows (CANCELLED + REVERSAL,
   `pairInRange` true); `includeCancelledPairs=false` hides both.
4. Bill on D, cancel on D+1: day D counts it; day D+1 counts the mirror on
   its own row (Bil mirror → Bil row; a Jrl → Rev row) as a lone half; range
   [D, D+1] → pair left out.
5. Receipt in bank (UPI via clearing) → CLEARING in; `clearingAsBank=false`
   moves it out of the type grid's bank columns, money grid unchanged;
   `CLEARING_PENDING` WARN.
6. Contra cash → bank: cash out = bank in = amount on the CONTRA row; Σ type
   amount still = Σ group Dr.
7. Draft expense → `drafts` 1 on ExpV, amounts unchanged, `DRAFTS` WARN;
   `/vouchers?draftsOnly=true` lists it.
8. Branch A vs All branches: All = Σ branches for every figure; money opening
   follows ledger statement §4.2 (COMPANY_LEVEL_ONLY note case).
9. `voucherTypeId` = Bil with a Rev-mirror cancel of a Jrl in range → the Bil
   row is right, the Jrl pair does not leak in (§6.1 note).
10. Range of 3 days: `/daily` closings chain (day n opening = day n−1 closing),
    and the last day's closing = `/summary` closing for the range.
11. Every money closing here = Ledger Statement closing for the same ledger,
    range end, branch (run both).
12. Rights: user without 284 and 181 → 403 on all seven routes; with 181 only
    → 200.
13. Number gaps: delete (soft) a draft-turned-posted voucher in range → shows
    under DELETED, not UNUSED.

---

## 12. Decisions

| # | question | recommendation |
|---|---|---|
| **D1** | Copy the ledger-statement SQL helpers or lift them into `reports/common/`? | Lift into a shared SQL helper file (one balance definition, two readers); no route sharing |
| **D2** | Cancelled pair fully inside the range: left out of the figures (counted only), or summed gross? | Left out — gross doubles the day's sales and cash movement. The Vouchers tab still lists both halves |
| **D3** | Group summary level default | `REPORT` = the root's child (§4.3); PRIMARY and LEDGER_GROUP selectable |
| **D4** | Type combo: hide Quo / SOr (no legs ever)? | Hide types with `vchr_category = 'INVENTORY'` and no posting role — today Quo, SOr; keep OPENING / Phy / StkAdj |
| **D5** | Tile set: Purchase tile? till paid-out/in under Payments/Receipts? | No Purchase tile until purchase entry exists; TPOut/TPIn stay under their natures |
| **D6** | Header index for POSTED+CANCELLED by date, or drive from legs? | Drive from legs (no new index); add `ix_avh_day` only if measured slow |
| **D7** | Number-gap check: ship it, or wait for the allocator's skips to be explained? | Ship as INFO, in-range only, DELETED vs UNUSED split |
| **D8** | 181 Day Book Detail: same screen on Vouchers tab, or retire 181? | Same screen, Vouchers tab; one right check `IN (284,181)` |
| **D9** | §3.5 tender ledgers (card/wallet straight to bank, RRN/CREDIT to a sales freight ledger) | Correct the tender master on the box; it is a data fix, but Settlement depends on it |

---

## 13. Do not build

- No write; no table; no DB function or trigger (notes 42).
- No call to any `/reports/ledger-statement/*` route, `/vouchers/*`, `/till/*`,
  grid or dropdown runner — §2.
- No use of `avh_doc_amount` as an amount except for drafts (§3.2).
- No sentence text in `checks` — codes and facts only.
- No server-side PDF / Excel / WhatsApp — `/export` returns data.
- No Trial Balance, Cash Book or Bank Book here. Cash Book = this report's
  money grid + `/vouchers?moneyKind=CASH`; if a separate menu is wanted later,
  it is a menu row on this screen, not a module.
