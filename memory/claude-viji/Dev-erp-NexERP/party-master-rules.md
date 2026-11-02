---
name: party-master-rules
description: "Party master rules plan (customer/supplier delete, dual party, GST types, last billed) decided 2026-10-29; P1 migrations applied, rest of the server build was running"
metadata:
  type: project
---

Plan `/home/vk/Shared/Prathap/party/plan-party-master-rules.md` (never edited); user answers in `party/decisions-party-master-rules-2026-10-29.md` ("go with all recommendations"); status `party/status-party-master-rules-2026-10-29.md`.

- Applied 10-29 17:32: `20261029110000_ledger_name_unique_skip_deleted` (name + Tally name unique among live rows only), `20261029120000_party_master_rules_p1` (ledger GST type + SEZ flag, five GST type codes REGULAR/COMPOSITION/SEZ/OVERSEAS/UNREGISTERED, state 96 Other Country, `ix_sb_cust_posted`, one-time reset of `cus_billed_*` / `sup_billed_date`). Customer bug batch + these migrations were committed by the user 10-29 13:04 (fafefb4b, 012860b7, 5eded5e5).
- Decisions: delete refused only when balance is not 0 (D1); dual party may drop one role while the ledger lives (Q2); un-cleared cheque blocks delete (Q3); Ledger Statement opens a deleted party read-only (Q4); UNREGISTERED/OVERSEAS with a GST No refused, old rows stay until next save (Q5); OVERSEAS = state 96, no GST No (Q6); last billed computed on read, never stored (Q9 = B); supplier last billed from last posted purchase invoice in phase 2 (Q10).
- The ck_cus_gst_type / ck_sup_gst_type CHECKs and dropping the `*_billed_*` columns are phase 2.

**Why:** customer and supplier are one ledger (see [[party-dual-role]]); stored "last billed" values only ever came from master saves, so they were wrong.

**How to apply:** do not touch the customer/supplier save path until the running server build reports. GST type rules apply at the next save of an old row. Related: [[dto-own-undefined-trap]], [[customer-creates-ledger]].
