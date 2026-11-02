---
name: printing-ui-screens
description: "Front-end decomposition for the printing module — one Designer screen, a Templates list, and why assignment must be its own branch-scoped screen"
metadata:
  node_type: memory
  type: project
---

Screen split for `17_printing.sql`, settled 2026-08-27. Mockup:
`/home/vk/Shared/Prathap/printing_ui_mockup.py` → `.png` (7 windows).

- **Print Templates (list)** → **Print Designer** (ONE screen, tabs Template / Layout / Data,
  with the version rail inside the Template tab). `print_template_version` is deliberately
  NOT its own screen: it cascades from its template, has no identity outside it, and the
  publish/rollback action is an UPDATE on the *template* row.
- **Create is two steps, edit is one.** "New template" is a 4-field dialog (code, name,
  purpose, owner) → saves `print_template` + rev 1 DRAFT → opens the Designer. Rationale:
  `ux_ptl_code` fails fast on a duplicate code before anyone lays out a page, and clone
  shares the same dialog (the only place `ptl_forked_from_id`/`_rev` are written).
- **Print Assignments is its OWN screen, scoped to a branch** — not a Designer tab. A
  per-design tab structurally *cannot* show the right answer: an override lives on a
  DIFFERENT template's row, so opening design A shows a company-wide row and implies every
  counter uses it. Screen shape: branch/counter/output selectors, purpose rows × three
  scope columns + an "Effective here" column. Also the only place a purpose with **nothing
  configured** is visible.
- Printer Profiles and Print History (read-only) are their own screens. Menus 66–70.

**The UI is the enforcement — there are no triggers in the file.** "A published version is
never UPDATEd" and "`ptd_dataset_no` is immutable once published" are prose rules only:
- A PUBLISHED revision must be read-only on screen; the only affordance is **New draft**
  (which must copy the `ptd_` rows too, or the draft renders nothing).
- Reordering the dataset list writes `ptd_sort_order`, **never** `ptd_dataset_no`.
- The Designer's save writes **identity columns only**. `ptl_published_rev_id` belongs to
  Publish/Roll back alone — a blind whole-row UPDATE silently reverts a publish, and
  `print_template` has no `_row_version` to catch it.
- `ptl_published_rev_id IS NULL` (draft only, reaches no counter) must be loud in the list.

See [[printing-schema-design]], [[widget-signal-wiring-rule]], [[grid-delegate-edit-pattern]].
