---
name: sale-bill-posting-dr-cr
description: "The sale bill → accounts posting: write order forced by CHECKs, which amount is DR vs CR, and the live-DB blockers"
metadata: 
  node_type: memory
  type: project
  originSessionId: a00e4b3d-7fec-4c75-a81a-54925d7dd66f
  modified: 2026-09-11T06:45:10.996Z
---

Diagram: `/home/vk/Shared/Prathap/sale_bill_posting_flow.png` (+ `.py`, PIL house style).
Built from the LIVE schema 2026-09-11, not from `vouchers.sql` — the two have diverged.

**The rule:** every amount column is stored POSITIVE (`av_amount > 0`,
`abl_bill_amount > 0` are CHECK-enforced). Direction lives in the `dr_cr` flag
alone. `av_signed_amount` is GENERATED (+DR / −CR). Never post a negative.

**DR** party/debtor `sb_bill_amt` (asset up). **CR** Sales `sb_taxable_amt`,
Output CGST/SGST/IGST/Cess, after-tax ADD charges, round-off when > 0.
**DR** side also takes cash discount, DEDUCT charges, round-off when < 0.
Do NOT post `sb_item_disc`/`spl`/`sch`/`bill_sch`/`addl1`/`addl2` — already
netted inside `sb_taxable_amt`.

**The before-tax trap:** `sb_taxable_amt = net goods + Σ(before-tax ADD) −
Σ(before-tax DEDUCT)`. If you post a before-tax charge to its own ledger you must
subtract it from the Sales credit or it is credited twice. A before-tax charge is
taxed at the ITEM's rate, so never post `cd_*_amt` for one — only after-tax
`cd_tax_apl` charges carry their own GST legs.

**Write order is forced, not preference:** bill → items → tender →
voucher_header (DRAFT) → acc_vouchers legs → POST → `acc_bill_balance`
(`ck_abl_voucher` makes `abl_voucher_id` NOT NULL for every non-OPENING bill) →
adjustments + PDC. `ck_avh_balanced` only bites at POSTED, so build as DRAFT.

**CREDIT tender (type 9) posts no leg at all** — the party DR just stays open.
PDC: `apd_posting_mode` ON_RECEIPT posts now, ON_CLEARING posts nothing until
clearing. An adjustment is never a tender — see [[party-adjustment-design]].

**Done 2026-09-11:** the 20 posting ledgers now exist on **Acme Foods Pvt Ltd**
(created via `POST /api/v1/account-ledger-masters/create`, login VKPOS), and
`accounts.acc_ledger_map` is specified in `/home/vk/Shared/Prathap/21_ledger_map.sql`
— role→ledger, company-scoped, branch-nullable, with a trigger that refuses to map
a role at the wrong kind of ledger, plus `fn_ledger_for_strict()` which RAISEs on an
unmapped role. NOT yet applied: the `claude` DB role has no CREATE, so Prathap runs it.
The map deliberately excludes tender and charge ledgers — `acc_tender_master.tnd_ledger_id`
and `charge_master.chg_ledger_code` already own those.

**Revenue account determination:** `22_item_income_ledger.sql` (share) adds
`inventory.item_income_ledger` — item OR group override, company-scoped, resolving
item → nearest ancestor group → `acc_ledger_map('SALES')`. Argued from the principle, not the seed (the 10,034 items on the
test box are not evidence — [[db-live-connection]]): items spanning several GST
rates all credit one Sales ledger, and every comparable ERP
(Tally stock-item allocation, Odoo product/category, ERPNext Item Defaults, SAP
VKOA) resolves revenue at item level. Not applied — Prathap runs it.
**Gotcha found doing this:** `item_group_master.itg_path_ids_cache` and `itg_level`
are UNMAINTAINED — nothing recomputes them, which showed up as wrong ancestor
paths across the seed. Walk `itg_parent_id` instead; that tree is derived from a
real column rather than a cache nobody refreshes. (The specific counts came off
the test box, so treat them as a symptom, not a measurement.) Also: item groups are GLOBAL (no company column) while ledgers are
company-scoped, which is why the override is a table and not a column.

**THE MODEL IS ONE SHARED CHART OF ACCOUNTS** (user decision, 2026-09-11). A
posting ledger is GLOBAL (`led_company_id IS NULL`) and serves every company;
company separation lives on `av_company_id` on the voucher rows, not on separate
ledgers. The pre-existing ledgers were already global — I wrongly created 20 new
ones scoped to Acme, which manufactured a fake "one cell, many companies" problem
and an unnecessary `item_tax_ledger` child table. Both dropped.

**So `item_tax_master`'s twelve ledger columns are the RIGHT home** — a rate is
global, its ledger is global, one value, one place. They only needed the twelve
FKs and a guard trigger, both now in `20_tax_rate_master.sql` + `21_accounts_posting_setup.sql`. The guard's
key rule: a tax row's ledger **must be global**, because a shared rate pointing at
a company-scoped ledger posts everyone's revenue into one company's books.
`acc_ledger_map.alm_company_id` is likewise NULLABLE (NULL = all companies),
resolving most-specific-first: branch → company → global.

**`item_tax_master` → `inventory.tax_rate_master`** (20_tax_rate_master.sql,
clean-sheet, not migrated — old data was test data). CGST/SGST/IGST are GENERATED
from one `tax_rate_perc`; the six purchase-rate columns are gone (they never once
differed). Seeded with 7 statutory slabs + ZERO_RATED/EXEMPT/NIL_RATED/NON_GST.
Second cess added (`tax_acess_*`, OUTPUT_ACESS/INPUT_ACESS roles) because
`sbi_acess_*`, `gdr_state_cess_value` and duty head 'State Cess' all expected one.

**LEDGER RESOLUTION IS TWO-LEVEL AND TWO-AXIS. Every column is OPTIONAL.**
`fn_tax_ledger_for(company, branch, tax_id, role, is_inter)`:
  1. `tax_rate_master`'s column — the per-rate override (18 ledger columns)
  2. `accounts.acc_ledger_map` — the single default, and the ONLY home for
     ROUND_OFF / DISCOUNT_ALLOWED / WRITE_OFF / ADVANCE_RECEIVED
Set nothing and you get one Sales + one Output CGST. That is a complete config.

Two independent splits are supported, and they compose:
  · **by rate** — 'Sales @ 18%', and 'Output CGST @ 2.5%' if the chart keeps
    output tax rate-wise (common Tally/VAT practice)
  · **by supply nature** — 'Local Sales' vs 'Interstate Sales', via
    `acc_ledger_map.alm_supply_nature` (NULL=both, INTRA/INTER; a scope
    dimension like company/branch, NOT a doubled role list) and the rate's
    `tax_*_inter_ledger_id` columns.
Precedence: branch › company › global, then nature-specific › both.

**MISTAKE TO NOT REPEAT:** I argued the tax-ledger columns were redundant because
"they'd hold the same uuid on every rate row" and dropped 10 of them. That was
only true given a chart *I* chose. Rate-wise output tax is legitimate and common;
the user caught it. Don't state a preference as a structural constraint — check
what the chart of accounts might legitimately want first.

**Still open (2026-09-11):** ~~`av_opp_ledger_id` is NOT NULL but a sales
voucher is multi-leg with no single opposite~~ — **WRONG, corrected 2026-09-12
against the live schema: `av_opp_ledger_id` IS NULLABLE**, and `ck_av_self` only
bites when it is set (`opp IS NULL OR opp <> av_ledger_id`). A leg with no single
opposite is legal, so this was never a blocker. Still open:
`led_is_bill_by_bill` false on every party; `av_bill_id` and `accounts.acc_bills`
don't exist live (linkage runs only via `abl_voucher_line_id`); no posting
function is deployed — see [[stock-engine-not-deployed]] for the same
spec-vs-NestJS split.

**Correction 2026-09-11:** the "no Sales/GST ledgers exist" blocker is stale —
they were created and 19/20 are deployed. The live blocker is now that those
ledgers are company-scoped; see [[posting-ledger-schema-files]] §7f. Also note
the CR amount is `sbi_net_gross` summed, NOT `sb_taxable_amt`, which includes
before-tax charges that post to their own ledgers.
