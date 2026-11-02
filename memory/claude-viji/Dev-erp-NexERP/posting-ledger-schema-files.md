---
name: posting-ledger-schema-files
description: "Files 19-24 in the share: the posting-role/ledger-map/tax-rate redesign, their forced run order, and the one blocker still open"
metadata:
  type: project
---

The accounting base agreed before go-live, in `/home/vk/Shared/Prathap`
(see [[db-schema-share-location]]). Run order is **forced by FKs**, not taste:

    19_posting_roles.sql       accounts.acc_ledger_role + fn_check_role_ledger()
    20_tax_rate_master.sql     inventory.tax_rate_master (24 cols, NO ledger
                               columns) + tax_rate_ledger (the overrides, as rows)
    21_accounts_posting_setup  sbi/soi/sqi_tax_id, acc_ledger_map, resolvers,
                               §1.2b led_tax_id + chg_tax_id + charge guard
    → then the client + NestJS release
    22_retire_ledger_gst_cols  CONTRACT phase: drops led_gst_rate, led_tax_rate,
                               led_taxability. Refuses until nothing needs them.
    23_ledger_party_fields     independent; party reg-type case fix + CHECKs + SEZ
    24_tax_rate_ui_registration  the GST rate screen's dropdowns/grids as SQL

**19 and 20 are DEPLOYED** as of 2026-09-11 (20 roles, 11 rates, tax_rate_ledger
empty). 21-24 were not yet run; 24's contents were registered live via the API
instead — see [[gst-rate-master-screen]].

**Expand → migrate → contract**, because the DB and NestJS ship separately and
the Qt client is a desktop install that does not update everywhere at once.
21 adds and backfills, drops nothing; 22 holds every DROP.

**The shape everything follows** (one fact, one home, NULL = inherit):

    inventory.tax_rate_ledger   overrides   accounts.acc_ledger_map
    charge_master.chg_tax_id    overrides   acc_ledger_master.led_tax_id
    item_master.item_default_tax_id  freezes onto  sale_bill_item.sbi_tax_id

**Kept, against first instinct** — checked by vocabulary, not row count:
`led_gst_duty_head` (one 18% rate → five duty heads; `alr_want_duty` validates
against it), `led_hsn_sac` (no HSN on a rate row), `led_gst_party_type`
(Deemed Export / Embassy·UN Body exist in no other column).

**STILL OPEN 2026-09-11:** §7f of file 21 — 20 posting ledgers carry
`led_company_id` = Acme Foods, so every role resolves to nothing. The API
**cannot** fix it — confirmed structurally 2026-09-12: the create DTO declares
`ledCompanyId` as a non-nullable uuid string, and there is no update route — (POST /account-ledger-masters/create with `ledCompanyId: null`
answers "updated successfully", moves led_modified_on, and leaves the column —
the service drops nulls on merge). Must be run as SQL; reported run twice, never
landed on 192.168.0.106/ERP (no trigger, no rule, no default — it simply did not
execute there; the host has 7 databases). §7g adds NULLS NOT DISTINCT to the two
unique indexes, also not run.
