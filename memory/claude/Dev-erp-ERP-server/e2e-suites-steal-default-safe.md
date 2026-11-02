---
name: e2e-suites-steal-default-safe
description: tender-settlement-http and expense-voucher-http e2e create a tsfIsDefault safe, which un-defaults the live ZTS safe; teardown never restores it
metadata:
  type: project
---

`test/tender-settlement-http.e2e-spec.ts` and `test/expense-voucher-http.e2e-spec.ts` create their safe
with `tsfIsDefault: true`; `till/safes/create` then clears the branch's old default (ux_tsf_default).
Their afterAll deletes their own safe but leaves the user's test-box safe ZTS (Acme / Coimbatore,
01a11f22-4681-7d5a-8c86-0e6954e86927) with `tsf_is_default = false` — back-office cash then has no
default safe (TILL_SAFE_MISSING).

**Why:** seen 2026-10-09 while verifying notes 99.

**How to apply:** snapshot `accounts.till_safe` defaults before running either suite and restore
afterwards (`UPDATE … SET tsf_is_default = true WHERE tsf_id = … AND NOT EXISTS (another live
default in the branch)`). See [[till-management-schema]], [[pre-existing-test-failures]].
