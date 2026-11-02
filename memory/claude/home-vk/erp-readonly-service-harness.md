---
name: erp-readonly-service-harness
description: Run ERP server services (from dist) against the dev DB on a Prisma connection forced read-only — real derive/lookups without login or HTTP
metadata:
  node_type: memory
  type: reference
  originSessionId: e05d84f3-470d-4bf0-bac8-1b311496d7c1
  modified: 2026-10-03T08:20:16.954Z
---

A way to run the server's real logic on dev data without credentials and without any chance of writing (2026-10-03).

**Steps:**
1. In a node script, require `dist/src/.../*.service.js` from `/home/vk/Dev/erp/ERP server`.
2. Build a `PrismaClient` whose URL is `DATABASE_URL` from `.env` plus `options=-c%20default_transaction_read_only%3Don`.
3. Assert `show default_transaction_read_only` returns `on` before doing anything else.
4. Construct the services by hand. Pass a request-context stub `{ getUserId: () => <a user who holds the menu right> }`. Pass `null` for collaborators that the read path never touches.

For example, `VoucherRegisterService.validate` and `get` need only prisma, the context and `VoucherTypesService`.

**User ids:** a null user gets NO_RIGHTS. Find a real user in `public.user_menus` (columns `um_user_id`, `um_menu_id`, `um_can_view`, …).

**Script on disk:** `derive-harness.cjs` in the session scratchpad, e5d84f3… session.

**Related facts:**
- The client dev server is `next-server` on **https**://localhost:3001; plain http returns an empty reply.
- `:3011` and `:3000` are ERP server processes.
- Local curl to them needs the sandbox disabled.
- Table names:
  - `accounts.acc_ledger_master`, `accounts.acc_tender_master`, `accounts.acc_cheque_book`, `accounts.acc_bill_balance`
  - `fixed.ui_tables` / `fixed.ui_table_columns` (`ui_tbl_clm_*`)
  - `fixed.menu_master` (`menu_parent`)

See also [[erp-server-http-testing-without-credentials]] (the full HTTP stack under jest).
