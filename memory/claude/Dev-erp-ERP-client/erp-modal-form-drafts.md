---
name: erp-modal-form-drafts
description: "Unsaved master-popup entries survive close/navigation/reload via opt-in `modalDraftKey` on CrudMasterPage (lib/form-drafts, localStorage per user); Customer + 17 simple masters on since 2026-10-16; the open popup also reopens on return"
metadata:
  node_type: memory
  type: project
  originSessionId: 1068c083-d622-4c54-bd79-c60a5bd22a16
  modified: 2026-10-13T12:31:11.070Z
---

Built 2026-10-13 because the user lost a half-typed Customer entry when the popup closed (stray backdrop click / Escape / going off to Sale Bill or User Admin and coming back).

- `lib/form-drafts.ts`: localStorage `erp_form_draft:v1:<userId>:<key>`, keeps only fields that differ from the values the form OPENED on, plus dropdown option names (`labels[field][value]`); 7-day expiry.
- `ERPDynamicModalForm`: `openModal(variant, { draftKey })` restores, writes on every change, any non-save close KEEPS it (Cancel too — deliberate choice), save or the "Discard" button in the amber notice bar drops it. `onDraftApplied` fires AFTER `onOpenChange` (the page's create-reset runs first).
- `CrudMasterPage` `modalDraftKey="x"` → `x:new` / `x:edit:<id>`; a create opened WITH override values (dropdown Alt+C typed name, ledger link) neither restores nor keeps a draft. `onModalDraftApplied` lets a page re-sync its own state.
- **Why opt-in:** screens that derive payload data from page state (Customer's `stateNameByCode` → `cusStateName`, pinned lazy-dropdown options) need an `onModalDraftApplied` hook, or a restored id saves without its name. Check each screen before switching it on. Customer's hook = `applyDraftDropdownSelections` (additive seeding so Discard can go back to the template/record seed).
- Browser-test gotcha: global Title-Case capitalization ([[erp-global-uppercase-inputs]]) rewrites typed text, so compare case-insensitively. Driver recipe: [[erp-client-ui-browser-verification]]; the customer toolbar button text is exactly "Add".
- **2026-10-16:** switched on for 17 masters with no derived state (charge, unit, ledger bank/shipping, supplier/customer groups, area, freight/loading charges, state, tender, designation, department, GSP provider/service, widget master, item section). CrudMasterPage now also REOPENS the popup that was open when the operator left the screen — see [[erp-screen-memory]]. Masters with lazy-dropdown/derived state (city, item brand/category/group, ledger groups, device list) were opted in later the same day with `onModalDraftApplied` re-syncs; Employee stays OFF by the user's choice (PAN/Aadhaar must not sit in localStorage); tax rate, suppliers, godown, item master not done.
