# accounts — the accounting scope, and the Voucher Register

> **2026-09-25 — the Voucher Register now lives in ONE file: `voucher_register.md`**
> (model, decisions A–E, types, worked examples, the `39_voucher_register.sql`
> migration, the `/vouchers/*` routes, the posting routine, lifecycle, the Qt
> client, tests) with `voucher_register_ui_mockup_rev2.png` as its screen.
> Nothing in this folder has been run. The earlier rev 1 set (a separate
> spec, backend plan, Qt plan and two PNGs) was deleted on 2026-09-25.

Two things live here: the **scope** of accounting as a whole, and the first
form built from it.

**Read in this order:**

| file | what it is |
|---|---|
| `accounting_scope.md` | **start here** — every form a complete accounting system needs, measured against TallyPrime and GoFrugal (3.0 is a reference column only), what we have, and what is left: 15 MUST forms + 12 MUST reports. §2 is the voucher-register model, §2.5 why day close reads the LEDGER and not tenders, §6.1 the day-close form, §9 decisions S1–S20, §10 the build order in four waves |
| `voucher_register.md` | **the Voucher Register, whole design** — model, decisions A–E, types, worked examples, migration, routes, posting routine, lifecycle, Qt client, tests |
| `voucher_register_ui_mockup_rev2.png` | **rev 2 (2026-09-25), approved — the screen for `voucher_register.md`** — one register + ONE posting routine that Receipt/Payment/goods docs also call; decisions A (no/several parties → party column nullable), B (GST band generates the tax legs), C (cancel = the live Rev reversal, own series, dated the original), D (menus 101–104 open it with the type fixed), E (rights per voucher TYPE via its own menu; band filtered, server checks every call), TDS leg in scope |
| `statutory_masters.md` | **reference, NOT a work order — read its §0 first** (only two small things are worth doing today; the rest waits for a client to ask). **Where Indian law lives** — the four layers Tally and GoFrugal both use (applicability · the shipped statutory pack · per-party certificates · the stamp on the document), why law must never sit in `app_setting_def`, DDL for `statutory_limits`, `tcs_rates`, `acc_party_tax_certificate` and `statutory_pack`, effective dates for `tax_rate_master`, the seed list with section references, and the resolution order that decides a rate (L1–L14) |

**One screen, eight voucher types:** Journal · Contra · Debit Note · Credit
Note · Purchase (accounting) · Sales (accounting) · Receipt and Payment in
voucher mode. The type is the only variation and it lives in
`accounts.acc_voucher_types` — **if a rule ends up in a `switch` in the client,
the design has failed.**

**"Sales" and "Purchase" here are not goods documents.** No item, no batch, no
godown, no stock. That is what makes them the home for every expense, service,
fixed asset and RCM entry — and why NexERP needs no separate GST-expense
screen.

**Prerequisite, and it is not optional:** `../receipt/plan-backend-receipt.md`
§2.4 — `avh_total_debit` / `avh_total_credit` must become DERIVED by trigger.
They are stamped today, so `ck_avh_balanced` cannot catch an unbalanced
voucher, and in a free-leg register that check is the only guardrail there is.
Asked as note (4); not run.

**Approved on the mockups before these documents were written:** S13 party
cardinality · S14 rule columns on the type · S15 the GST band in
`acc_voucher_doc_detail` · S16 the five guards · S17 one canonical payload ·
S21 GST note vs financial note. **Still open and the user's call: S20** — one
cash ledger per branch, or per counter/drawer.

**Out by decision:** tenders on a register voucher (day close reads the
ledger) · items and stock · multi-currency · cost centres (S1) · reversing and
memorandum types · recurring entries · e-invoice API calls (the `gdr_` row is
written, the call belongs to the e-invoice screen).
