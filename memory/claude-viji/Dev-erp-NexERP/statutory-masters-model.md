---
name: statutory-masters-model
description: "REFERENCE not a work order (notes 17): where Indian law lives (share/accounts/statutory_masters.md, 2026-09-15): the four-layer Tally/GoFrugal model — applicability / shipped statutory pack / per-party certificates / the stamp — plus statutory_limits, tcs_rates, acc_party_tax_certificate, statutory_pack and tax_rate_master effective dates"
metadata:
  type: project
---

**File:** `/home/vk/Shared/Prathap/accounts/statutory_masters.md`. Notes (16).
Written because the user said *"do how tally/gofrugal operates this im blind
about this so your vision is my erp now for accounts"* — so this is the house
model for statutory data. **But it is a MAP, NOT A WORK ORDER** (user, same
day: *"may be today we dont need this option but in future client must need
this"*). Notes (17) scopes it: **do not build** `statutory_limits`,
`tcs_rates`, `acc_party_tax_certificate`, `statutory_pack` or
`statutory_due_dates` until a client asks — all are consulted at entry time, so
adding one later changes nothing already posted.

**Only TWO things are worth doing now, because they get expensive later —
and BOTH ARE NOW SPECIFIED (notes 18):**
(1) **stamp WHY a rate applied** — `tdsRateSource` = MASTER | CERTIFICATE |
NO_PAN | BELOW_THRESHOLD, one column in tables already being created; without
it a posted deduction can never be re-explained, and 26Q cannot report a
certificate deduction made before the layer existed. **This column is what
keeps everything else optional.** (2) ~~effective dates on `tax_rate_master`~~ — **DOWNGRADED TO OPTIONAL
2026-09-15, notes (19)**: the user corrected the premise — *"govt wont change
tax completed from 5 to 7, it only happens by items — for that we have item tax
history table"*. The GST **slabs are stable**; what a notification moves is
**which slab an ITEM sits in**, and that edge is already dated by
`inventory.item_tax_history` (`ith_effective_from/_to`, indexed on item+dates,
FK to tax_rate_master since 25 ran). `share/27_tax_rate_effective_dates.sql`
exists and is harmless but nothing depends on it.
**THE REAL GAP: `item_tax_history` has 0 ROWS — nothing writes it** when
`item_default_tax_id` changes. Needs a trigger on `item_master` (new row with
`ith_effective_from` = the NOTIFICATION's date, not now(); close the previous
row). Open behaviour decision: does the sale bill resolve an item's tax through
that history as at the document date, or take the current slab? Only matters
for back-dated documents and audit answers — reprints/GSTR are safe either way
because the bill stamps its own rate. `tdsRateSource` (item 1) is unaffected
and went into receipt plan §2.14 and payment plan §2.9/§4.1.

**How Tally does it, and we copy it:** a **Statutory Masters version** ("stat
file") updated INDEPENDENTLY of the product version; F11 as the company's
applicability switchboard; TDS Nature of Payment as a real master (section,
rate by deductee, exemption limit, effective from); and a **§197 certificate on
a party that BEATS the master rate**. GoFrugal: numbered configs for choices,
effective-dated tax masters, HQ pushes masters to outlets.

**FOUR LAYERS — every parameter belongs to exactly one:**
- **L0 applicability** — client's choice; `comp_tds_applicable` /
  `comp_tcs_applicable` already exist. Missing: e-invoice/e-way applicability
  with its own effective date (belongs with `11_gst_provider.sql`).
- **L1 the statutory pack** — WE ship it, effective-dated: `accounts.tds_rates`
  (done), `accounts.tcs_rates` (NEW — its own table because **206CC no-PAN is
  1 %, not 20 %**), `public.statutory_limits` (NEW — one table for every scalar
  constant, with `stl_enforcement` WARN/REFUSE/INFO as DATA).
- **L2 party override** — `accounts.acc_party_tax_certificate` (NEW): §197 /
  206C(9), rate (0 = nil), **ceiling amount**, validity, EXCLUDE on overlap.
  **We had missed this layer entirely; Tally has it.**
- **L3 the stamp** — the document keeps what was applied; already true.

**RESOLUTION ORDER (the point of the layers):** applicability → **certificate**
→ master → **no-PAN, which OUTRANKS the certificate (206AA)** → stamp section,
rate AND **why** (`tdsRateSource` = MASTER | CERTIFICATE | NO_PAN |
BELOW_THRESHOLD — 26Q reports a certificate deduction with its number).
Getting that order wrong is the classic TDS defect.

**LAW NEVER GOES IN `app_setting_def`** — it has no temporal columns, so
editing a rate there silently rewrites the past. Settings are for choices
(`tcs_basis`, `receipt_bill_sort`).

**Four `statutory_limits` seeds belong to screens being built now:** 269ST
2,00,000 (cash receipt per person per day → the Receipt screen REFUSES; no such
check exists today), 40A(3) 10,000 / 35,000 transport (Payment warns), MSME 45
days 43B(h) (supplier due date + payables ageing — `led_msme_type` /
`led_udyam_no` exist with nothing reading them), e-invoice turnover 5 cr.

**`public.statutory_pack`** = Tally's stat version in our schema; every
statutory migration inserts a row so support can ask "which pack are you on?".
**`inventory.tax_rate_master` needs `tax_effective_from/_to`** — it has
`tax_supersedes_id`, which chains versions without saying WHEN each applied
(11 rows, 21 inbound FKs → expand/migrate/contract).

Every seeded number is **the shape, not the authority** — check against the
current Finance Act before loading. Open for the user: L7 (269ST refuse vs
warn), L8 (packs as migrations), L14 (where e-invoice applicability lives).
See [[tcs-billwise-design]], [[accounting-scope-register]].
