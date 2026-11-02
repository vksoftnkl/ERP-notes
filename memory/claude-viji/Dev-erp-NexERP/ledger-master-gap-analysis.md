---
name: ledger-master-gap-analysis
description: "2026-09-17 audit of acc_group_master + acc_ledger_master vs their DTOs, forms and the Tally export target — what is missing for a complete GST ERP"
metadata: 
  node_type: memory
  type: project
  originSessionId: 710f791a-1472-4ec1-9cd3-20c6a06e4ac6
  modified: 2026-09-17T12:35:38.408Z
---

Three-way diff done 2026-09-17: table columns ↔ save/GET DTOs ↔ what the two
forms expose. The end goal stated by the user is **exporting the finished ERP
into Tally**, so every gap is judged against that too. Handed to Prathap as
`/home/vk/Shared/Prathap/ledger_group_changes.md`.

**CLOSED the same day (2026-09-17) — 1.3, 1.4, 2.1, 2.2, 3.1.** The five group
flags are in both DTOs and on the form; `accGroupChildIds` is gone from the GET
AND the table; `led_itc_eligibility` (CHECK, 5 values) and
`led_is_reverse_charge` are live and on the ledger form; the six balance columns
are out of the ledger save DTO and now 400 if sent.

**RE-VERIFIED 2026-09-18** (`tests/ledger/run.sh` — all checks pass, exit 0;
plus live DB + `/api/docs-json`). Newly closed since the audit:
- **1.1 is CLOSED.** `fk_cus_ledger` and `fk_sup_ledger` both exist on
  `sales.customers.cus_id` / `purchase.suppliers.sup_id` -> `acc_ledger_master
  (led_id)` ON UPDATE CASCADE ON DELETE RESTRICT, and the orphan count is now
  **0 customers / 0 suppliers** (was 2 + 2) — the legacy rows were backfilled.
- **The `ledLedgerType` 500 is FIXED** (see [[acc-group-master-contract]]):
  an invalid type now returns a 400 listing the nine allowed values.

Still open, confirmed 2026-09-18: 1.2 (one collision left,
`VAPI BATTERY & AUTO ELECTRICALS` x2), 1.5 (no cost-centre table at all),
1.6 (no currency table), 2.3, 2.4 (no `tds_rates`/`tcs_rates`/`statutory_*`
on the box), 2.5 (ledger has only `led_rounding_limit`; credit limit still on
`sales.customers`), 2.6 (grid exists, read-only — the ledger DTO still has no
`ledgerShipAddress[]`), 3.2-3.5. `led_category` is still blank on 7 of 65 live
ledgers. Tally sync: 1 of 108 ledgers and **0 of 113 groups** have a guid.

**The enum tightening bit us.** Prathap also made the combo-backed columns
strict, so the server stopped normalising `''` → NULL. An untouched
NexComboBox sits at `currentIndex() == -1` and `collectJson()` was writing `""`,
so a ledger create 400d on four enums at once. Fixed in the ENGINE: an invalid
ApiCombo/StaticCombo now submits `QJsonValue::Null`, matching what ApiDropdown
already did. Expect the same class of break on any other form whose endpoint
gets tightened.

**Already designed for, all still EMPTY.** Tally sync columns exist on
`acc_group_master` (4), `acc_ledger_master` (5), `acc_voucher_header`,
`item_master`, `item_unit_master`, `fiscal_years`, plus
`acc_voucher_types.vchr_tally_{export_enabled,voucher_type_name,base_voucher_type}`.
0 of 61 ledgers has a guid, master id or tally name. Nothing writes them.

**Blocks the export**

1. `sales.customers.cus_id` / `purchase.suppliers.sup_id` are supposed to BE
   `led_id` — **no FK**. 2 of 12 customers and 2 of 6 suppliers (all created
   before June 2026, when auto-creation was added) have no ledger row at all.
2. Name collisions across company scopes — see [[company-scope-nullable]].
3. The four Tally group flags are in **neither DTO and on no form**:
   `acc_group_behave_as_subledger` (ISSUBLEDGER), `acc_group_net_debit_credit`,
   `acc_group_used_for_calculation`, `acc_group_affects_gross_profit`. The whole
   group save DTO is 7 keys. `acc_group_is_active` is in neither DTO and the
   form's checkbox is commented out, so a group can never be deactivated.
4. `acc_group_child_ids` is stale (Customers says 1 child, has 11; 21 rows NULL)
   and the GET marks it REQUIRED. Walk `parent_id`; ignore or drop the column.
5. **No cost centre master**, though `led_is_cost_center_req` is a form checkbox
   and `acc_vouchers.av_cost_centre_id` is a column with no FK and 0 of 104 rows
   populated. Three-way dead end.
6. No currency master — no multi-currency or forex.

**Blocks GST regardless of Tally**

7. **No ITC eligibility flag anywhere** (grepped `itc|input_credit|eligib`;
   `cus_itcoll_*` is income-tax TCS, not ITC). GSTR-3B 4(D) cannot be produced.
8. **No reverse-charge flag on a party or expense ledger** — only
   `tax_rate_master.tax_is_reverse_charge` and `gdr_is_reverse_charge`.
9. No "applicable from" date on GST details; Tally stores them as a dated list.
10. TDS/TCS is typed, not computed: `led_tds_deductee_type` and
    `led_tds_nature_of_payment` are free text, and **no `tds_rates`/`tcs_rates`/
    `statutory_*` table exists on the box** (see [[statutory-masters-model]] —
    designed in the share, not deployed).
11. Credit limit lives on `sales.customers`/`purchase.suppliers`, not the
    ledger, so `acc_bill_balance` (which does ageing) cannot see it.
12. `acc_ship_addrs` + `/ledger-shipping-addresses/*` (3 routes) have **no
    screen**; it carries `saa_gstin`, `saa_state_code`, `saa_distance_km`.

**Form-level**

- `led_category` is free text but the column is NOT NULL DEFAULT 'GENERAL' and
  REQUIRED in the GET — 7 rows already hold `''`.
- `ledAllowEdit`/`ledIsEntry` are in the DTO and appear nowhere in the client.
- The form submits every HIDDEN field too (a bank ledger posts
  `ledGstPartyRegType`); the server normalises `''` → NULL, so it works by
  leaning on that.
- The group list (grid 53) is flat; groups are a tree and an export must emit
  parents first.
- No root group can be created — `accGroupParentId` is required.

Checked and NOT a bug: `NexComboBox::selectByText` is case-insensitive, so the
server upper-casing `led_gst_party_reg_type` to `UNREGISTERED` still selects the
form's "Unregistered".
