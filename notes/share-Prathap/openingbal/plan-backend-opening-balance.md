# Opening Balances — backend plan (NestJS)

**For Prathap.** Written against the schema at 192.168.0.106/ERP as read on
2026-09-14, and against the conventions the existing routes already follow.
Nothing is assumed; where the schema does not decide something it is marked
**DECISION** and left to you. The full list is in §11.

That box is a TEST database: its schema is authoritative and every rule here
rests on it, but the row counts quoted (29 balance-sheet ledgers, 4
bill-by-bill, 2 of 44 with a legacy opening) only indicate the SHAPE of the
problem. Do not size anything from them.

Companion documents, all in this folder (`Prathap/openingbal/`):

    opening_balance_flow.md          the flow, table by table, column by
                                     column — the spec this plan implements
    opening_balance_tables.png       the same thing as one diagram
    opening_balance_ui_mockup.png    the screen — ONE page, the bill-wise
                                     breakup is a detail panel on it
    schema_gaps_2026-09-14.md        the audit behind §2: what was found
                                     missing and why each fix is what it is

> Written for **NestJS**, the stack the ERP Server API already is. If Next.js
> route handlers were meant instead, the contract in §4 is unchanged — only the
> decorators move.

**The whole module in one line:** one row per balance-sheet ledger per
company-year, always positive with the side in a flag, never posted to the
voucher tables; a bill-by-bill party opens with its BILLS, not a figure; and
the set must balance.

---

## 1. What this module is, and the one thing that blocks it

`accounts.acc_opening_balance` is deployed, correct, and **empty**, and it has
no route of any kind. Neither does the OPENING side of
`accounts.acc_bill_balance`. So today a company cannot be opened on the system
at all: there is no way to say what anything was worth on the day the books
begin.

**Before any code — DECISION 1.** There are two homes for an opening balance and
the wrong one has the data:

| | `acc_ledger_master.led_ob_amount` / `led_ob_type` / `led_ob_as_on` | `accounts.acc_opening_balance` |
|---|---|---|
| shape | one figure per ledger, **no year** | one row per company/branch/**year**/ledger |
| provenance | none | `op_source`, `op_generated_at/by` |
| staleness | none | `op_is_stale`, `op_stale_since` |

A ledger's opening differs *per year*, so the ledger-master columns cannot
express 2025-26 and 2026-27 at once — they are a leftover of the Tally import
shape, where an opening balance is a property of the ledger. My reading is
that `acc_opening_balance` is authoritative and `led_ob_*` should be dropped
or explicitly redefined as "migration seed only". **This plan assumes that.**
If you disagree, say so before §4 is built, because every endpoint changes.

There is also a THIRD set of balance columns on the ledger master —
`led_total_balance / led_total_dr / led_total_cr`, stamped `NOT NULL DEFAULT 0`.
They are not derived from anything. Nothing in this module reads them.

---

## 2. Schema prerequisites — run these BEFORE the module

The table was deployed without the constraints that make it safe, and two
things the flow depends on have no home. All of it is small; §2.1–§2.4 should
not wait. Reasoning for each is in `schema_gaps_2026-09-14.md`.

### 2.1 Foreign keys on `acc_opening_balance` — it has none

`pg_constraint` holds the PK, the NOT NULLs and four CHECKs; nothing of
`contype = 'f'`. An opening for a ledger that does not exist is accepted.

```sql
ALTER TABLE accounts.acc_opening_balance
  ADD CONSTRAINT fk_op_ledger  FOREIGN KEY (op_ledger_id)
      REFERENCES accounts.acc_ledger_master (led_id)
      ON UPDATE CASCADE ON DELETE RESTRICT,
  ADD CONSTRAINT fk_op_company FOREIGN KEY (op_company_id)
      REFERENCES public.companys (comp_id)
      ON UPDATE CASCADE ON DELETE RESTRICT,
  ADD CONSTRAINT fk_op_branch  FOREIGN KEY (op_branch_id)
      REFERENCES public.branch_master (br_id)
      ON UPDATE CASCADE ON DELETE RESTRICT;
```

### 2.2 `op_tenant_id`

Every other partitioned accounts table carries one (`abl_`, `abj_`, `apd_`,
`td_`, `avh_`, `av_tenant_id`). This is the only one without.

```sql
ALTER TABLE accounts.acc_opening_balance ADD COLUMN op_tenant_id uuid;
```

Same type and nullability as `abl_tenant_id`; stamp it from the same source.

### 2.3 Two posting roles — the plug ledger and the P&L ledger

The flow has to find two ledgers by ROLE, never by name: 'Difference in
Opening Balances' (the plug, §7.1) and 'Profit & Loss A/c' (DECISION 2).
Nothing on the ledger marks either — `led_category` is only GENERAL/NULL and
there is no reserved flag. The mechanism already exists: `acc_ledger_role` +
`acc_ledger_map`, which is how the sale bill finds `ROUND_OFF`.

```sql
INSERT INTO accounts.acc_ledger_role (alr_role, alr_label, alr_want_nature, alr_want_type)
VALUES ('OPENING_DIFFERENCE', 'Difference in opening balances', 'Liabilities', NULL),
       ('RETAINED_EARNINGS',  'Profit & Loss / retained earnings', 'Liabilities', NULL);
```

Then one `acc_ledger_map` row per company for each. No DDL.

### 2.4 `acc_group_nature` — checked, and eventually NOT NULL

It is nullable with no CHECK, and a child group does not inherit its parent's
nature. That is why five ledgers live under groups with no nature at all.

```sql
ALTER TABLE accounts.acc_group_master
  ADD CONSTRAINT ck_acc_group_nature
  CHECK (acc_group_nature IS NULL
         OR acc_group_nature IN ('Assets','Liabilities','Income','Expenses'));
```

Back-fill the NULLs from the parent chain, then NOT NULL — or a trigger that
copies the parent's nature on insert. Until the back-fill, §4.1 keeps
returning `unclassified[]`.

### 2.5 `fiscal_years` — uniqueness

Live constraints are `chk_fy_dates` and the `fy_prev_fy_id` self-FK. Nothing
stops two `2026-2027` rows for one company, or two current years.

```sql
CREATE UNIQUE INDEX ux_fy_company_year    ON public.fiscal_years (comp_id, fy_year_name);
CREATE UNIQUE INDEX ux_fy_company_current ON public.fiscal_years (comp_id) WHERE fy_is_current;
ALTER TABLE public.fiscal_years
  ADD CONSTRAINT ck_fy_status CHECK (fy_status IN ('OPEN','CLOSED'));
```

Carry-forward (§5.2 step 6) resolves the two `fy_id`s by `(comp_id,
fy_year_name)` and needs the first index to be unambiguous.

### 2.6 `acc_opening_run` — DECISION 11

`fy_carried_forward_at` is one timestamp that a regenerate overwrites. The
counts and totals a carry-forward returns have no home once the response is
gone. Small, optional; this plan assumes it exists.

```sql
CREATE TABLE accounts.acc_opening_run (
  aor_id               uuid          PRIMARY KEY DEFAULT uuidv7(),
  aor_company_id       uuid          NOT NULL REFERENCES public.companys (comp_id),
  aor_branch_id        uuid          REFERENCES public.branch_master (br_id),
  aor_from_acc_year    char(9)       NOT NULL,
  aor_to_acc_year      char(9)       NOT NULL,
  aor_run_at           timestamptz   NOT NULL DEFAULT now(),
  aor_run_by           varchar(50),
  aor_created          int           NOT NULL,
  aor_updated          int           NOT NULL,
  aor_skipped_manual   int           NOT NULL,
  aor_total_debit      numeric(18,2) NOT NULL,
  aor_total_credit     numeric(18,2) NOT NULL,
  aor_overwrite_manual boolean       NOT NULL
);
```

### 2.7 What stays as it is — read this before "tidying"

- **`acc_bill_balance.abl_branch_id` stays NOT NULL.** A ledger opening can be
  company-level (`op_branch_id` NULL; `ux_op_scope` COALESCEs it) but a bill
  cannot. This plan takes **DECISION 9 as (a)**: bill-wise openings are
  branch-scoped, a bill-by-bill party's company-level opening is the sum of
  its branch bills, and the screen disables the breakup under "All branches".
  The alternative — nullable for OPENING — touches `ix_abl_party_credit` and
  every credit-limit query, for nothing.
- **`abl_src_doc_id` stays un-FK'd.** It points at many tables. The check that
  it names a live opening row is the service's (§5.3).
- **`op_dr_cr` is `'D'/'C'`, `abl_dr_cr` is `'DR'/'CR'`.** Different columns,
  different tables, both already constrained. Do not unify them in a DTO.

### 2.8 Deferred — known, not blocking

| what | decision |
|---|---|
| `led_ob_amount / led_ob_type / led_ob_as_on` — drop or redefine | 1 |
| a fourth `op_source` for the stock tie (`ck_op_source` refuses it today) | 7 |
| `op_stale_reason` / the voucher that made a row stale | with 5 |
| a cost-centre column — no cost-centre master exists yet, `av_cost_centre_id` points at nothing | later |

---

## 3. Module layout

```
src/modules/opening-balance/
  opening-balance.module.ts
  opening-balance.controller.ts
  opening-balance.service.ts        // §5.1 list / save / trial balance / delete
  carry-forward.service.ts          // §5.2 — its own service, it is a job
  bill-wise.service.ts              // §5.3
  ledger-roles.ts                   // resolves OPENING_DIFFERENCE / RETAINED_EARNINGS
                                    //   through acc_ledger_map, the way posting resolves ROUND_OFF
  dto/
    save-opening-balance.dto.ts
    save-opening-bill.dto.ts
    carry-forward.dto.ts
    opening-balance-payload.dto.ts
    trial-balance-payload.dto.ts
```

`carry-forward.service.ts` is separate deliberately: it is a batch operation
over a whole company-year inside one transaction, and it will grow a scheduled
caller. It should not live inside the CRUD service.

---

## 4. The endpoints

Six. The first four mirror the shape used everywhere else. Payload keys are
camelCase (DECISION 6 — see §8).

### 4.1 `GET /api/v1/opening-balances/list`

Query: `company_id` (required), `acc_year` (required), `branch_id` (optional
— absent means company level, `op_branch_id IS NULL`), `include_zero`
(optional, default false).

Returns **every balance-sheet ledger visible to the company**, whether or not
it has an opening — the screen is a review surface over a full set, not a
list of rows that happen to exist.

```sql
FROM accounts.acc_ledger_master l
JOIN accounts.acc_group_master  g ON g.acc_group_id = l.led_group_id
LEFT JOIN accounts.acc_opening_balance o
       ON o.op_ledger_id  = l.led_id
      AND o.op_company_id = :company_id
      AND o.op_acc_year   = :acc_year
      AND o.op_branch_id IS NOT DISTINCT FROM :branch_id
      AND NOT o.op_is_deleted
WHERE NOT l.led_is_deleted
  AND (l.led_company_id IS NULL OR l.led_company_id = :company_id)   -- DECISION 10
  AND g.acc_group_nature IN ('Assets','Liabilities')
```

`led_company_id` is nullable — a ledger is global (NULL) or owned by one
company — so the second WHERE line is not optional; without it one company's
private ledgers are listed to every other. **DECISION 10:** global + this
company (above), or global only, which is what grid 105 does. Pick one and say
which.

Per row: `opId`, `ledId`, `ledName`, `groupName`, `groupNature`,
`ledIsBillByBill`, `opAmount`, `opDrCr`, `opSource`, `opIsStale`,
`opStaleSince`, `priorClosingAmount` / `priorClosingDrCr` (§5.2 step 1, so the
screen shows last year's closing beside this year's opening), and `billCount`
for a bill-wise party.

Return, separately, the ledgers whose `acc_group_nature IS NULL` as
`unclassified[]`. They are a data problem the screen must surface, not
silently drop; the list empties once §2.4's back-fill is done.

**No paging cap.** A chart is a few hundred ledgers and the client shows a
trial balance over all of them; a capped list is a wrong trial balance.

### 4.2 `POST /api/v1/opening-balances/create`

Takes the **whole set for one company-year**, the same replace semantics as
`tax-rates` `lines[]`:

```jsonc
{
  "opCompanyId": "…", "opBranchId": null, "opAccYear": "2026-2027",
  "opTenantId": "…",
  "rows": [
    { "opId": "…",            // present = update, absent = insert
      "opLedgerId": "…",
      "opAmount": 124500.00,  // ALWAYS POSITIVE
      "opDrCr": "D",          // 'D' | 'C'  — ONE character
      "opSource": "MIGRATION",
      "opRemarks": null }
  ],
  "replace": true             // rows absent from the array are soft deleted
}
```

`rows: []` with `replace: true` means *delete every opening for this
company-year* — a legitimate request. Do not let it be confused with an omitted
key.

This is also the edit endpoint — there is no other. See §5.5 for what an edit
must do that a plain upsert does not (source flips to MANUAL; a bill-by-bill
ledger's amount is refused unless it echoes the bill total).

### 4.3 `GET /api/v1/opening-balances/trial-balance`

Query: `company_id`, `acc_year`, `branch_id?`. Returns

```jsonc
{ "totalDebit": …, "totalCredit": …, "difference": …, "isBalanced": true,
  "unmappedCount": 0,
  "differenceLedgerId": "…" }   // the OPENING_DIFFERENCE ledger via acc_ledger_map; null if unmapped
```

Its own endpoint rather than a field on the list, because the screen shows it
permanently and recomputes it as the operator types — and because the save
path checks it server-side too (§7.1). `differenceLedgerId` is how the screen
writes the plug row against the right ledger without knowing its name; null
means the company has not mapped the role and the plug cannot be offered.

### 4.4 `DELETE /api/v1/opening-balances/delete`

Query `op_id` + `acc_year` (the partition key — the PK is
`(op_id, op_acc_year)`). Soft delete: `op_is_deleted = true`. Refused while
OPENING bills still point at the row (§5.3) — delete the bills first.

### 4.5 `POST /api/v1/opening-balances/carry-forward`

```jsonc
{ "companyId": "…", "fromAccYear": "2025-2026", "toAccYear": "2026-2027",
  "branchId": null,
  "overwriteManual": false }   // §7.3
```

Returns `{ runId, created, updated, skippedManual, totalDebit, totalCredit,
difference }`. Full logic in §5.2. **One transaction.**

### 4.6 Bill-wise — `GET` and `POST /api/v1/opening-balances/bills`

`GET` query: `party_id`, `company_id`, `acc_year`, **`branch_id` (required)**
→ the OPENING rows for that party at that branch, plus the tie figures.

`POST` takes the whole breakup for one party at one branch, same replace
semantics:

```jsonc
{ "companyId": "…", "branchId": "…",        // REQUIRED — abl_branch_id is NOT NULL (§2.7)
  "accYear": "2026-2027",
  "partyId": "…",
  "opId": "…",                              // the party's acc_opening_balance row (§5.3)
  "bills": [
    { "ablId": "…",
      "ablDocRefno": "SB/2025/0412",        // NOT NULL — the ORIGINAL invoice no
      "ablDocDate": "2026-01-12",           // the ORIGINAL invoice date
      "ablDueDate": "2026-02-11",           // nullable; >= ablDocDate
      "ablCreditDays": 30,
      "ablGraceDays": 0,
      "ablDrCr": "DR",                      // 'DR' | 'CR' — TWO characters here
      "ablBillAmount": 48600.00,            // > 0
      "ablNarration": null }
  ],
  "replace": true }
```

This endpoint OWNS the party's opening figure: it writes `op_amount /
op_dr_cr` from the bills in the same transaction (§5.5 rule 2). A bill that
has been receipted against accepts only date / days / narration changes
(§5.5 rule 3).

---

## 5. Service logic

### 5.1 Save (§4.2)

1. Reject a row whose ledger's group nature is not Assets or Liabilities. An
   opening for an income ledger is a category error, not a zero.
2. Reject a ledger the company cannot see under the §4.1 visibility rule.
   `fk_op_ledger` proves the ledger exists, not that this company may use it.
3. Reject `opAmount < 0` before the database does, with the row index and the
   ledger name — `ck_op_amount` will refuse it anyway but the message will name
   a constraint.
4. Skip-and-report a row whose amount is 0: write **no row**. Absence is the
   zero, and `ux_op_scope` stays meaningful.
5. Upsert on `ux_op_scope` = `(op_company_id, COALESCE(op_branch_id, '000…0'),
   op_acc_year, op_ledger_id) WHERE NOT op_is_deleted`. Note the COALESCE —
   a plain `ON CONFLICT` on the four raw columns will not match it.
6. `replace: true` → soft delete the rows for that company-year not present in
   the array — except a bill-by-bill party's row that still has OPENING bills
   (§5.3); report those instead.
7. Stamp `op_tenant_id` the way `abl_tenant_id` is stamped.
8. Translate constraint failures into messages: `fk_op_ledger` → "ledger no
   longer exists", `fk_op_branch` → "branch not found". A constraint name is
   not a message.
9. Whole thing in **one transaction**.

### 5.2 Carry forward (§4.5) — the part worth getting exactly right

Inside one transaction:

**Step 1 — prior closing per ledger.**

```sql
prior_opening_signed =
    op_amount * CASE op_dr_cr WHEN 'D' THEN 1 ELSE -1 END
  FROM accounts.acc_opening_balance
 WHERE op_company_id=:c AND op_acc_year=:from AND NOT op_is_deleted

movement =
    sum(v.av_signed_amount)          -- GENERATED: CASE av_dr_cr WHEN 'DR' THEN av_amount ELSE -av_amount END
  FROM accounts.acc_vouchers v
  JOIN accounts.acc_voucher_header h
    ON h.avh_voucher_id = v.av_voucher_id AND h.avh_acc_year = v.av_acc_year
 WHERE v.av_company_id=:c AND v.av_acc_year=:from
   AND NOT v.av_is_deleted
   AND h.avh_voucher_status = 'POSTED'

closing_signed = COALESCE(prior_opening_signed,0) + COALESCE(movement,0)
```

`ck_avh_status` allows DRAFT | APPROVED | POSTED | CANCELLED, and
`ck_avh_balanced` only requires debit = credit **at POSTED** — so POSTED is the
only status a closing balance may be built from. Do not include APPROVED. And
never `led_total_*` (§1).

**Step 2 — split the sign back out.** This is the step that gets written wrong:

```
op_amount = abs(closing_signed)
op_dr_cr  = closing_signed >= 0 ? 'D' : 'C'
```

Never store a signed figure; `ck_op_amount` refuses it.

**Step 3 — balance-sheet ledgers only**, same filter as §4.1. Income and
expense net off to the P&L result — **DECISION 2:** does that land as a
CARRY_FORWARD row on the `RETAINED_EARNINGS` ledger (Tally's model), or is it
derived on the fly? Nothing in the schema decides it, and the trial balance
will not balance until it is. If the Tally model: the ledger is resolved
through the role (§2.3), and a company with no mapping cannot be carried
forward — say so in the response.

**Step 4 — bill-wise parties carry their BILLS, not just the figure.** For
every party with `led_is_bill_by_bill`, copy each still-open
`acc_bill_balance` row of the from-year into the to-year as a **new** row:

```
abl_bill_type        = 'OPENING'
abl_bill_amount      = <the old row's abl_pending_amount>
abl_alloc_amount     = 0, abl_disc_amount = 0, abl_writeoff_amount = 0
abl_branch_id        = <the old row's branch>          -- NOT NULL; bills carry per branch
abl_voucher_id       = NULL, abl_voucher_type_id = NULL -- ck_abl_voucher forces it
abl_parent_bill_id   = <the old row's abl_id>          -- chains the bill across the year-end
abl_parent_acc_year  = <from-year>
abl_src_module       = 'ACCOUNTS'
abl_src_doc_type     = 'OPENING_BALANCE'
abl_src_doc_id       = <the party's new op_id in the to-year>
abl_src_acc_year     = <to-year>
```

`fk_abl_parent_1..3` already FK the parent pair across the partitions and
`ck_abl_parent` only *requires* it for INTEREST — an OPENING row is free to set
it. Without the parent link a receipt history across the boundary cannot be
reconstructed; without the src link the tie in §7.2 has no key. The party's
`op_amount` must then equal the total of its carried bills. **Without this
step, carry-forward silently destroys the ageing of every debtor.**

**Step 5 — `op_source = 'CARRY_FORWARD'`, `op_generated_at = now()`,
`op_generated_by = <user>`**, and `op_is_stale = false`.

**Step 6 — stamp the year:**

```sql
UPDATE public.fiscal_years
   SET fy_is_carried_forward = true,
       fy_carried_forward_at = now(),
       fy_prev_fy_id         = :from_fy_id
 WHERE fy_id = :to_fy_id
```

Both ids resolved by `(comp_id, fy_year_name)`, unique after §2.5. Those three
columns already exist for exactly this; live, `fy_is_carried_forward` is
**false on every row** — no year has ever been carried forward.

**Step 7 — record the run** (§2.6): one `acc_opening_run` row with the counts
and totals; its `aor_id` is the `runId` in the response. A regenerate adds a
row, never overwrites one.

### 5.3 Bill-wise (§4.6)

Per row:

```
abl_bill_type        = 'OPENING'
abl_branch_id        = <branchId>                 -- NOT NULL
abl_voucher_id       = NULL, abl_voucher_type_id = NULL
abl_src_module       = 'ACCOUNTS'
abl_src_doc_type     = 'OPENING_BALANCE'
abl_src_doc_id       = <opId>
abl_src_acc_year     = <accYear>
abl_parent_bill_id   = NULL                       -- typed bills have no predecessor
```

The source link is what makes the tie (§7.2) a keyed query rather than a join
on (company, year, party) that also sweeps up the party's SALES bills of the
same year. `ck_abl_src_doc` requires id and type together, and
`ix_abl_src_doc` already indexes the pair. Before writing, check that `opId`
names a live `acc_opening_balance` row for that party, company and year —
`abl_src_doc_id` is deliberately un-FK'd (§2.7), so this check is yours.

**Never send `abl_pending_amount` or `abl_status` — both are GENERATED ALWAYS:**

```sql
abl_pending_amount = abl_bill_amount - abl_alloc_amount
                                     - abl_disc_amount - abl_writeoff_amount
abl_status         = CASE WHEN pending <= 0 THEN 'CLOSED'
                          WHEN alloc+disc+writeoff > 0 THEN 'PARTIAL'
                          ELSE 'OPEN' END
```

Then enforce the tie (§7.2).

### 5.4 Nothing is posted to `acc_vouchers` — settled, not assumed

The opening does not write `acc_voucher_header` or `acc_vouchers`, and the
service should not be built with a switch for it. Two pieces of evidence, both
from the live schema:

- `ck_abl_voucher` **forces** an `'OPENING'` bill's `abl_voucher_id` and
  `abl_voucher_type_id` to be NULL. A party's opening bills could not reference
  an opening voucher even if one were posted.
- the argument that settlement needs a voucher line is **false**:
  `acc_bill_adjustment.abj_bill_id` is NOT NULL and FKs to `acc_bill_balance`,
  while `abj_voucher_id` and `abj_voucher_line_id` are both nullable. An
  allocation is against the BILL; `abj_voucher_id` is the receipt that pays it.

So an opening bill settles exactly like any other. The cost of not posting is
that a balance is `opening + movement`, never `movement` alone — pay it once
with a view (DECISION 8), not with a fabricated voucher.

### 5.5 Editing after the first save

There is no separate edit mode and no edit endpoint. The unit of work is the
company-year: the screen loads the whole set (§4.1), the operator changes what
needs changing, and `POST /create` with `replace: true` upserts it — `opId`
present = update, absent = insert, amount 0 or row absent = soft delete. Bills
edit the same way through `POST /bills`. A CLOSED or locked year refuses the
write (§7.4). That much falls out of the upsert. Four rules do not, and each
one is a way an edit silently corrupts something:

**Rule 1 — an edited CARRY_FORWARD row becomes MANUAL.** The client sends
`opSource` back as it received it. If the service stores it as sent, an
accountant's correction to a generated figure stays `CARRY_FORWARD`, and the
next regenerate — which only spares `MANUAL` rows (§7.3) — undoes it without a
word. So: when a row's stored `op_source` is `CARRY_FORWARD` and the incoming
`opAmount` or `opDrCr` differs from what is stored, write `op_source =
'MANUAL'` regardless of what was sent, and keep `op_generated_at / _by` as
they were — they record what the figure was derived from before it was
overridden. `MIGRATION` rows stay `MIGRATION`; typing them is what migration
is.

**Rule 2 — for a bill-by-bill party, the bills write the opening.** The
party's `op_amount` is the total of its OPENING bills, not a figure of its
own, and the screen shows it read-only. So `POST /bills` recomputes
`op_amount / op_dr_cr` from the bills it has just written and updates the
party's `acc_opening_balance` row **in the same transaction** — inserting it
if the party had none. The tie (§7.2) is then true by construction. The
other half: `POST /create` REFUSES a row whose ledger is `led_is_bill_by_bill`
unless the amount and side equal the current bill total (i.e. the screen is
just echoing it back). Two endpoints writing the same figure from different
sources is how the tie breaks; one owner, and the other only reads.

**Rule 3 — a bill that has been receipted against is frozen.** Once
`abl_alloc_amount + abl_disc_amount + abl_writeoff_amount > 0` (status
PARTIAL or CLOSED), reducing `abl_bill_amount` below that fails at
`ck_abl_settled`, changing `abl_dr_cr` inverts a settlement, and soft-deleting
the row leaves `acc_bill_adjustment` pointing at a bill that is gone. So on
such a bill the service accepts changes to `abl_due_date`, `abl_credit_days`,
`abl_grace_days` and `abl_narration` only; any other change, or its absence
from a `replace: true` array, is refused with the bill's refno and its
allocated amount in the message.

This freezes ONE bill's figure — not the party, not the screen. Every other
row and bill saves as normal, and the frozen bill rides along unchanged in
the array. When the figure genuinely was wrong there are two honest fixes,
and the message should name them: **add a correcting OPENING bill** for the
difference (a further Dr or Cr line — the party's opening is the SUM of its
bills, so a correction is one more bill, and the receipt's allocation on the
original is untouched); or, when the receipt was allocated to the wrong bill,
reverse the allocation in the receipt screen first, which returns the bill
to OPEN with `abl_alloc_amount = 0`, then edit or delete it and re-allocate.
Ledger rows without bills have no freeze at all: vouchers posted against a
ledger never block editing its opening, because a balance is always
opening + movement.

**Rule 4 — editing year N after N+1 was carried forward stales N+1.** §7.5
covers a voucher back-dated into N. An edit to N's openings, or to an OPENING
bill of N, moves N's closing just the same, so N+1's `CARRY_FORWARD` rows are
now derived from a figure that no longer exists. Whatever implements §7.5
(trigger or service) must also fire on writes to `acc_opening_balance` and to
OPENING rows of `acc_bill_balance`: if `fiscal_years` for N+1 has
`fy_is_carried_forward`, mark N+1's `CARRY_FORWARD` rows for the same company
(and branch, or all branches when `op_branch_id` is NULL) stale. Flag, never
correct — same reasoning as §7.5.

Two smaller things, not rules:

- **History.** The table holds `op_modified_at / _by` and nothing about what
  the figure was before. If "who changed 1,24,500 to 1,20,000 and when"
  matters — and for an opening balance it usually does — this screen wants an
  `audit.audit_screen` id like the master screens have, so the client's
  History button works. Nothing to do in this module; noting it so it is not
  discovered after go-live.
- **Two operators on one year.** `replace: true` means the last save wins and
  soft-deletes whatever the other operator added since. Cheap guard: the
  client sends the `op_modified_at` it loaded per row, the service refuses a
  row whose stored value is newer, and the screen reloads. Optional; say if
  you want it in the DTO.

---

## 6. Rules the DATABASE enforces — do not restate them differently

| constraint | what it forces |
|---|---|
| `ck_op_amount` | `op_amount >= 0` — positive amount, side in `op_dr_cr` |
| `ck_op_dr_cr` | `'D'` or `'C'` |
| `ck_op_source` | `CARRY_FORWARD` \| `MANUAL` \| `MIGRATION` — **only these three** |
| `ck_op_acc_year` | `^\d{4}-\d{4}$` and right = left + 1 |
| `ux_op_scope` | one live row per company / branch-or-zero / year / ledger |
| `fk_op_ledger` / `fk_op_company` / `fk_op_branch` | *(§2.1)* the opening points at a real ledger, company, branch |
| `ck_abl_voucher` | `'OPENING'` ⇒ `abl_voucher_id` **and** `abl_voucher_type_id` NULL; every other type ⇒ both NOT NULL |
| `abl_branch_id NOT NULL` | a bill always has a branch (§2.7) |
| `ck_abl_amount` | `abl_bill_amount > 0` — never zero, never negative |
| `ck_abl_due_date` | `abl_due_date IS NULL OR >= abl_doc_date` |
| `ck_abl_src_doc` | `abl_src_doc_id` and `abl_src_doc_type` both NULL or both set |
| `ck_abl_settled` | `alloc + disc + writeoff <= bill_amount` |
| `ux_abl_doc_refno` | one live bill per (company, party, type, year, refno) — an OPENING reference cannot be typed twice |
| `fk_abl_parent_*` | a carried bill's parent exists, in the partition it names |
| `ck_acc_group_nature` | *(§2.4)* a group's nature is one of the four or NULL |
| `ux_fy_company_year` / `ux_fy_company_current` | *(§2.5)* one row per company-year; one current year per company |

Note the asymmetry, and do not "tidy" it in the DTOs: **`op_dr_cr` is one
character (`D`/`C`) and `abl_dr_cr` is two (`DR`/`CR`).**

---

## 7. Rules ONLY the service can enforce

### 7.1 The set must balance
`sum(op_amount) FILTER (op_dr_cr='D')` must equal the `'C'` total for the
company-year. An unbalanced opening set is not an opening set. **DECISION 3:**
refuse the save, or accept it and report the difference the way Tally does (it
shows "Difference in Opening Balances" and lets you carry on). My
recommendation is to accept the save but return `isBalanced: false` with the
difference, and let the screen refuse — refusing server-side blocks an operator
who is halfway through typing 200 ledgers. The plug ledger is the one mapped to
`OPENING_DIFFERENCE` (§2.3), returned by §4.3 as `differenceLedgerId`.

### 7.2 A bill-wise party's bills must total its opening

```sql
SELECT sum(abl_bill_amount * CASE abl_dr_cr WHEN 'DR' THEN 1 ELSE -1 END)
  FROM accounts.acc_bill_balance
 WHERE abl_src_doc_type = 'OPENING_BALANCE'
   AND abl_src_doc_id   = :op_id          -- the §5.3 link; ix_abl_src_doc
   AND abl_acc_year     = :acc_year
   AND NOT abl_is_deleted
-- must equal
   op_amount * CASE op_dr_cr WHEN 'D' THEN 1 ELSE -1 END
```

Keyed on the source link, not on (company, year, party). Nothing in the
database checks this. **DECISION 4:** trigger or service. If the service, it
must run inside the same transaction as both writes.

### 7.3 A MANUAL row is not silently overwritten
Carry-forward must skip rows with `op_source = 'MANUAL'` unless
`overwriteManual: true`, and report `skippedManual` either way. Otherwise a
regenerate quietly undoes the accountant's correction.

### 7.4 A closed or locked year is not writable
`fy_status = 'CLOSED'` and `fy_lock_date` are both on `fiscal_years` and nothing
consults them. Refuse writes into a closed year, and into a year whose
`fy_lock_date` is past, with a message naming the year.

### 7.5 Staleness
A voucher posted into a year **earlier** than one already carried forward
invalidates every later CARRY_FORWARD opening derived from it. Set
`op_is_stale = true`, `op_stale_since = now()` — **flag, never correct**: the
figure may already have been reported. **DECISION 5:** a trigger on
`acc_vouchers`, or a nightly job. The partial index `ix_op_stale` is already
there to make the "which are stale" query free. The same mechanism must fire
on an EDIT to an earlier year's openings or OPENING bills — §5.5 rule 4.

---

## 8. Conventions to follow from the existing API

1. **Paging caps are real and they bite.** `/dropdown-details/run` declares
   `limit` max 100 and answers **400** above it — that is what left the GST rate
   master's role combo silently empty for a day. §4.1 has no cap for that
   reason.
2. **`create` is an upsert** — id present = update. Keep that.
3. **`replace` semantics** for child arrays, exactly as `tax-rates` `lines[]`.
4. **Nulls must be writable.** `SaveAccountLedgerMasterDto.ledCompanyId` is a
   non-nullable uuid string and the service drops nulls on merge, which is why
   no ledger can be moved back to global from any screen. Do not repeat that
   here: `opBranchId` and `opRemarks` must be settable to null explicitly.
5. **Tenant.** Stamp `op_tenant_id` from the same source `abl_tenant_id` and
   `av_tenant_id` come from.
6. **DECISION 6 — payload casing.** `tax-rates` uses snake_case (`tax_id`),
   everything else uses camelCase (`ledId`, `sbId`). This plan uses camelCase
   (`opId`, `opLedgerId`) because the module sits beside the accounts modules.
   Just be deliberate — the client hard-codes key names.

---

## 9. Test checklist

- A zero-amount row writes **no** row, and does not leave a stale one behind.
- `opAmount: -500` is refused by the service with a readable message, before
  the constraint fires.
- Upsert matches `ux_op_scope` **including the COALESCE on branch** — a company
  row and a branch row for the same ledger must be able to coexist.
- `rows: []` + `replace: true` clears the year; an omitted `rows` key does not.
- A ledger owned by another company is refused on save.
- An opening for a ledger that does not exist is refused with a message, not
  with `fk_op_ledger`.
- Carry-forward: a ledger closing 1,000 Dr opens 1,000 D; one closing 1,000 Cr
  opens 1,000 **C** — the sign split, in both directions.
- Carry-forward skips a MANUAL row and reports it.
- Carry-forward of a bill-wise party produces one OPENING bill per still-open
  bill, with `abl_pending_amount` of the source as the new `abl_bill_amount`,
  `abl_parent_bill_id` = the source bill, the source bill's branch, and the
  party's opening equals their total.
- Carry-forward writes one `acc_opening_run` row; a second run adds a second.
- An OPENING bill with a voucher id is refused by `ck_abl_voucher` — assert the
  service never sends one.
- A bill POSTed with no `branchId` is refused with a message, before
  `abl_branch_id`'s NOT NULL fires.
- Every OPENING bill written carries `abl_src_doc_type = 'OPENING_BALANCE'`
  and the party's `op_id`; a bill whose `opId` names another party's opening,
  or a deleted one, is refused.
- The tie counts ONLY bills linked to the opening — a SALES bill of the same
  party and year does not move it.
- Deleting an opening row that still has OPENING bills is refused.
- Editing the amount of a CARRY_FORWARD row and saving stores it as MANUAL;
  the next carry-forward skips it and reports it. Re-saving it unchanged keeps
  CARRY_FORWARD.
- `POST /bills` for a party with three bills totalling 2,18,400 Dr leaves the
  party's opening row at 2,18,400 D; adding a 2,300 Cr bill moves it to
  2,16,100 D without any call to `/create`.
- `POST /create` with a hand-typed amount on a bill-by-bill ledger that does
  not equal its bill total is refused.
- A PARTIAL bill accepts a new due date; refuses a smaller amount, a flipped
  side, and omission from a `replace: true` array — each with the refno.
- Editing 2025-2026's openings after 2026-2027 was carried forward marks
  2026-2027's CARRY_FORWARD rows stale; MANUAL rows in 2026-2027 are untouched.
- A write into a CLOSED year is refused.
- Trial balance: Dr = Cr returns `isBalanced: true`; a 9,310 difference is
  reported as 9,310 and not swallowed; `differenceLedgerId` is null for a
  company with no `OPENING_DIFFERENCE` mapping.

---

## 10. Phases

0. **§2** — the schema prerequisites, and the two roles mapped for at least
   one company.
1. **DECISION 1**, then §4.1 + §4.2 + §4.3 — list, save, trial balance. That is
   a company being opened by hand, which is the first thing anyone needs.
2. §4.6 bill-wise, with the §7.2 tie.
3. §4.5 carry-forward, including step 4 (the bills). Not before 2 — a
   carry-forward that cannot carry bills is worse than none.
4. §7.5 staleness.
5. DECISION 7 stock tie; DECISION 8 balance view.

---

## 11. Decisions

| # | where | status |
|---|---|---|
| 1 | §1 — `led_ob_*` vs `acc_opening_balance` | **open — blocks everything** |
| 2 | §5.2 step 3 — P&L result: `RETAINED_EARNINGS` row or derived | open |
| 3 | §7.1 — unbalanced set: refuse or accept + report | recommended: accept + report |
| 4 | §7.2 — bill-wise tie: trigger or service | open |
| 5 | §7.5 — what sets `op_is_stale` | open |
| 6 | §8 — payload casing | plan uses camelCase |
| 7 | §2.8 — stock tie / fourth `op_source` | open |
| 8 | flow doc §6.4 — the balance view; zero views exist today | open |
| 9 | §2.7 — `abl_branch_id` NOT NULL | **plan takes (a): bills are branch-scoped** |
| 10 | §4.1 — ledger visibility: global + this company, or global only | open; plan assumes global + this company |
| 11 | §2.6 — carry-forward run log | plan assumes yes |

---

## 12. Do not build

- **No opening voucher, and no switch for one.** §5.4. `ck_abl_voucher`
  documents the decision.
- **No signed amounts anywhere** on the wire. Positive + flag, in both
  spellings.
- **No reading of `led_ob_*` or `led_total_*`** for any figure this module
  returns. The first is DECISION 1's legacy home; the second is stamped and
  derived from nothing.
- **No "unclassified means zero".** A ledger under a NULL-nature group is
  reported, never dropped and never defaulted to Assets.
- **No paging on the list.** A partial chart is a wrong trial balance.
- **No name lookups** for the plug or P&L ledger. Roles, through the map.
- **No second writer of a bill-by-bill party's opening.** The bills endpoint
  owns it; `create` only echoes it (§5.5 rule 2).
- **No "edit" endpoint.** The upsert is the edit; a separate PUT would be a
  second code path for the same rules.

