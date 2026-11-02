---
name: fn-ledger-for-is-typescript
description: accounts.fn_ledger_for is NOT a plpgsql function — the role->ledger resolver is ledger-map.helper.ts; the .prisma comments that name it are stale
metadata:
  type: project
---

DECIDED 2026-09-12: posting-configuration logic lives in the application layer,
never in plpgsql. Both halves now exist as TypeScript:

- write side — `src/modules/accountsModule/ledgerRole/ledger-role.helper.ts`
  ("may this ledger be mapped to this role?")
- read side — `src/modules/accountsModule/ledgerRole/ledger-map.helper.ts`
  (`resolveRoleLedgers` / `requireRoleLedgers`: tax_rate_ledger override first,
  then acc_ledger_map, branch > company > global, nature-specific > neutral)

**Why:** a plpgsql guard/resolver answers with a RAISE that Prisma surfaces as
an opaque 500, one failure at a time. The TS form returns a 400 with a field
path per bad line.

**How to apply:** several comments still promise SQL that was never written and
will not be — `accountLedgerMap.prisma` ("fn_ledger_for already prefers it"),
`accountLedgerRole.prisma` ("via accounts.fn_check_role_ledger"), the
`20260911070000_add_acc_ledger_role` header (cites a migration
`20260911071000_add_fn_check_role_ledger` that does not exist), and
`20260912070000_add_chg_tax_id` ("lives in the trigger below" — there is no
trigger). The retire-GST-columns SQL drafted on 2026-09-12 likewise cites
`public.fn_charge_tax_id()` and a `public.vw_charge_tax_setup` view, neither of
which exists — its §4 had to be rewritten to test charge_master and the ledger
inline. Do not go looking for those functions in the database, and do not write
them; extend the helpers instead.
