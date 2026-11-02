---
name: erp-master-modal-legacy-tab-layout
description: "How master entry modals are laid out to match the legacy desktop screens (heading=tab, subheading=group, span helper, denseGrid off)"
metadata: 
  node_type: memory
  type: project
  originSessionId: ea564b0e-4213-4be4-a412-f1a1625ba989
  modified: 2026-08-08T13:18:39.707Z
---

When a master's entry modal has to match the legacy desktop screen (Qt "… Entry" dialog: tab strip + label-left/control-right pairs), the house pattern is:

- `type: "heading"` field ⇒ one **tab** (with `modalSectionNavigationMode="tabs"`); its `sectionGridColumns` sets that tab's column count. `type: "subheading"` ⇒ an in-tab group band (Address / Contact / Terms …), full width.
- Use a local `const span = (n) => ({ fieldStyle: { gridColumn: `span ${n}` } })`; a 12-col section with `span(6)` gives the legacy two-pairs-per-row look. Full-width fields use `colSpan: 2` (renders as `1 / -1`).
- **`modalFormDenseGrid={false}` is mandatory** with subheadings, or fields back-fill above them into the wrong group — see [[erp-modal-dense-grid-heading-bleed]].
- Pin BOTH `width` and `height` in the panel style (sized to the tallest tab) so the modal keeps one size across tabs; each tab body scrolls.
- Ported this way: Customer (`features/masters/sales/customer/page.tsx`, plus a `customer-modal.module.scss` maroon skin) and Supplier (`features/masters/purchase/suppliers/fields-schema.ts`, tabs Identity / Notes / Regional Details, bank-accounts grid living under Notes → "Bank Details", 2026-08-08).

Verify with the headless browser recipe in [[erp-client-ui-browser-verification]] — the layout bugs here are visual, not type errors.
