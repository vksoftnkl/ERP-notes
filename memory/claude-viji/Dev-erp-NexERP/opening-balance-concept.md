---
name: opening-balance-concept
description: "accounts.acc_opening_balance: what an opening balance is here, the rules read off the schema, and the screen design proposed 2026-09-12"
metadata:
  type: project
---

**Everything lives in `/home/vk/Shared/Prathap/openingbal/`** (moved there
2026-09-12, with a README.md index): the screen mockup, a table/column
diagram `opening_balance_tables.png`, the flow spec, and the backend plan. The
`.py` beside each PNG regenerates it and its output path points into the folder.
They follow the PIL house style — see [[react-port-plans]] for where the
other design artefacts sit.
**SUPERSEDED IN PART 2026-09-15: the six endpoints are LIVE and the Qt screen is BUILT — see [[opening-balance-screen-built]] for what the live API actually does, which differs from what this note assumed in four places.** (Was: table DEPLOYED and EMPTY, no endpoint, no Qt screen.) Menu 55 "Opening
Balance" already exists in `fixed.menu_master` (parent 5, position 0) and is
NOT registered in `form_accounts_menu.h`.

**Read off the schema (facts, verified 2026-09-12):**
- `op_amount >= 0` + `op_dr_cr` ('D'/'C') — positive amount, side in the flag,
  the same rule as [[sale-bill-posting-dr-cr]]. Never a negative.
- `op_source` CHECK allows exactly **CARRY_FORWARD | MANUAL | MIGRATION**.
- `op_is_stale` + `op_stale_since` + a partial index on stale rows; plus
  `op_generated_at` / `op_generated_by` SEPARATE from created/modified — the
  generator is a different actor from the operator.
- `ux_op_scope` = (company, COALESCE(branch,0-uuid), year, ledger) — one row per
  ledger per year, and a company-level opening coexists with branch ones.
- Partitioned LIST by `op_acc_year`, so the year is part of identity, not a
  filter over one set.
- `acc_bill_balance.abl_bill_type` includes **'OPENING'** — that is where a
  bill-by-bill party's opening lives.
- The shape of the live seed (test data, so indicative only — it is useful
  for sizing a mockup, never for an argument): 44 ledgers, 29 balance-sheet,
  10 income/expense, 5 with a NULL group nature, 4 bill-by-bill.

**The design proposed (NOT yet approved):**
1. It is a REVIEW surface over a derived set, not a data-entry grid. "Carry
   forward from <last year>" is the primary action; typing is the exception.
2. **Only balance-sheet ledgers are listed.** Income and expense close to the
   P&L and open at zero — an opening for 'Sales Account' is a category error,
   not a blank field.
3. **ONE PAGE (user's call, 2026-09-12): the bill-wise breakup is a DETAIL
   PANEL under the ledger list, not a modal.** A modal hides the trial balance
   at the moment the operator is changing figures that move it, and the tie a
   party must satisfy has one number on the ledger row and the other in the
   breakup. `opening_billwise_ui_mockup.*` was merged away — do not recreate it.
4. **A bill-by-bill party does not open with one number** — it opens with its
   outstanding bills (`abl_bill_type='OPENING'`). Its amount cell is read-only
   and drills through; typing one figure breaks ageing and set-off from day one.
5. A permanent **total Dr / total Cr / difference** strip; save is REFUSED when
   it does not balance, and the 'Difference in Opening Balances' plug is shown
   as a row rather than hidden in a total.
6. Stale = flagged and offered for regenerate, never silently corrected — the
   figure may already have been reported.

**BACKEND PLAN WRITTEN FOR PRATHAP 2026-09-12:**
`/home/vk/Shared/Prathap/openingbal/plan-backend-opening-balance.md` — six NestJS
endpoints (list / create / trial-balance / delete / carry-forward / bills), the
carry-forward transaction step by step, what the DB already enforces vs what
only the service can, and 7 numbered DECISIONs. Pointer added to notes.txt (3).
The user wrote "nextjs" but the backend is NestJS — plan says the contract is
unchanged if Next.js route handlers were meant.

**The carry-forward step everyone forgets** (§4.2 step 4): a bill-wise party
carries its BILLS, not just its figure — each still-open `acc_bill_balance` row
becomes a new OPENING row in the next year with the old `abl_pending_amount` as
the new `abl_bill_amount`. Skip it and carry-forward silently destroys every
debtor's ageing.

**THE FULL FLOW IS DOCUMENTED:** `/home/vk/Shared/Prathap/openingbal/opening_balance_flow.md`
— every table, column, default, generated column and CHECK read off the live DB
2026-09-12, with the three flows (MIGRATION / CARRY_FORWARD / bill-wise) written
column by column, and 7 open decisions listed. 

**The decision that blocks everything (§1 of that doc): there are TWO homes for
an opening balance.** `acc_ledger_master.led_ob_amount` / `led_ob_type` /
`led_ob_as_on` (the Tally shape — one figure per ledger, **no year**) versus
`accounts.acc_opening_balance` (per year, with provenance and staleness).

The argument is STRUCTURAL, not "which one has rows": a ledger's opening differs
per year, so the ledger-master columns cannot express 2025-26 and 2026-27 at
once, and `acc_opening_balance` can. Row counts on that box prove nothing
([[db-live-connection]]).

**CORRECTIONS found 2026-09-14 (re-read off the live DB):**
- `acc_bill_balance.abl_branch_id` is **NOT NULL**. The share docs say
  "branch or NULL" / `"branchId": null` — WRONG. A ledger opening can be
  company-level (`op_branch_id` NULL) but an OPENING bill must name a branch.
  New DECISION 9 for Prathap; the screen must either scope bill-wise entry to a
  branch or disable the breakup under "All branches".
  **All of this is now WRITTEN UP for Prathap in
  `openingbal/schema_gaps_2026-09-14.md`** (DECISIONs 9/10/11, FK/tenant/
  role/fiscal_years gaps, abl_src_* + abl_parent_* to be USED not NULLed).
  **The backend plan was REWRITTEN from scratch 2026-09-14** (nothing had
  been built, so no changelog language): §2 = schema prerequisites with DDL
  (FKs, op_tenant_id, roles OPENING_DIFFERENCE/RETAINED_EARNINGS,
  ck_acc_group_nature, fiscal_years uniqueness, acc_opening_run), §4-5 =
  contracts with branch required on bills, opId in the bills payload,
  abl_src_*/abl_parent_* set, tie keyed on abl_src_doc_id, §11 = decisions
  1-11, §12 = do-not-build, **§5.5 = editing after the first save** (asked
  2026-09-14: the upsert IS the edit; four rules — edited CARRY_FORWARD →
  MANUAL; POST /bills OWNS a bill-by-bill party's op_amount and /create only
  echoes it; a receipted bill accepts date/days/narration only; an edit to
  year N stales N+1's CARRY_FORWARD rows). The gaps file is now the audit trail, not the
  to-do list. Flow doc §4.3 corrected in place. README updated. **In notes.txt as
  2026-09-14 (5)**, which supersedes (3).
- `acc_ledger_master.led_company_id` is nullable (global vs company ledgers);
  the plan's §3.1 list join has no company filter on the ledger — needs
  `(led_company_id IS NULL OR led_company_id = :company)`.
- A THIRD balance home exists: `led_total_balance / led_total_dr / led_total_cr`
  (stamped, NOT NULL DEFAULT 0). Nothing reads them; never derive from them.
- Menu 55 has `menu_visiblity = false` — flip server-side as well as registering.
- Still true 2026-09-14: 0 rows, no route, no trigger, DECISION 1 open,
  partitions 2024_2025 / 2025_2026 / 2026_2027 exist.

**Mockup regenerated 2026-09-14** (`opening_balance_ui_mockup.py/.png`, Qt
entry-screen style like `opening_stock_entry_ui_mockup`): no company/branch/
year pickers (session), one This-branch/All-branches combo, bill panel closed
under All branches, MANUAL flip on an edited CARRY FORWARD row, frozen
receipted bill, plug via differenceLedgerId. User said "do not write code" —
the PNG is the deliverable, no Qt screen built.

**Client architecture fit (studied 2026-09-14):** it is a DOCUMENT screen
(`NexDialog` + .ui + two `NexTable`s + delegate + Cols enum), modelled on
`OpeningStockEntry` (menu 44) — not `NexCrudMasterPage` (read-only paginated
grid) and not `NexCrudForm` (one record). Row serialisation with replace
semantics + `seedLocalLayout()` fallback come from `TaxRateLedgerGrid`. Scope
comes once from `AppSession`; the only real picker is this-branch vs all-branches.
Needs two new `fixed.ui_tables` layouts and `ApiEndpoints::OpeningBalance`.

**Other verified facts worth not re-deriving:**
- `public.fiscal_years` already models carry-forward: `fy_is_carried_forward`,
  `fy_carried_forward_at`, `fy_prev_fy_id`, plus `fy_books_begin_date`,
  `fy_lock_date` and `fy_detached` (Tally's split-company). No year has ever
  been carried forward (all false).
- `ck_abl_voucher` FORCES an opening bill to be voucher-less: `'OPENING'`
  requires `abl_voucher_id` AND `abl_voucher_type_id` NULL; every other type
  requires both present.
- `abl_pending_amount` and `abl_status` are **GENERATED ALWAYS** — never send
  them. `abl_dr_cr` is `'DR'/'CR'` (two chars) while `op_dr_cr` is `'D'/'C'`.
- `av_signed_amount` is GENERATED (+DR/−CR); `ck_avh_status` is
  DRAFT|APPROVED|POSTED|CANCELLED and only POSTED is balanced.
- Voucher type 1 `OPENING` (Opening Stock) has **`vchr_affects_accounts = false`**
  — the stock opening posts nothing to the ledgers, so the Stock-in-Hand tie
  cannot derive itself today.

**Dr/Cr across the three tables (asked 2026-09-12, all verified):** the RULE is
identical everywhere — amount positive, direction in the flag (`ck_op_amount >= 0`,
`ck_abl_amount > 0`, `ck_av_amount > 0`). The SPELLING is not: `op_dr_cr` is
`char(1)` `'D'/'C'`, while `abl_dr_cr` and `av_dr_cr` are `char(2)` `'DR'/'CR'`.
`acc_vouchers` has the generated `av_signed_amount`; **`acc_opening_balance` has
no signed twin**, so every consumer applies the sign itself.

**An opening does NOT post to acc_vouchers, and the schema already decided it:**
`ck_abl_voucher` forces an OPENING bill's voucher columns to be NULL, so a
party's opening bills could not point at an opening voucher if one existed.
Technically a voucher WOULD now fit (see the correction in
[[sale-bill-posting-dr-cr]]: `av_opp_ledger_id` is NULLABLE, and `ck_avh_balanced`
only bites at POSTED) — but it would need a new voucher type (only 2 of 6 affect
accounts; no Journal type exists), an ALTER to `ck_abl_voucher`, and carry-forward
would become cancel-and-repost with nowhere to put `op_source` / `op_is_stale`.
**The serious argument FOR posting was tested and fails:** if allocations keyed
off voucher LINES, an opening bill with no voucher could never be receipted.
But `acc_bill_adjustment.abj_bill_id` is NOT NULL and FKs to `acc_bill_balance`,
while `abj_voucher_id` / `abj_voucher_line_id` are both NULLABLE —
an allocation is against the BILL, and `abj_voucher_id` is the RECEIPT that pays
it, not a property of the bill being paid. An opening bill settles like any
other. **Answer: do not post.** Pay the cost once with a balance
VIEW instead — there are currently **zero views in the whole database**, which is
exactly how a report ends up wrong by the opening balance.

**Gap the mockup exposes:** it shows a **FROM STOCK** source for 'Stock in Hand'
(the opening must equal the opening stock valuation, menu 44) — and
`ck_op_source` does not allow that value. Either add it to the CHECK or store it
as CARRY_FORWARD with the tie enforced elsewhere. Unresolved.
