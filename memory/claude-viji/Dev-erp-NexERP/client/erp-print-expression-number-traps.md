---
name: erp-print-expression-number-traps
description: Print-template expressions - coalesce returns a string (so + concatenates) and node-pg NUMERIC arrives as a string (so a bare truthiness `visible` never hides a zero row)
metadata:
  type: project
---

Two number traps in print template expressions (jexl), both verified 2026-09-15
against the real engine:

- `|coalesce(0)` returns a **string**. `{{ a|coalesce(0) + b|coalesce(0) }}`
  concatenates ("10" + "0" = "100"). Use `|num` for arithmetic — it coerces and
  reads null as 0 — and parenthesise before a transform, because `a + b|fmt()`
  formats `b` alone: `{{ (a|num + b|num)|fmt('##,##,##0.00') }}`.
- `PgService` only overrides the DATE parser, so node-pg hands **NUMERIC back as
  a string**. `visible: "{{ header.cess_amt }}"` is therefore truthy on "0.00"
  and the row prints anyway. Guard with `{{ header.cess_amt|num != 0 }}`.
  QUOTATIONA5 still carries the bare form.

**Why:** both fail silently — a wrong total or a zero row on a customer's
invoice, with no warning from the layout engine.

**How to apply:** when writing or reviewing a definition, run it through the
server's `ExpressionValidator` (parse + root whitelist) and `LayoutEngine` with
real rows, feeding numerics BOTH as numbers and as strings — psql `json_agg`
gives numbers and hides the second trap. See [[erp-print-template-contract]],
[[erp-print-designer-ui]].
