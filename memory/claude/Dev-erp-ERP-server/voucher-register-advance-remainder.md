---
name: voucher-register-advance-remainder
description: "notes (57) 2026-09-28: on RcpV/PmtV (DEMAND + money side) a party leg allocated short keeps leg − Σ allocations as ONE ADVANCE bill on the party (leg's side, dated the voucher, no due date); over-allocation still VCH_BILLWISE_SHORT; setting accounts.voucher_allow_advance=false restores the strict rule"
metadata:
  type: project
---

Built 2026-09-28 in voucher-derive.ts (DEMAND branch of the party-leg loop): the remainder
is pushed onto `bills` as `{ billType: 'ADVANCE', isAdvance: true, dueDate: null }`, so the
post path raises it through `raiseBill` (narration "On account from voucher …", refno = the
voucher's, `/<lineRowNo>` suffix only for a second advance on the same party) and the cancel
path soft-deletes it with the raised bills — now for EVERY voucher of the set, because an
advance left on a post-dated cheque line is raised on that cheque's own carrier voucher.
The cancel guard `otherVoucherOnRaisedBills` also runs per voucher of the set. On a TDS
payment the leg is the gross, so the advance = gross − allocated (the TDS is the leg's).
`derived.bills[]` shows it before Post (no separate `advanceLines`). Setting
`accounts.voucher_allow_advance` (BOOL, COMPANY, default true, migration 20260928180000).

**Why:** the Qt register's bill-wise popup leaves a remainder the way menu 99 does; the strict
DEMAND rule forced a second voucher.

**How to apply:** a test that wants the strict refusal must pass `allowAdvance: false`; the
http e2e "short" probes are now `validate` calls asserting `derived.bills`. A Journal on
/vouchers with an allocation against the ADVANCE bill pairs ADVANCE_ADJUST rows and then
blocks the receipt's cancel (VCH_ALLOCATED_ELSEWHERE) until it is released.
