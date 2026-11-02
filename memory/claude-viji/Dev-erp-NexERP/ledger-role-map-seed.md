---
name: ledger-role-map-seed
description: "accounts.acc_ledger_map was empty and blocks every posting engine; 26_ledger_map_seed.sql fills all 22 roles, and the §7f blocker is gone"
metadata:
  type: project
---

**Found 2026-09-15 while wiring the opening-balance screen** ([[opening-balance-screen-built]]):
`accounts.acc_ledger_role` has **22 roles**, `accounts.acc_ledger_map` had
**ZERO rows**, and there is **no API route** to write one (checked all 308).
So `accounts.fn_ledger_for()` returns NULL for every role — every posting
engine that resolves a ledger by role resolves nothing. Not just opening
balances: `19_posting_roles.sql`, `20_tax_rate_master.sql`,
`21_accounts_posting_setup.sql`, `24_tax_rate_ui_registration.sql` and the
receipt / payment / cheques / register plans all depend on it.

**The §7f blocker is GONE** — the twenty posting ledgers ARE global
(`led_company_id IS NULL`) now, so `21_…` §5's ten-role seed would fire today.
It has simply never been run. Supersedes what [[posting-ledger-schema-files]]
records as blocking.

**Written: `/home/vk/Shared/Prathap/26_ledger_map_seed.sql`** — all 22 roles,
supersedes §5 (run one or the other, not both). It creates the **twelve
ledgers eight roles have no home for**: Purchase Account, Purchase Return,
Input CGST/SGST/IGST/Cess/State Cess, Output State Cess, Difference in Opening
Balances (Suspense A/c), Retained Earnings (Reserves & Surplus), TCS/TDS
Payable. Re-runnable; dry-run end to end on a throwaway PG 18.1 cluster loaded
with the live schema + data — 12 ledgers + 22 map rows, second run 0 + 0.

**`19_posting_roles.sql` defines only TWENTY roles** — no OPENING_DIFFERENCE,
no RETAINED_EARNINGS, though both are live (sort 250/260), added straight to
the DB. So 26_ seeds all 22 itself (§1b, ON CONFLICT DO NOTHING) and stands
alone; 19_ should still get the two rows.

**Facts that shaped it, all verified:**
- **Neither the guard trigger NOR the validator is deployed.** 19_ defines
  `fn_ledger_role_check`; it is not in `pg_proc`, and there is no trigger on
  `acc_ledger_map` or `tax_rate_ledger`. 19_ was applied in PART — table and
  rows yes, function no. Worth checking 20_/21_ the same way. `21_…` talks about "the guard"
  enforcing `alr_want_type` / `alr_want_duty` / `alr_want_nature`; nothing is
  deployed (checked `pg_trigger`). So the seed checks the fit itself and
  ABORTS on a mismatch. The trigger is still owed.
- `(led_company_id, led_name)` is a PLAIN unique, so **NULL company means
  duplicate GLOBAL names are allowed** — `ON CONFLICT` has no target and would
  silently duplicate on a re-run. Guard with `NOT EXISTS`. (Five ledgers are
  already called 'opening stock' for this reason.)
- `led_ledger_type` CHECK: PARTY|BANK|CASH|TAX|ROUNDOFF|DISCOUNT|EXPENSE|
  INCOME|GENERAL. `led_gst_duty_head` CHECK: Central Tax|State Tax|Integrated
  Tax|Cess|State Cess.
- `acc_ledger_master` needs only `led_group_id` + `led_name`; everything else
  defaults.
- **`INPUT_ACESS` / `OUTPUT_ACESS` are NOT typos** — `19_posting_roles.sql`
  defines them as State Cess, deliberately distinct from CESS. Nearly reported
  as a bug.

**How to dry-run a schema change on this box** (no write access to the live
DB — the `claude` role is read-only): `pg_dump --schema-only -t …` plus
`--data-only` from 192.168.0.106, then `initdb` a local PG 18 cluster and load
it. **The socket dir must be short** (`mktemp -d /tmp/pgsockXXXX`) — the
scratchpad path exceeds the 107-byte limit.

**SEEDED ON THE LIVE SERVER 2026-09-15**: `acc_ledger_map` now has **22 rows,
no unmapped active role**, and the chart is 56 ledgers. The opening-balance
plug resolves — `/list` returns `differenceLedgerName: "Difference in Opening
Balances"` where it answered null all day. A plain-INSERT variant without the
PL/pgSQL guards is `acc_ledger_map_insert.sql` beside it (statement A creates
the twelve ledgers, B maps all 22; B alone maps only 10 and silently leaves
twelve roles unmapped).

**Still open:** a Posting Ledgers screen (it can build itself from
`alr_group` / `alr_sort_order`). Until then the .sql IS the configuration.
