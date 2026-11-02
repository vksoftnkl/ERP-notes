---
name: receipt-payment-menu-rights
description: Since notes 62 (2026-09-29) every /receipts and /payments route checks user_menus on menu 99 / 100 — live users without um_can_post lose posting; e2e suites must grant tester1
metadata:
  node_type: memory
  type: project
  originSessionId: 56955dac-749e-4490-8721-ee15a5da424c
  modified: 2026-09-29T05:25:59.630Z
---

notes (62) D2 put `assertMenuRight` (src/common/posting/rights.ts) at the top of every route in
receipt.controller.ts (menu 99, codes RCT_RIGHT_*) and payment.controller.ts (menu 100, PMT_RIGHT_*).
Mapping: reads → view; /create → create (new) or edit (existing id); /update-header → edit; post,
cancel, delete, amend → their own flag. `/receipts/regularise-pdc` is deliberately NOT gated
(cron sweep).

**Why it matters:** there is no SUPER ADMIN bypass. On dev on 2026-09-29 only VKPOS and VKPOS1 held
post/cancel/amend on menu 99; ravi and VIJAY held view/create/edit/delete only; tester1 and
prathap held nothing. Uncommitted as of that date, and the user was told to grant rights before
deploying.

**How to apply:** a 403 RCT_/PMT_RIGHT_* is a missing user_menus flag, not a bug. Every receipt and
payment e2e suite grants tester1 the flags via test/helpers/menu-rights.ts (`grantMenuRights` /
`restoreMenuRights`, or `PaymentFixtures.grantRights`) and must run `--runInBand`. A new suite
hitting those routes needs the same. See [[sales-e2e-specs-rights-fixture]], [[payment-module]].
