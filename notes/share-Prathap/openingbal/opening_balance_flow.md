# Opening balances — the complete flow, table by table, column by column

Every table, column, default, generated column and CHECK below was read off the
live database at 192.168.0.106/ERP on 2026-09-12. Nothing here is assumed; where
something is undecided it is marked **OPEN DECISION** rather than guessed.

**On the row counts quoted throughout:** that box is a TEST/DEV database. Its
SCHEMA is authoritative — columns, constraints, generated expressions, indexes,
partitions — and every rule in this document rests on that. Its DATA is not: row
counts and populated values appear here only to show the SHAPE of a thing (how
many ledgers a screen must list, what a mapping has to cover). Re-measure before
running anything against real books.

Everything about opening balances now lives in this folder — see `README.md`.
**Corrected 2026-09-14** in two places in §4.3 (`abl_branch_id` NOT NULL;
`abl_src_*` set, not NULL) — `schema_gaps_2026-09-14.md` has the reasons and
the rest of the second-pass findings.
Diagram of the same thing: `opening_balance_tables.png`. Screen:
`opening_balance_ui_mockup.png` — one page, with the bill-wise breakup as a
detail panel rather than a modal (merged 2026-09-12).

---

## 0. The tables involved, and what each one is for

| table | partitioned | role in this flow |
|---|---|---|
| `public.fiscal_years` | no | defines the year, its books-begin date, and whether it has been carried forward |
| `accounts.acc_ledger_master` | no | the ledger; carries `led_is_bill_by_bill` **and a second, older opening-balance home** (§1) |
| `accounts.acc_group_master` | no | `acc_group_nature` decides whether a ledger has an opening at all |
| `accounts.acc_opening_balance` | **yes**, LIST on `op_acc_year` | the per-year opening, one row per ledger |
| `accounts.acc_bill_balance` | **yes**, LIST on `abl_acc_year` | the bill-wise opening, one row per outstanding bill |
| `accounts.acc_voucher_header` / `acc_vouchers` | yes | **NOT written by this flow** (§6) |
| `accounts.acc_voucher_types` | no | type 1 `OPENING` is INVENTORY / STOCK_JOURNAL and `vchr_affects_accounts = false` (§5) |

---

## 1. OPEN DECISION — there are TWO homes for an opening balance today

This has to be settled before anything is built, because both exist and they
disagree.

**Home A — on the ledger master (the Tally shape).** Three columns, all live:

    accounts.acc_ledger_master
      led_ob_amount   numeric   NOT NULL  DEFAULT 0
      led_ob_type     varchar   NOT NULL  DEFAULT 'DR'
      led_ob_as_on    date      NULL

One figure per ledger, no year. This is exactly how Tally stores it — the
opening balance is a *property of the ledger*, entered at the bottom of the
ledger master screen. **It is the one with data**: 2 of 44 ledgers have a
non-zero `led_ob_amount` (both 'opening stock', 12 and 11), and 1 has an
`led_ob_as_on` of 2026-02-26.

**Home B — `accounts.acc_opening_balance` (the per-year shape).** One row per
(company, branch, year, ledger), with provenance and staleness. **0 rows.**

They cannot both be authoritative. A ledger's opening differs *per year*, so
Home A cannot express 2025-2026 and 2026-2027 at once; Home B can. My reading is
that **Home B is correct and Home A is a legacy carry-over from the Tally import
shape**, and that `led_ob_*` should either be dropped or explicitly redefined as
"the migration seed only". But that is Prathap's call, and until it is made,
neither a screen nor a report knows which number to trust.

Everything below assumes **Home B is authoritative**.

---

## 2. `public.fiscal_years` — the year is a record, not a string

    fy_id                   uuid    PK
    comp_id                 uuid    NOT NULL        -- per company
    fy_year_name            char(9) NOT NULL        -- '2026-2027'
    fy_begin_date           date    NOT NULL        -- 2026-04-01
    fy_end_date             date    NOT NULL        -- 2027-03-31
    fy_books_begin_date     date                    -- Tally's "books beginning from"
    fy_status               varchar NOT NULL  DEFAULT 'OPEN'    -- OPEN | CLOSED
    fy_is_current           bool    NOT NULL  DEFAULT false
    fy_lock_date            date
    fy_closed_at            timestamptz
    fy_closed_by            uuid
    fy_is_carried_forward   bool    NOT NULL  DEFAULT false     -- ← the carry-forward flag
    fy_carried_forward_at   timestamptz
    fy_prev_fy_id           uuid                                -- ← the chain
    fy_detached             bool    NOT NULL  DEFAULT false     -- Tally's "split company data"
    fy_remarks              varchar(500)
    tally_guid / tally_master_id / tally_alter_id

**The carry-forward operation is already a first-class concept here.** The
screen's "Carry forward from 2025-2026" button is not inventing a workflow; it
is the thing `fy_is_carried_forward` / `fy_carried_forward_at` / `fy_prev_fy_id`
were put there to record.

Live: Acme Foods has 2026-2027 OPEN and current, 2025-2026 CLOSED with
`fy_lock_date` 2026-03-31 — and `fy_is_carried_forward = false` on every row,
i.e. **no year has ever been carried forward.**

---

## 3. `accounts.acc_opening_balance` — one row per ledger per year

    op_id            uuid       NOT NULL  DEFAULT uuidv7()
    op_company_id    uuid       NOT NULL
    op_branch_id     uuid       NULL                 -- NULL = company level
    op_acc_year      char(9)    NOT NULL             -- PARTITION KEY
    op_ledger_id     uuid       NOT NULL
    op_amount        numeric(18,2) NOT NULL DEFAULT 0
    op_dr_cr         char(1)    NOT NULL             -- 'D' | 'C'
    op_source        varchar(20) NOT NULL DEFAULT 'CARRY_FORWARD'
    op_is_stale      bool       NOT NULL DEFAULT false
    op_stale_since   timestamptz
    op_generated_at  timestamptz
    op_generated_by  varchar(50)
    op_remarks       text
    op_is_deleted    bool       NOT NULL DEFAULT false
    op_created_at / op_created_by / op_modified_at / op_modified_by

Constraints that shape the flow:

    ck_op_amount    op_amount >= 0                     -- NEVER a negative
    ck_op_dr_cr     op_dr_cr IN ('D','C')
    ck_op_source    op_source IN ('CARRY_FORWARD','MANUAL','MIGRATION')
    ck_op_acc_year  '^[0-9]{4}-[0-9]{4}$' AND right = left + 1
    ux_op_scope     UNIQUE (op_company_id,
                            COALESCE(op_branch_id,'000…0'::uuid),
                            op_acc_year, op_ledger_id) WHERE NOT op_is_deleted

`op_generated_at` / `op_generated_by` are **separate from** `op_created_*`: the
generator and the operator are different actors, and a row can be regenerated
without being re-created.

---

## 4. The three flows

### 4.1 FLOW A — first year on the system (`op_source = 'MIGRATION'`)

There is no prior year in the database; the figures come off the accountant's
closing trial balance.

**Step 1 — decide which ledgers qualify.** Read, do not write:

    SELECT l.led_id, l.led_name, l.led_is_bill_by_bill, g.acc_group_nature
      FROM accounts.acc_ledger_master l
      JOIN accounts.acc_group_master  g ON g.acc_group_id = l.led_group_id
     WHERE NOT l.led_is_deleted
       AND g.acc_group_nature IN ('Assets','Liabilities')

Live this returns **29 of 44** ledgers (17 Assets, 12 Liabilities). Excluded:
7 Income + 3 Expenses (they close to the P&L and open at zero) and **5 ledgers
whose group has a NULL nature** — those 5 are a data problem the screen must
surface, not silently drop.

**Step 2 — write one row per ledger with a balance.**

    INSERT accounts.acc_opening_balance
      op_company_id   = <company>
      op_branch_id    = NULL            -- company level unless branch openings are kept
      op_acc_year     = '2026-2027'
      op_ledger_id    = <ledger>
      op_amount       = 124500.00       -- ALWAYS POSITIVE
      op_dr_cr        = 'D'             -- the side lives here
      op_source       = 'MIGRATION'
      op_generated_at = now()           -- NULL if typed rather than imported
      op_generated_by = <user>

A ledger with a zero balance gets **no row at all** — absence is the zero, and
`ux_op_scope` then stays meaningful.

**Step 3 — bill-wise parties additionally get §4.3.**

**Step 4 — the set must balance.**

    SELECT sum(op_amount) FILTER (WHERE op_dr_cr='D') AS dr,
           sum(op_amount) FILTER (WHERE op_dr_cr='C') AS cr
      FROM accounts.acc_opening_balance
     WHERE op_company_id=<c> AND op_acc_year='2026-2027' AND NOT op_is_deleted

`dr - cr` is the difference. It is carried on a suspense ledger
('Difference in Opening Balances') as an ordinary row with
`op_source = 'MANUAL'` — visible, never folded into a total.

### 4.2 FLOW B — year-end carry forward (`op_source = 'CARRY_FORWARD'`)

**Step 1 — the prior year's closing balance per ledger.** Closing = its opening
plus every posted voucher line in that year:

    SELECT v.av_ledger_id,
           sum(v.av_signed_amount) AS movement
           -- av_signed_amount is GENERATED ALWAYS:
           --   CASE WHEN av_dr_cr = 'DR' THEN av_amount ELSE -av_amount END
      FROM accounts.acc_vouchers v
      JOIN accounts.acc_voucher_header h
        ON h.avh_voucher_id = v.av_voucher_id AND h.avh_acc_year = v.av_acc_year
     WHERE v.av_company_id = <c>
       AND v.av_acc_year   = '2025-2026'
       AND NOT v.av_is_deleted
       AND h.avh_voucher_status = 'POSTED'
       -- ck_avh_status allows DRAFT | APPROVED | POSTED | CANCELLED, and
       -- ck_avh_balanced only requires debit = credit at POSTED. So POSTED is
       -- the only status a closing balance may be built from.
     GROUP BY 1

    closing_signed = COALESCE(prior_opening_signed,0) + COALESCE(movement,0)
    where prior_opening_signed = op_amount * CASE op_dr_cr WHEN 'D' THEN 1 ELSE -1 END

**Step 2 — split the sign back out.** This is the step that gets written wrong:

    op_amount = abs(closing_signed)
    op_dr_cr  = CASE WHEN closing_signed >= 0 THEN 'D' ELSE 'C' END

Never store the signed figure. `ck_op_amount` refuses it anyway.

**Step 3 — only balance-sheet ledgers carry.** Income and expense net off into
retained earnings / P&L; whether that lands on a 'Profit & Loss A/c' ledger as a
CARRY_FORWARD row is an **OPEN DECISION** — Tally keeps it in a reserved P&L
ledger. Nothing in the schema decides it.

**Step 4 — a MANUAL row is not overwritten.** An upsert on `ux_op_scope` must
skip, or explicitly ask about, rows where `op_source = 'MANUAL'`, or the
regenerate silently undoes the accountant's correction.

**Step 5 — stamp the year:**

    UPDATE public.fiscal_years
       SET fy_is_carried_forward = true,
           fy_carried_forward_at = now(),
           fy_prev_fy_id         = <the 2025-2026 fy_id>
     WHERE fy_id = <the 2026-2027 fy_id>

### 4.3 FLOW C — bill-wise opening for a party

Applies when `acc_ledger_master.led_is_bill_by_bill = true` (live: **4 of 44**).
One row of `acc_bill_balance` per outstanding invoice — this is the same shape
Tally's "Bill-wise Breakup of: <ledger>" sub-screen produces.

    INSERT accounts.acc_bill_balance
      abl_company_id     = <company>
      abl_branch_id      = <branch>         -- NOT NULL (corrected 2026-09-14, DECISION 9):
                                            --   a bill is always branch-scoped, unlike op_branch_id
      abl_acc_year       = '2026-2027'      -- PARTITION KEY: the year being OPENED
      abl_party_id       = <led_id of the party>
      abl_bill_type      = 'OPENING'        -- ← the whole distinction
      abl_voucher_id     = NULL             -- FORCED, see below
      abl_voucher_type_id= NULL             -- FORCED, see below
      abl_doc_refno      = 'SB/2025/0412'   -- NOT NULL: the ORIGINAL invoice no
      abl_doc_date       = 2026-01-12       -- the ORIGINAL invoice date, BEFORE books begin
      abl_credit_days    = 30               -- NOT NULL DEFAULT 0
      abl_due_date       = 2026-02-11       -- NULL allowed; >= abl_doc_date (ck_abl_due_date)
      abl_dr_cr          = 'DR'             -- 'DR' | 'CR'  (note: TWO chars here,
                                            --  unlike op_dr_cr's single 'D'/'C')
      abl_bill_amount    = 48600.00         -- > 0 (ck_abl_amount) — never zero, never negative
      abl_alloc_amount   = 0                -- DEFAULT 0
      abl_disc_amount    = 0
      abl_writeoff_amount= 0
      abl_narration      = <optional>
      abl_src_module     = 'ACCOUNTS'       -- SET, not NULL (corrected 2026-09-14):
      abl_src_doc_type   = 'OPENING_BALANCE' --   links the bill to its acc_opening_balance row,
      abl_src_doc_id     = <op_id>          --   which is what the tie below is keyed on
      abl_src_acc_year   = <acc_year>
      abl_is_active      = true             -- DEFAULT true

**Do NOT send these two — they are GENERATED ALWAYS:**

    abl_pending_amount = abl_bill_amount - abl_alloc_amount
                                         - abl_disc_amount - abl_writeoff_amount
    abl_status         = CASE WHEN pending <= 0 THEN 'CLOSED'
                              WHEN alloc+disc+writeoff > 0 THEN 'PARTIAL'
                              ELSE 'OPEN' END

**The constraint that makes an opening bill voucher-less is already in place:**

    ck_abl_voucher CHECK (
        (abl_bill_type =  'OPENING' AND abl_voucher_id IS NULL
                                    AND abl_voucher_type_id IS NULL)
     OR (abl_bill_type <> 'OPENING' AND abl_voucher_id IS NOT NULL
                                    AND abl_voucher_type_id IS NOT NULL))

So the schema has *already decided* that an opening bill has no voucher behind
it — exactly Tally's model, where an opening reference is not a voucher either.

Two more CHECKs to respect: `ck_abl_src_doc` (src_doc_id and src_doc_type are
both NULL or both set — for an opening, both NULL) and `ck_abl_settled`
(alloc + disc + writeoff <= bill_amount).

**The tie:** for each bill-wise party,

    sum(abl_bill_amount * CASE abl_dr_cr WHEN 'DR' THEN 1 ELSE -1 END)
      = op_amount * CASE op_dr_cr WHEN 'D' THEN 1 ELSE -1 END

The screen must not let the breakup close while these differ. Nothing in the
database enforces it — **OPEN DECISION**: a trigger, or the service, has to.

---

## 5. The stock tie — and why it does not work today

The 'Stock in Hand' ledger's opening should equal the opening stock valuation.
It cannot be derived automatically at present:

    accounts.acc_voucher_types, row 1:
        vchr_type_code        = 'OPENING'
        vchr_type_name        = 'Opening Stock'
        vchr_category         = 'INVENTORY'
        vchr_nature           = 'STOCK_JOURNAL'
        vchr_affects_inventory= true
        vchr_affects_accounts = FALSE      ← posts nothing to the ledgers

So the opening stock entry (menu 44) moves inventory and writes **no accounting
row**. The Stock in Hand opening therefore has to be typed, or computed from the
stock valuation and written as an `acc_opening_balance` row by whatever performs
the carry-forward.

**OPEN DECISION:** the mockup shows a `FROM STOCK` source chip for this row, but
`ck_op_source` allows only CARRY_FORWARD / MANUAL / MIGRATION. Either add a
fourth value, or store it as MANUAL and hold the tie elsewhere.

(Tally's equivalent: with *Integrate Accounts and Inventory = Yes*, the Stock in
Hand ledger's opening is taken from the stock items and is **not editable**.)

---

## 6. Dr/Cr, and the `acc_vouchers` question

### 6.1 Three tables carry a direction flag, and all three disagree on spelling

| table | column | type | allowed | amount rule |
|---|---|---|---|---|
| `acc_opening_balance` | `op_dr_cr` | `char(1)` | `'D'` \| `'C'` | `ck_op_amount`: `op_amount >= 0` |
| `acc_bill_balance` | `abl_dr_cr` | `char(2)` | `'DR'` \| `'CR'` | `ck_abl_amount`: `abl_bill_amount > 0` |
| `acc_vouchers` | `av_dr_cr` | `char(2)` | `'DR'` \| `'CR'` (`ck_av_dr_cr`) | `ck_av_amount`: `av_amount > 0` |

The **rule** is identical everywhere and is the one thing not to break: the
amount is always positive, and the direction lives in the flag. The
**spelling** is not identical — `op_dr_cr` is one character and the other two
are two. Do not "tidy" that in a DTO; each is already constrained, and a client
that sends `'D'` to `acc_bill_balance` is rejected.

`acc_vouchers` also has `av_signed_amount`, GENERATED:

    CASE WHEN av_dr_cr = 'DR' THEN av_amount ELSE -av_amount END

`acc_opening_balance` has **no** generated signed column. Whoever sums openings
must apply the sign themselves:

    op_amount * CASE op_dr_cr WHEN 'D' THEN 1 ELSE -1 END

### 6.2 An opening balance does NOT post to `acc_vouchers` — and that is decided

Not by preference: `ck_abl_voucher` forces it. An `abl_bill_type = 'OPENING'`
row must have `abl_voucher_id` **and** `abl_voucher_type_id` NULL. If openings
were posted as a voucher, a party's opening bills would naturally point at it —
and that constraint refuses them. The schema has already chosen.

It is also Tally's model: an opening balance is a property of the ledger, and an
opening bill reference is not a voucher.

### 6.3 Could it be posted as a voucher? Technically yes, now

Worth stating because it was previously assumed impossible, and the live schema
says otherwise:

- `av_opp_ledger_id` **IS NULLABLE** — so a leg with no single opposite is
  legal. `ck_av_self` only bites when it is set (`opp IS NULL OR opp <> ledger`).
- `ck_avh_balanced` only requires `avh_total_debit = avh_total_credit` **at
  POSTED**, and a balanced opening set satisfies that by definition. A
  half-entered one can sit at DRAFT.

So one "Opening Journal" voucher per company-year, one line per ledger, would
fit the voucher tables. What is missing and what it would cost:

- **There is no voucher type for it.** Only 2 of the 6 deployed types affect
  accounts (Sales Bill, Order Advance Receipt); there is no Journal type at all,
  even though menu 103 'Journal' exists.
- **`ck_abl_voucher` would have to be altered**, which is the constraint that
  currently documents the decision.
- **Carry-forward becomes a cancel-and-repost** rather than an upsert, and
  `op_is_stale` / `op_source` / `op_generated_by` have no home on a voucher row.
- The opening stops being distinguishable from a real transaction in a ledger
  view unless every query filters on the voucher type.

### 6.3b The one argument FOR posting — tested, and it does not hold

The serious case for an opening voucher is not tidiness, it is **settlement**:
if allocations key off voucher LINES, then an opening bill with no voucher line
could never be receipted against, and posting would be forced.

Checked, and it is false. `accounts.acc_bill_adjustment` allocates against the
**bill**, not the voucher:

    abj_bill_id          NOT NULL   FK -> acc_bill_balance(abl_id, abl_acc_year)
    abj_against_bill_id  NULL       FK -> acc_bill_balance(abl_id, abl_acc_year)
    abj_voucher_id       NULL       (ck_abj_voucher_pair: null together with its year)
    abj_voucher_line_id  NULL
    abl_voucher_id       NULL       (and FORCED null for OPENING by ck_abl_voucher)
    abl_voucher_line_id  NULL

`abj_voucher_id` is the RECEIPT that pays a bill, not a property of the bill
being paid. So an opening bill settles exactly like any other bill: the receipt
voucher supplies its own id, and the bill it settles needs none of its own.

**Nothing in the settlement machinery requires an opening to be a voucher.**

**Recommendation: do not post it.** Keep `acc_opening_balance` authoritative.
The only thing genuinely lost is a ledger statement that shows "Opening Balance"
as its first line — and that is a query concern, answered by the view in §6.4,
not a reason to fabricate a voucher.

One place where a voucher IS a legitimate alternative: the **P&L close**
(DECISION 2). Transferring the year's net result to retained earnings as a
closing journal voucher is a real design, unlike an opening journal. Tally does
not do it either, but it is at least arguable there.

### 6.4 The cost of not posting it, and how to pay it once

Every balance is then:

    balance = opening (acc_opening_balance, this year, this company/branch)
            + sum(av_signed_amount) over POSTED vouchers of this year

and **not** `sum(av_signed_amount)` alone. Anything reading only `acc_vouchers`
is wrong by exactly the opening balance.

**There are zero views in this database** (`information_schema.views` returns no
rows in `accounts`, `inventory` or `public`), so nothing encapsulates that sum
today and every report will re-derive it. That is how the mistake gets made.

The fix is one view, not a fake voucher — something like
`accounts.vw_ledger_balance (company, branch, acc_year, ledger, opening_signed,
movement_signed, closing_signed)` doing the addition once, with reports reading
it instead of the tables. **OPEN DECISION** (and it belongs with DECISION 2, the
P&L carry).

---

## 6b. A ledger statement — how it is actually populated

Asked 2026-09-12. Two different statements, reading two different tables, and
conflating them is the usual mistake.

### 6b.1 One constraint to fix first

`avh_total_debit` / `avh_total_credit` are **stamped by the writer, not derived
from the legs**, and `ck_avh_balanced` compares only those two columns:

    CHECK (avh_voucher_status <> 'POSTED' OR avh_total_debit = avh_total_credit)

So a voucher whose legs do not add up — or one with no legs at all — passes the
one check written to prevent it, and a statement built on those legs is short by
however much is missing with nothing complaining. A trigger recomputing both
totals from `acc_vouchers` closes it. Reported in notes.txt 2026-09-12 (4).

(This surfaced because the dev box holds voucher headers with no legs — which is
simply because the posting to `acc_vouchers` is not written yet, not a defect.
The constraint gap is the finding, and the moment to close it is while that
posting is being written.)

### 6b.2 The movement statement (the ledger printout)

    opening line  =  the YEAR's opening        (acc_opening_balance)
                  +  movement BEFORE from_date (acc_vouchers)
    body          =  movements between from_date and to_date
    closing       =  opening line + the body

The opening LINE is not the year's opening unless the statement starts on the
first day of the year. That is the part that gets written wrong.

```sql
-- :company, :branch, :year, :ledger, :from, :to
WITH year_open AS (
  SELECT COALESCE(sum(op_amount * CASE op_dr_cr WHEN 'D' THEN 1 ELSE -1 END),0) AS signed
    FROM accounts.acc_opening_balance
   WHERE op_company_id = :company AND op_acc_year = :year
     AND op_ledger_id  = :ledger
     AND op_branch_id IS NOT DISTINCT FROM :branch
     AND NOT op_is_deleted
),
moved AS (
  SELECT v.*, h.avh_voucher_refno, h.avh_voucher_no
    FROM accounts.acc_vouchers v
    JOIN accounts.acc_voucher_header h
      ON h.avh_voucher_id = v.av_voucher_id
     AND h.avh_acc_year   = v.av_acc_year
   WHERE v.av_company_id = :company AND v.av_acc_year = :year
     AND v.av_ledger_id  = :ledger
     AND NOT v.av_is_deleted
     AND h.avh_voucher_status = 'POSTED'      -- never DRAFT or CANCELLED
),
before AS (
  SELECT COALESCE(sum(av_signed_amount),0) AS signed
    FROM moved WHERE av_voucher_date::date < :from
),
opening AS (
  SELECT (SELECT signed FROM year_open) + (SELECT signed FROM before) AS signed
)
SELECT m.av_voucher_date::date                         AS date,
       t.vchr_type_name                                AS voucher_type,
       m.avh_voucher_refno                             AS refno,
       COALESCE(o.led_name,'(as per details)')         AS particulars,
       CASE WHEN m.av_dr_cr = 'DR' THEN m.av_amount END AS debit,
       CASE WHEN m.av_dr_cr = 'CR' THEN m.av_amount END AS credit,
       (SELECT signed FROM opening)
         + sum(m.av_signed_amount) OVER (ORDER BY m.av_voucher_date,
                                                  m.avh_voucher_no, m.av_row_no
                                         ROWS UNBOUNDED PRECEDING) AS running_signed
  FROM moved m
  JOIN accounts.acc_voucher_types  t ON t.vchr_type_id = m.av_voucher_type_id
  LEFT JOIN accounts.acc_ledger_master o ON o.led_id  = m.av_opp_ledger_id
 WHERE m.av_voucher_date::date BETWEEN :from AND :to
 ORDER BY m.av_voucher_date, m.avh_voucher_no, m.av_row_no;
```

Four things in that query that are not decoration:

- **`running_signed > 0` prints as Dr, `< 0` as Cr.** The statement never shows a
  negative; same rule as the stored columns.
- **Particulars is the OPPOSITE ledger** (`av_opp_ledger_id`), which is how Tally
  reads. It is nullable — on a multi-leg voucher there is no single opposite, so
  print `(as per details)` rather than a blank.
- **`POSTED` only.** DRAFT is half-entered and CANCELLED is not a transaction;
  there is one CANCELLED header live already.
- **A statement spanning two years reads two partitions AND two opening rows.**
  Take the opening of the FIRST year only and union the movements — summing the
  openings of both years double-counts the whole of year one.

### 6b.3 The outstanding statement (a party's ageing) — a different table

For a bill-by-bill party, "statement of account" usually means the outstanding
invoices, not the movement. That reads `acc_bill_balance` and never touches
`acc_vouchers`:

```sql
SELECT abl_doc_date, abl_doc_refno, abl_bill_type, abl_due_date,
       abl_bill_amount, abl_pending_amount, abl_status,
       GREATEST(CURRENT_DATE - abl_due_date, 0) AS days_overdue
  FROM accounts.acc_bill_balance
 WHERE abl_company_id = :company AND abl_acc_year = :year
   AND abl_party_id   = :ledger
   AND NOT abl_is_deleted
   AND abl_pending_amount > 0          -- GENERATED, so this is always honest
 ORDER BY abl_doc_date;
```

**OPENING bills appear here with no special handling** — that is the whole payoff
of entering them one by one instead of as a single figure. `abl_pending_amount`
and `abl_status` are generated from the allocations, so ageing is correct the
moment a receipt is allocated.

### 6b.4 Put it behind a view / function, once

This is DECISION 8. `fn_ledger_statement(company, branch, year, ledger, from, to)`
for §6b.2 and `vw_party_outstanding` for §6b.3. There are **no views in this
database at all** today, so every report currently re-derives the opening join —
and that is precisely how one of them ends up wrong by exactly the opening.

## 7. Staleness

`op_is_stale` / `op_stale_since` + the partial index
`ix_op_stale (op_company_id, op_acc_year) WHERE op_is_stale AND NOT op_is_deleted`.

A voucher back-dated into a closed year changes that year's closing, which makes
every later CARRY_FORWARD opening derived from it wrong. The row is **flagged,
not corrected** — the figure may already have been reported, so a silent change
would be worse than a visible disagreement.

Nothing sets this flag today. **OPEN DECISION:** a trigger on `acc_vouchers`
(when `av_acc_year` < the current year), or a nightly job.

---

## 8. Every open decision in one place

1. **`led_ob_amount` vs `acc_opening_balance`** — which is authoritative (§1).
   Nothing can be built until this is answered.
2. Does the P&L result carry forward as a row on a 'Profit & Loss A/c' ledger,
   or is it derived on the fly (§4.2 step 3)?
3. A fourth `op_source` value for the stock tie, or not (§5)?
4. Who enforces the bill-wise tie — trigger or service (§4.3)?
5. What sets `op_is_stale` (§7)?
6. Branch-level openings: `op_branch_id` is nullable and `ux_op_scope` coalesces
   it, so both levels can coexist. Is that intended, and which one do reports
   read?
7. **No endpoint exists** for `acc_opening_balance` or for writing OPENING rows
   into `acc_bill_balance`.
8. **A balance view** (§6.4). There are no views in the database at all, so
   every report will otherwise re-derive `opening + movement` and one of them
   will forget.
