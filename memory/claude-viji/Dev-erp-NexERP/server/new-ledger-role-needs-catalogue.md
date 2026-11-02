---
name: new-ledger-role-needs-catalogue
description: A new acc_ledger_role row must also go into fn_ledger_map_catalogue() or Acc_Ledger_Map.sql aborts every boot; that seed COMMITs its own txn
metadata:
  type: project
---

Twice now (20260921220000 sales roles → fixed 20260922060000; 20260925160000 voucher-register roles → fixed 20260926163000 on 2026-09-26) a migration inserted into `accounts.acc_ledger_role` without a `CREATE OR REPLACE` of `accounts.fn_ledger_map_catalogue()`. `fn_seed_ledger_map()` then refuses ("These active roles are not in accounts.fn_ledger_map_catalogue()") and the boot seed stops re-asserting the whole map. The app still boots and /health stays green, so this only shows up in the pm2 error log.

**Why:** the seed treats an uncatalogued role as a silent posting-time null and aborts on purpose.

**How to apply:** after a deploy, grep the pm2 err log for `[Seed] ✖`. Do not dry-run `prisma/seed/Acc_Ledger_Map.sql` inside `BEGIN … ROLLBACK`: the file has its own BEGIN/COMMIT and commits the outer transaction. See [[vps-deploy-topology]] and [[fn-ledger-for-is-typescript]].
