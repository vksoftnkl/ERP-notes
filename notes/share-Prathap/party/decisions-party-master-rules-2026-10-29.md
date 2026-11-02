# Party master rules: user answers (2026-10-29)

Answers to §8 of `plan-party-master-rules.md`. The plan is not edited in place; this file is the record.

The user said: "go with all recommendations".

| Q | Question | Answer |
|---|---|---|
| 1 | Same delete rule for suppliers | Yes |
| 2 | Dual party: delete one role while the other keeps the ledger and its balance | Yes (the balance guard is skipped when the ledger stays alive) |
| 3 | A cheque not yet cleared blocks the delete | Yes |
| 4 | The Ledger Statement opens a deleted party, read-only ("Show deleted") | Yes |
| 5 | UNREGISTERED / OVERSEAS with a GST No is refused (old rows stay until their next save) | Yes |
| 6 | OVERSEAS = state 96 "Other Country", with no GST No | Yes |
| 7 | Company/branch on a shared party | (a): kept the same on all three rows; a company change is refused once the party has entries in its current company |
| 8 | Customer-only rights may change the supplier's party details through the sync (audited) | Yes |
| 9 | Last billed: stored, or computed when read | Computed when read (option B, §3.6); nothing stored |
| 10 | Supplier "last billed" = last posted purchase invoice | Yes, in phase 2; for now supplier saves stop writing it |

Earlier answers from the same day, which the plan already follows:
- D1: delete is refused only when the balance is not 0.
- D2: five upper-case GST types.
- D3: last billed means the real last sale bill.
- D4: customer and supplier details are synced both ways.

Related, from the customer bug batch (server, uncommitted):
- GSTIN accepts every GSTN type (normal, TDS-deductor, UIN, OIDAR, NRTP); the state-prefix check applies to NORMAL only.
- E-commerce GSTIN is checked for format only.
- cusDebitBalance may be negative.
