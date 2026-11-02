---
name: ledger-tds-pair
description: The ledger's Deductee Type / Nature of Payment are rate-lookup keys, now coded combos — and the INTERIM static section list waiting on accounts.tds_rates
metadata:
  type: project
---

Settled 2026-09-19. `led_tds_deductee_type` and `led_tds_nature_of_payment` on
acc_ledger_master are **join keys** into `accounts.tds_rates`
(section × deductee × date → rate, payment/plan-backend-payment.md §2.9), not
prose. They were `F::text`; they are now `F::staticComboCoded` writing the bare
code (`COMPANY`, `194C`). Free text matched no rate row, so a payment deducted
nothing and said nothing — the failure had no symptom.

**Why:** the column widths gave it away — ledger varchar(40)/(80) vs
tdr_ varchar(16)/(16) vs atd_ varchar(20)/(20). A value that fit the ledger
could not be stamped into the register.

**How to apply:** the Deductee Type list is CLOSED (the six of ck_tdr_deductee:
COMPANY/INDIVIDUAL/HUF/FIRM/AOP/ANY) and stays static. The section list is
**INTERIM static** — five seeded sections in `kTdsSections` in
ledger_entry.cpp — and must become `F::apiDropdown` the day a
`/tds-rates/sections` route (or a dropdown id) exists, because a Finance Act
adds sections and a hard-coded list costs a client build to follow one. Same
argument as the served role list, [[ledger-map-screen-built]].

Also in: the deductee type defaults from the **4th character of the PAN**
(C company, P individual, H HUF, F firm, A/B AOP; T/G/L/J left blank) and fills
only when unset — a default, never an override. Both fields are disabled until
TDS Applicable is ticked and required once it is (extraValidate; the DB has no
CHECK and the box carries one ledger with the flag on and both blank).

**Open on the backend: notes (37)** — accounts.tds_rates does not exist on the
box at all, the sections route, and the CHECK + width fix. Tests: §5b of
tests/ledger/run.sh, 13 checks, see [[ledger-group-tests]].
