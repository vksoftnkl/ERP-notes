---
name: crud-blank-number-dropped
description: NexCrudField::blankAs() — what a CLEARED Number/Integer field sends; all 35 master-form number fields annotated from the column + DTO; any NEW number field needs it too
metadata:
  type: project
---

**The bug:** NexCrudForm left a cleared Number/Integer field OUT of the body,
and an update only touches the keys it receives, so clearing a value on edit
kept the old one. The user hit it on the customer credit limit (232 stayed 232).

**The fix (2026-09-25):** `NexCrudField::blankAs(QJsonValue)`. The engine
sends that value when the field is blank, and a field without it still omits
the key. All 35 F::number / F::intField fields across the 14 NexCrudForm
masters are annotated by a mechanical rule:
- the column is NOT NULL, or the DTO is not nullable → the column default
  (0, except tndConversionRate → 1);
- otherwise → `.blankAs(QJsonValue())`, i.e. null ("not set").
The two mismatches: compEwayIntraLimit (DTO nullable, column NOT NULL) → 0;
ledSortOrder (column nullable, DTO not nullable) → 0.
The scan script is /tmp/claude-1000/numscan.py: camelCase key → snake column →
information_schema, plus the Swagger spec `nullable`.

**How to apply:** give every NEW number field on a NexCrudForm a `.blankAs`,
chosen the same way. The rule does NOT cover own-.ui masters (item master,
etc.): they collect numbers in their own code. Verified live with harness
`custlimit` (limit → 0, distance → NULL); the crudform and ledger suites pass.

Related: [[shared-payload-per-dto-keys]], [[charge-master-tax-rate]]

**The item master is NOT affected (checked 2026-09-25):** its header numbers are
spin boxes (always sent), and the unit / price / reorder grids send
`cellText().toDouble()`, so a cleared cell goes out as 0. Every grid column is
NOT NULL DEFAULT 0 except iuc_unit_factor and iuc_to_base_factor (DEFAULT 1).
A blank or 0 unit factor is refused at save (item_entry.cpp:454), and the
to-base factor is computed, never typed. Live proof: harness `itemclear` sets
Min Price 5, then clears it, and the DB shows 0 (it gives ZT-ITEM-A a brand and
category, because the item master requires both).
