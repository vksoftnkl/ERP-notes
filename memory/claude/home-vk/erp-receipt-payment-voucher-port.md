---
name: erp-receipt-payment-voucher-port
description: "2026-10-03 Receipt Voucher (RcpV, menu 260) and Payment Voucher (PmtV, 261) ported to React on the Contra voucher screen — instruments, bill-wise popup, TDS, post-dated cheques; uncommitted"
metadata:
  node_type: memory
  type: project
  originSessionId: e05d84f3-470d-4bf0-bac8-1b311496d7c1
  modified: 2026-10-03T08:20:10.188Z
---

Built 2026-10-03 in `/home/vk/Dev/erp/ERP client`, UNCOMMITTED, by extending `features/accounts/vouchers/` (see [[erp-contra-voucher-register-port]]).

- **New domain modules:** `domain/instruments.ts`, `domain/bills.ts`, `domain/generated.ts` (`readPosted`).
- **New components:** `components/instrument-dialog.tsx`, `billwise-dialog.tsx`, `voucher-extras.tsx`, plus `vouchers.module.scss`.
- **Pages and routes:**
  - `receipt-voucher-page.tsx` and `payment-voucher-page.tsx`.
  - Routes `/accounts/receipt-voucher` and `/accounts/payment-voucher`, each with a keyed `[company]/[branch]/[year]/[id]` route.
  - Menu labels "Receipt Voucher" and "Payment Voucher".
  - Registry entry: ui table 40 `voucherBills`.
- **Issued Cheques:** Enter on a PmtV cheque now opens the CARRIER voucher by exact keys: `voucherId` plus `voucherAccYear`. A post-dated child voucher links back to its original.

**Contract facts learned (all verified on the dev DB):**
- `/get` does not return `employeeIds` or leg `lineRowNo`.
- `av_row_no` is renumbered per voucher, so instruments are matched by `instruments[].lineRowNo` and party.
- A cancelled voucher's `/get` has no instruments; its money legs come back `generated: false`.
- `/create` stores the body wholesale, so `allocations` are always sent.
- PmtV TDS lines: the operator keys the net, and the party leg is the gross. The bill-wise popup settles the gross taken from `/validate`.
- An overridable WARN (`VCH_TDS_BELOW_THRESHOLD`) blocks post unless it is sent in `overrides` and the user has the override right.

**Deliberate departures from Qt:**
- Lines that carry an instrument are left out of the party-first balance.
- A cheque tender's ledger is not offered as a paying bank.
- The instrument dialog has a "No instrument" button.
- The TDS cell has three states: ledger default / Yes / No.

**Verified without login:**
- Bodies checked against the compiled DTOs.
- The server's own `validate` run on client-built bodies, and `get` read back through `readPosted`, using [[erp-readonly-service-harness]].
- Render smoke tests, routes return 200, tsc clean, 3103 tests pass.

**Not done:** never clicked through logged-in.
