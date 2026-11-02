---
name: erp-supplier-qt-parity
description: "Supplier master brought to the Qt Supplier Entry 2026-10-05 — Qt option words, GSTIN rules, \"Also a Customer\" (notes 81 party link), plus two shared-shell fixes; open follow-ups (collection-day off-by-one on customer screens, reverse \"Also a Supplier\")"
metadata:
  node_type: memory
  type: project
  originSessionId: 59591584-91d1-44dd-ac7a-c056d6399d2d
  modified: 2026-10-05T13:22:34.152Z
---

Done 2026-10-05 against `NexERP/src/modules/purchase/master/supplier` (Qt).

- `sup_gst_type` / `sup_purchase_type` are free text. The supplier form now saves the Qt
  words: GST Type Regular/Composition/SEZ/Overseas/Unregistered (default Regular),
  Purchase Type LOCAL/IMPORT (default LOCAL). Old web rows (REGULAR…, "Goods Supplier") load
  case-insensitively / as an extra option. SEZ/Overseas are safe for suppliers because
  `supGstType` never reaches the ledger (unlike the customer).
- Collection days: Qt `SupCollectionDaysWidget` writes 0=Mon…6=Sun. Supplier uses its own
  0-based options now. **Still off by one:** Customer and Customer Group (Qt uses the same
  0-based widget) and Area use the shared 1-based `COLLECTION_DAY_OPTIONS` in utils/constant.
- "Also a Customer": `features/masters/shared/party-role-link.ts` (ledger → form prefill),
  `suppliers/use-also-customer.tsx` (probe `/customers/get` 404 + customer canCreate),
  `sales/customer/linked-customer-entry.tsx` + `CustomerPage` prop `linkLedger` (name locked,
  `cusLinkLedId` on create). **Not built:** the reverse "Also a Supplier" on the customer
  form (needs a supplier-side `linkLedger` + modal-only mount).

**Shared fixes made along the way (why stacked forms now behave):**
- `CrudMasterPage.openCreate({values})` MERGES over `createInitialValues`; it used to
  replace them, dropping Active and the customer template (also hit Alt+C typed-name create).
- `ERPDynamicModalForm` window keys (Escape, Ctrl+S/F12, ArrowDown) answer only for the
  topmost open modal (module-level stack) and Escape is `preventDefault`ed — before, Escape
  closed both stacked forms and Ctrl+S saved the one underneath too. See
  [[erp-nested-dialog-escape-race]].

Verified in a headless browser with the fake-API recipe in [[erp-client-ui-browser-verification]].
Related: [[erp-company-branch-qt-parity]], [[erp-master-modal-legacy-tab-layout]].
