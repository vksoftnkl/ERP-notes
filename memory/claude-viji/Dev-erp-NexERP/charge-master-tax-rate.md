---
name: charge-master-tax-rate
description: Charge master (ChargeEntry) chgTaxId per-charge tax-rate override via dropdown 36 + chgTaxName on edit; live only when chgTaxApl && !chgBeforeTax (ck_chg_tax_id); null cost-alloc loads as NONE
metadata:
  type: project
---

Added 2026-09-25. The server's SaveChargeMasterDto takes `chgTaxId` (an
inventory.tax_rate_master id). It overrides the posting ledger's ledTaxId for
that one charge, and null inherits the ledger's rate. GET returns `chgTaxId` +
`chgTaxName`. The DB CHECK ck_chg_tax_id requires null unless chgTaxApl=true and
chgBeforeTax=false.

The Qt form (src/modules/admin/master/charge/charge_entry.cpp) has an
apiDropdown "Tax Rate" on dropdown 36 with display key chgTaxName, so the NAME
fills on edit through the engine. syncTaxRate() greys and clears it unless Tax
Applicable is on and Before Tax is off. beforeSubmit sends an explicit null
(never drops the key), so turning tax off clears the stored rate.

**Why the fillFromJson override exists:** save maps NONE → null for
ck_chg_cost_alloc, and the edit loaded null as a BLANK required combo, so every
re-save of a non-landing charge failed with "Choose the cost allocation"
(a pre-existing bug). It now loads null as NONE.

Test: `tests/salebill/run-full.sh chargeform` creates a ZT charge, then checks
the edit prefill, the untick → NULL, and the delete. A form the harness drives
must be set as g_screen, or the modal-closer accepts and deletes it.
