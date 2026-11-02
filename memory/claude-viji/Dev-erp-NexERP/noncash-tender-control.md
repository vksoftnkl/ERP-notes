---
name: noncash-tender-control
description: "Card/UPI/wallet control plan REV 1 (2026-10-07) — share till/plan-noncash-tender-control.md + 49_noncash_tender_control.sql, validated on throwaway PG, NOT applied"
metadata:
  node_type: memory
  type: project
  originSessionId: 95fb9728-f266-4303-ad2e-7585b7adda08
  modified: 2026-10-07T08:15:10.713Z
---

Asked 2026-10-07 after [[till-management-design]]: cash is counted, but how do you control card / UPI gaps — slip by slip or by machine?
Answer = by machine, in three layers:
1. Reference at billing: `tnd_needs_ref`, auth + last4 or UTR, duplicate refused, `td_settle_status` PENDING.
2. One batch total per terminal at close; slip check only on a mismatch (S5 panel).
3. Next-day statement matching (REF > AUTH > AMOUNT_TIME suggested); one TSet voucher per payout.

**Approvals and treatment:**
- A non-cash gap is SUSPENSE at close, never EXPENSE.
- Two exception lists: not received → NONCASH_WRITE_OFF; money with no bill → SETTLEMENT_RESOLVE.

**Key facts:**
- Matching runs on the STORE server. `acc_tender_detail` rows are store-written, and a cloud write of `td_settle_*` would be overwritten by the next push.
- Two new tables, justified: `acc_settlement_import` (one payout), `acc_settlement_line` (provider line, can be money with no bill).
- New column `tnd_statement_format` jsonb on the tender master.
- On the box: one CARD + one UPI tender per company, no settlement ledger, card/UPI rows at NA. Split per terminal at go-live.
- 49 restates 48's CHECK lists (reason / event / approval).

**Validation:** rebuild the throwaway cluster from `pg_dump --schema-only` of live, plus data for tender/voucher/seq/role/settings/companys/branch_master/device/menu, loaded with `session_replication_role=replica`. 16 probes pass. The snap Firefox render trick for PNGs: stage in $HOME, not /tmp.

**Open (user):** which acquirer/PSP (sample files), static vs dynamic QR, RECOVER vs LOSS default.

**Mockups 2026-10-07 (not yet reviewed):** `till/till_settlement_ui_mockups.py` writes match / exceptions / import PNGs. Run it without `-I`: it imports the local `till_mock`.
- The story (review rev 2): HDFC payout 08-10, 21 lines, net 17,889.32. The card gap is −160 SUSPENSE on T2 (bil00611, slip-check mockup).
- The 500 with no bill = a UPI-QR2 payment = SURESH B's −500 C02 cash shortage, paired via `asl_tvr_id`. C02's session ran 09:10–15:30, so every T2 card time sits inside it.
- LINKED on a CLOSED session = pairing only: no voucher, no reversal (plan-till-review-rev2 §2.1). The REVERSED+RETENDERED idea was wrong.
