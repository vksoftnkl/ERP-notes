# Opening balances — schema gaps and plan corrections (2026-09-14)

**For Prathap.** A second pass over the live schema at 192.168.0.106/ERP, two
days after `opening_balance_flow.md` and `plan-backend-opening-balance.md`
were written. Everything below was read off `pg_constraint`, `pg_indexes` and
`information_schema` today; nothing is assumed. **This file is the audit trail; the plan is the spec.**
`plan-backend-opening-balance.md` was rewritten the same day with everything
here folded in — the DDL as its §2 "Schema prerequisites", the corrected
contracts in its §4–§5, decisions 9–11 in its §11. Read the plan to build;
read this to see WHY each of those is what it is. `opening_balance_flow.md`
§4.3 has its two wrong lines corrected in place.

Still true today: `acc_opening_balance` has 0 rows, no route, no trigger;
partitions `2024_2025 / 2025_2026 / 2026_2027` exist; DECISION 1
(`led_ob_*` vs `acc_opening_balance`) is still open.

---

## 1. Corrections to the two earlier documents

### 1.1 CORRECTION — `abl_branch_id` is NOT NULL (new DECISION 9)

`opening_balance_flow.md` §4.3 says `abl_branch_id = <branch or NULL>` and the
backend plan §3.6 shows `"branchId": null`. Both are wrong:

    accounts.acc_bill_balance.abl_branch_id   uuid   NOT NULL

So the two tables disagree on scope. A *ledger* opening can be company-level
(`op_branch_id` NULL, and `ux_op_scope` COALESCEs it), but a party's *bills*
cannot — every OPENING row in `acc_bill_balance` must name a branch.

**DECISION 9** — one of:

- (a) bill-wise openings are always branch-scoped; the screen disables the
  breakup panel when the scope is "All branches", and a bill-by-bill party's
  company-level opening is the SUM of its branch-level bills; or
- (b) `abl_branch_id` becomes nullable for `abl_bill_type = 'OPENING'` — which
  touches `ix_abl_party_credit` and every credit-limit query that keys on it.

(a) needs no DDL and matches how the sale bill already writes bills. My
recommendation is (a).

### 1.2 CORRECTION — the list join needs a company filter on the LEDGER

`acc_ledger_master.led_company_id` is nullable: a ledger is either global
(NULL) or owned by one company. The §3.1 list SQL in the plan filters the
opening rows by company but not the ledgers, so it would list one company's
private ledgers to every other company. Add:

    AND (l.led_company_id IS NULL OR l.led_company_id = :company_id)

(Grid 105 "POPUP - LEDGERS FOR ROLE" filters `led_company_id IS NULL` only —
global ledgers — which is the other defensible rule. Pick one and say which.)

### 1.3 A THIRD balance home exists — do not read it

`acc_ledger_master` also carries `led_total_balance / led_total_dr /
led_total_cr` (numeric, NOT NULL DEFAULT 0). They are stamped, not derived —
the same shape as `avh_total_debit / avh_total_credit` (notes.txt (4)). The
carry-forward closing balance (§4.2 step 1) must be built from
`acc_opening_balance + acc_vouchers` and never from these.

---

## 2. Missing on `accounts.acc_opening_balance` itself

| # | gap | kind | consequence |
|---|---|---|---|
| 2.1 | **No foreign keys at all.** `pg_constraint` holds the PK, the NOT NULLs and four CHECKs — nothing of `contype = 'f'`. `op_ledger_id`, `op_company_id`, `op_branch_id` reference nothing. Compare `acc_bill_balance`, which has `fk_abl_party`. | DDL | An opening for a deleted or non-existent ledger is accepted silently, and a ledger with an opening can be hard-deleted from under it. |
| 2.2 | **No `op_tenant_id`.** Every other partitioned accounts table has one: `abl_`, `abj_`, `apd_`, `td_`, `avh_`, `av_tenant_id`. This is the only one without. | column | Whatever tenant scoping the API applies elsewhere cannot be applied here. |
| 2.3 | **No cost-centre column.** `acc_vouchers.av_cost_centre_id` and `led_is_cost_center_req` exist, but there is no cost-centre master table anywhere in the database and the opening has no slot. | column, later | A cost-centre-required ledger's opening cannot be split. Not urgent until cost centres exist at all — noted so the column is not forgotten when they do. |
| 2.4 | `ck_op_source` has no value for the stock tie. | CHECK | Known — DECISION 7. |
| 2.5 | `op_is_stale` / `op_stale_since` but nothing records *why* — no `op_stale_reason`, no reference to the voucher that invalidated it. | column, minor | The screen can say "stale" but not "because voucher X was posted into 2025-2026 on <date>", which is what the accountant will ask. |

Suggested DDL for 2.1 (the one that should not wait):

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

(Column names on `companys` / `branch_master` as read today: `comp_id`,
`br_id`. Verify before running — the point is the FKs, not the spelling.)

---

## 3. Missing AROUND it — what the flow needs and nothing provides

### 3.1 No way to identify the two system ledgers the flow depends on

The flow needs to find, by role and not by name, two ledgers:

- **'Difference in Opening Balances'** — the plug row (flow §4.1 step 4,
  plan §6.1);
- **'Profit & Loss A/c'** — where the P&L result lands on carry-forward
  (DECISION 2), if that is the model chosen.

There is nothing on the ledger to mark either. `acc_group_master` has
`acc_group_is_reserved`; `acc_ledger_master` has no reserved / system flag,
and `led_category` holds only `GENERAL` or NULL.

**The mechanism already exists and needs no DDL:** `accounts.acc_ledger_role`
+ `accounts.acc_ledger_map`. Twenty roles are live today, all transactional
(`SALES`, `ROUND_OFF`, `INPUT_CGST`, `ADVANCE_RECEIVED`, …), and the sale bill
finds its Round Off ledger through exactly this. Two new rows:

```sql
INSERT INTO accounts.acc_ledger_role (alr_role, alr_label, alr_want_nature, alr_want_type)
VALUES ('OPENING_DIFFERENCE', 'Difference in opening balances', 'Liabilities', NULL),
       ('RETAINED_EARNINGS',  'Profit & Loss / retained earnings', 'Liabilities', NULL);
```

then one `acc_ledger_map` row per company pointing each at its ledger. The
opening-balance service resolves them the same way the posting service
resolves `ROUND_OFF`. (The `alr_want_nature` values are a suggestion —
'Suspense A/c' is a Liabilities group live, and P&L sits under Capital.)

### 3.2 `acc_group_nature` is nullable, unchecked, and not inherited

    accounts.acc_group_master.acc_group_nature   varchar(20)   NULL   — no CHECK

This is *why* five ledgers are "unclassified": a child group ("branch Office
accounts", "personal accounts", …) can be created under a parent with no
nature of its own, and nothing fills it in from the parent chain. The
`unclassified[]` return in plan §3.1 treats the symptom. The fix is at the
group:

```sql
ALTER TABLE accounts.acc_group_master
  ADD CONSTRAINT ck_acc_group_nature
  CHECK (acc_group_nature IS NULL
         OR acc_group_nature IN ('Assets','Liabilities','Income','Expenses'));
```

plus either NOT NULL (after back-filling from the parent chain) or a trigger
that copies the parent's nature on insert when none is given. Until then the
screen keeps showing the unclassified list.

### 3.3 `public.fiscal_years` has no uniqueness

Constraints live today: `chk_fy_dates (fy_end_date > fy_begin_date)` and the
`fy_prev_fy_id` self-FK. That is all. So:

- two `2026-2027` rows for one company are accepted, and carry-forward
  (plan §4.2 step 6) has nothing unambiguous to key `:to_fy_id` on;
- two rows can both be `fy_is_current` for one company;
- `fy_status` is free text (`'OPEN' | 'CLOSED'` by convention only).

```sql
CREATE UNIQUE INDEX ux_fy_company_year ON public.fiscal_years (comp_id, fy_year_name);
CREATE UNIQUE INDEX ux_fy_company_current ON public.fiscal_years (comp_id)
  WHERE fy_is_current;
ALTER TABLE public.fiscal_years
  ADD CONSTRAINT ck_fy_status CHECK (fy_status IN ('OPEN','CLOSED'));
```

### 3.4 No carry-forward run log

`fy_is_carried_forward / fy_carried_forward_at / fy_prev_fy_id` record THAT it
happened and WHEN — once. A regenerate overwrites the timestamp, and
`skippedManual`, the Dr/Cr totals, the difference, and who ran it have no
home. The plan's §3.5 response returns them and then they are gone.

Optional, small:

```sql
CREATE TABLE accounts.acc_opening_run (
  aor_id            uuid        PRIMARY KEY DEFAULT uuidv7(),
  aor_company_id    uuid        NOT NULL,
  aor_branch_id     uuid,
  aor_from_acc_year char(9)     NOT NULL,
  aor_to_acc_year   char(9)     NOT NULL,
  aor_run_at        timestamptz NOT NULL DEFAULT now(),
  aor_run_by        varchar(50),
  aor_created       int         NOT NULL,
  aor_updated       int         NOT NULL,
  aor_skipped_manual int        NOT NULL,
  aor_total_debit   numeric(18,2) NOT NULL,
  aor_total_credit  numeric(18,2) NOT NULL,
  aor_overwrite_manual boolean  NOT NULL
);
```

Skip it if history is not wanted; but decide, rather than lose it by default.

### 3.5 Already listed, still open

- Balance view / ledger-statement function — DECISION 8 (there are still
  zero views in the database).
- What sets `op_is_stale` — DECISION 5.
- `led_ob_amount / led_ob_type / led_ob_as_on` — DECISION 1.

---

## 4. NOT missing — exists, and the plan tells you to leave it NULL

These change the plan, not the schema. Both are on `acc_bill_balance`.

### 4.1 Link an OPENING bill to its opening row via `abl_src_*`

The plan (§4.3) says `abl_src_module / abl_src_doc_type / abl_src_doc_id /
abl_src_acc_year = NULL` for an opening bill. Use them instead:

    abl_src_module    = 'ACCOUNTS'
    abl_src_doc_type  = 'OPENING_BALANCE'
    abl_src_doc_id    = <op_id>
    abl_src_acc_year  = <op_acc_year>

`ck_abl_src_doc` only requires id and type to be set together, and
`ix_abl_src_doc (abl_src_doc_type, abl_src_doc_id)` already indexes it. Then
the bill-wise tie (DECISION 4) has an actual key to check against —
"sum of bills whose src_doc_id = this op_id" — rather than a join on
(company, year, party) that also catches bills of other types.

### 4.2 Carry-forward: chain the new OPENING bill to last year's bill

`abl_parent_bill_id / abl_parent_acc_year` exist, are FK'd across the
partitions (`fk_abl_parent_1..3`), and `ck_abl_parent` only *requires* them for
`INTEREST` — an OPENING row is free to set them. Plan §4.2 step 4 should:

    abl_parent_bill_id   = <the from-year bill's abl_id>
    abl_parent_acc_year  = <from-year>

Without it the ageing chain breaks at every year-end: the carried bill knows
its original `abl_doc_date` but not which bill it continues, so a receipt
history across the boundary cannot be reconstructed.

### 4.3 Indexes the screen needs — all present

- `ux_abl_doc_refno (company, party, bill_type, acc_year, doc_refno)` — an
  OPENING bill's reference cannot be entered twice in a year. (The same refno
  as the original SALES bill in the *previous* year is allowed — different
  year and type — which is correct.)
- `ix_abl_open (company, party, doc_date) INCLUDE (refno, amount, pending, dr_cr) WHERE pending <> 0`
  — covers the breakup panel.
- `ix_abl_party_credit (party, company, branch) INCLUDE (pending, due, dr_cr)`
  — covers the tie sum.

Nothing to add on `acc_bill_balance`.

---

## 5. If only three get done

1. §2.1 — the three FKs on `acc_opening_balance`.
2. §3.1 — two `acc_ledger_role` rows (`OPENING_DIFFERENCE`, `RETAINED_EARNINGS`). No DDL.
3. §3.3 — `ux_fy_company_year` on `fiscal_years`.

And the two plan changes in §4 cost nothing and save the ageing.

## 6. Decisions added by this file

| # | decision |
|---|---|
| 9 | `abl_branch_id` NOT NULL vs `op_branch_id` nullable — §1.1 (recommend (a)) |
| 10 | Ledger visibility in the list: global-only, or global + this company — §1.2 |
| 11 | Carry-forward run log: keep history (§3.4) or accept losing it |

Decisions 1–8 are unchanged and live in the two earlier documents.

