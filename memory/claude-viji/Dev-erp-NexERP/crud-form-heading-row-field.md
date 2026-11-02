---
name: crud-form-heading-row-field
description: NexCrudField::inHeadingRow() — put a one-field section's control ON its heading row instead of a labelled row below it
metadata:
  type: reference
---

Added to the NexCrudForm engine 2026-09-17 (`src/shared/ui/nex_crud_form.*`).

`F::apiDropdown(...).inHeadingRow()` renders the control **on the preceding
folded heading's row** and registers **no label** — the heading is the label:

```
Tax Rate  [ GST 18%            v ] ─────────────────────
```

instead of a "Tax Rate" heading with a "Tax" labelled row under it, which reads
as two things when it is one. Use it for a section that holds exactly one
control and must stay its own section so `setSectionVisible()` can toggle it.

Mechanics worth knowing:
- The heading row is a `QHBoxLayout` of `[caption][rule(stretch 1)]`. The
  control is inserted **before the rule, with stretch 1**, so the two split the
  leftover width 50/50 — without that the control sits at its bare
  `controlMinWidth` and looks narrower than the ordinary fields above it.
- The control's **parent is the heading widget**, not the card grid, so
  `setSectionVisible()` hides it either way.
- `m_labels` has no entry for it; `setFieldVisible()` already copes with a null
  label.
- Ignored (falls back to an ordinary row) when no folded heading precedes it,
  or when the last heading opened a tab — a tab header has no row to fold into.

First user: the ledger form's tax picker, [[ledger-tax-id-change]].
