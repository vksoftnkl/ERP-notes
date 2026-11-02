---
name: ledger-map-screen-plan
description: "acc_ledger_map has no API and no screen — the plans (share/ledgermap/), the shared-set-only scope decision, and why the role list must become a route"
metadata:
  type: project
---

**Plans written 2026-09-18: `/home/vk/Shared/Prathap/ledgermap/`** —
`plan-backend-ledger-map.md` + `plan-qt-ledger-map.md`, sent as notes (35).
Client to be built in parallel with Prathap's backend.

**The state that justified it.** `accounts.acc_ledger_map` turns every posting
ROLE into a ledger. 28 rows, every role mapped, nothing broken — but **zero
API routes of any kind**, so the only way to change a mapping is SQL on the
box. A wrong mapping does not fail; it posts money to the wrong account
silently. That is the whole case for the screen, and it is worth stating that
way rather than as "completeness".

**Scope: the SHARED set only** (user, 2026-09-18). One ledger per role, same
for every company; `alm_company_id` / `alm_branch_id` stay NULL. The
per-company override `ux_alm_role` allows stays in the schema, unbuilt —
and the API must **400 on `almCompanyId`** so nothing starts depending on a
half-built one. `alm_supply_nature` (INTRA/INTER for the GST roles) likewise
carried, never set.

**Two non-obvious things in the backend plan:**
- **No guard exists** against mapping a role to a ledger that cannot hold it —
  `DISCOUNT_ALLOWED` pointed at a bank is accepted today. The plan carries the
  role-family → ledger-type table (§2.1). Catching it when typed beats catching
  it at the first posting, because a mapping is set once and lived with.
- **`GET /ledger-map/roles` should own the role list**, not the client.
  `ReceiptRoles` in `receipt_entity.h` is a second hand-maintained copy and the
  two have ALREADY disagreed: `ROUND_OFF` is in the map and in the client, and
  `/receipts/post` rejects it ([[posted-amend-decision]] neighbours; the round-off
  gap is notes 34). Pointing the receipt's Type combo at the served list is a
  follow-up, not part of the screen, but it is why `/roles` is a route.

**Grids 104 / 105 / 106** (POSTING ROLES, LEDGERS FOR ROLE, SETUP - POSTING
LEDGER MAP) were registered long ago for exactly this and are unused. **Check
they have live columns before trusting them** — a grid can carry SQL and zero
live columns, answer 200 and render nothing ([[nextable-grid-traps]]).
