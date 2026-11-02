---
name: erp-sale-bill-quick-add-customer
description: "Sale bill (menu 12) Quick-add Customer dialog — port of Qt CustomerQuickAddDialog (2026-10-08); endpoint-name collision with testbill, required state name, Enter-walk helper"
metadata:
  node_type: memory
  type: project
  originSessionId: d02db684-da9b-4a50-9a0a-ae2c5672e8a3
  modified: 2026-10-08T06:06:45.403Z
---

Built 2026-10-08: the quick strip's "+" on `features/sales/salebill` opens
`components/customer-quick-add.tsx` (rules in `salebill.quick-add.ts` + test), a
port of Qt `src/modules/sales/common/customer_quick_add_dialog.cpp`. Before this
the button only focused the customer box and hinted Alt+C (which opens the FULL
customer master — that shortcut still exists separately).

Non-obvious bits:
- The RTK endpoint is `createQuickCustomer` in `saleBillApi`, NOT `quickAddCustomer`:
  `features/sales/testbill/api/bills.ts` injects `quickAddCustomer` with
  `overrideExisting: true`, so reusing the name lets the two replace each other.
- `cusStateName` is `@IsNotEmpty` server-side. A state code filled in from a GSTIN
  (Qt `onGstinEdited`) has no name until it is looked up on dropdown 21 by code,
  so the save refuses a code without a name rather than 400ing.
- `cusGstType` goes UPPER-case (REGULAR/COMPOSITION/UNREGISTERED): Qt sends Title
  case, but the React customer master's select stores and expects upper-case.
- New id is routed through the entry view's `onRequestCustomer`, so the §4.3
  "change the customer?" confirm and customer-detail/party-context reads run as
  for an operator pick (Qt: `confirmCustomerChange` → setSelection → fetches).
- **Customer Template applies (user ask, same day):** `quickAddTemplateFrom` reads
  `masters.customer_form_defaults` off the session settings slice
  (`selectAppSettingText` — same scope as the master's `/effective` read). Seeds
  GST type/area/group/state (company state when the template names none — no TN
  fallback); sends a FIXED allow-list of the master's payload keys (credit terms,
  flags, charges, region, price level…) typed like the master's save. Fixed list
  because the create is `forbidNonWhitelisted`; all 53 keys validated against
  `SaveCustomerDto`. Area/group stored without a `*Name` are dropped (uuid in box).
- **Enter-walk trap:** the generic `enterMovesToNextField`
  (`features/stocks/selling-price/components/enter-navigation.ts`) walks "next
  focusable element"; a quotation `DropdownCombo` opens its menu on focus, so the
  next focusables are its OPTION BUTTONS — Enter lands on row 1 and the next Enter
  picks it over the stored value. Use an explicit id walk in any dialog that holds
  DropdownCombos. `DropdownCombo` also gained `required`.
- Known limitation (shared ModalShell): Esc while a combo menu is open closes the
  whole dialog — ModalShell's document-capture Escape listener runs first.

Verified via the fake-API CDP recipe ([[erp-client-ui-browser-verification]]) and
the captured body validated against the server's `SaveCustomerDto`.
Related: [[erp-sale-bill-lifecycle-port]], [[erp-testbill-module]].
