---
name: voucher-totals-trigger-missing
description: FIXED 2026-09-17 by migration 20260917170000 — the voucher-totals trigger is now live on localhost; the columns are DERIVED and must never be written
metadata: 
  node_type: memory
  type: project
  originSessionId: 99e184ae-6cfd-4d68-9fd9-e701b9c28596
  modified: 2026-09-17T14:16:42.077Z
---

**RESOLVED.** `accounts.tr_av_refresh_totals` now **exists** on localhost/ERP,
installed 2026-09-17 by
`20260917170000_install_missing_voucher_totals_trigger`, which carries §11 of
`20260915120000` verbatim so there is one definition of the arithmetic.

Verified after applying: the trigger and both functions
(`fn_avh_refresh_totals`, `fn_avh_recompute_totals`) are present, all 39
affected headers back-filled from 0/0 to their true figures, and **zero POSTED
headers unbalanced**. Probed on `rct00033` in rolled-back transactions —
retiring the CR legs alone is refused by `ck_avh_balanced`, as is tampering
with one leg's amount.

**How to apply now:** `avh_total_debit` / `avh_total_credit` are **DERIVED**.
Never write them from application code — the trigger overwrites the value on
the next leg write, and a stamped value is what made `ck_avh_balanced` compare
a writer's claim with itself. Read them freely; derive with
`deriveVoucherTotals()` (`accountVoucherHeader/voucher-totals.helper.ts`) only
when you need to refuse an imbalance with a message that names the voucher,
which is what the posting service does before flipping a header to POSTED.

## One thing the trigger still does NOT catch

Postgres queues AFTER-ROW triggers to end-of-statement, so a single `UPDATE`
that retires **every** leg of a voucher leaves the totals at 0/0 — which
balances, and is accepted while POSTED. Only a **one-sided** subset is refused.
Measured both ways. So do not rely on the constraint alone if you ever retire
legs; cancel by writing a reversal voucher, which is what
[[stock-ledger-cancel-leaves-forward-row]] argues for elsewhere.

## Why it was missing for two days — the reusable lesson

§11 was **appended to `20260915120000` after that migration had already been
applied**. Prisma records a migration as applied and will never run text added
to it afterwards, so the database never saw it while the file claimed it had.
`20260915130000` lines 17-24 and 123-130 document this. It is NOT a
partial-execution bug — do not go looking for one.

**Never edit a migration that has been applied anywhere.** The added text runs
on fresh databases and silently never runs on existing ones: the file says the
object exists, every environment that ran it before the edit disagrees, and
nothing reports the difference. The delta belongs in a new migration.

**How to detect it:** `prisma migrate status` will not tell you. Compare each
file's SHA-256 against `_prisma_migrations.checksum` — drift means the file was
edited after apply. An audit of all 342 recorded migrations on 2026-09-17 found
**five** drifted: `20260905060000_add_stock_track_preset`,
`20260912050000_add_line_tax_id`,
`20260912110000_repoint_items_to_tax_rate_master`,
`20260915090000_opening_balance_schema_prerequisites` and this one. Of those,
only two lost objects — this one, and see
[[fk-cd-tax-and-ix-cd-tax-missing]].

Related: [[prisma-migration-gotchas]], [[prisma-schema-is-generated]]
