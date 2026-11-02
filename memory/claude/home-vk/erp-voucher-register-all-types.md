---
name: erp-voucher-register-all-types
description: "2026-10-03 React voucher screen now serves every register type — Journal, Sales/Purchase (Accounting), Debit/Credit Note, and the all-types Voucher Register (262); uncommitted, never run logged-in"
metadata:
  node_type: memory
  type: project
  originSessionId: e05d84f3-470d-4bf0-bac8-1b311496d7c1
  modified: 2026-10-03T08:47:45.178Z
---

**Built 2026-10-03, UNCOMMITTED,** in `/home/vk/Dev/erp/ERP client/features/accounts/vouchers/`. It extends [[erp-contra-voucher-register-port]] and [[erp-receipt-payment-voucher-port]].

**Pages and routes:**
- New pages: journal / sales-accounting / purchase-accounting / debit-note / credit-note.
- Routes `/accounts/<same>`, each with a keyed `[company]/[branch]/[year]/[id]` route.
- `/accounts/voucher-register`: `typeCode ""` makes the list show every type, and the entry has a type band.
- Menu labels match `menu_master`: Journal 103, Sales (Accounting) 259, Purchase (Accounting) 163, Debit Note 101, Credit Note 102, Voucher Register 262.

**New parts on the screen:**
- One party on the header (ONE mode).
- GST and HSN cells, plus a GST box with ITC per rate and reverse charge.
- Place of supply, using dropdown 21.
- A one-party bill card, with allocations sent as `lineRowNo 0` and due days sent as `newBill`.
- Party facts.
- An Exceptions dialog on grid 118, shown for Jrl / DrN / CrN and the register.

**Server facts that drove the design (from a read-only contract run):**
- **Header party:** the server accepts any ledger in the party side's groups, and the notes' party groups are empty. So the client filters `/vouchers/ledger-pick` to `isParty`. Qt's dropdown 59 is dev-DB only.
- **GST lines:**
  - `gst` is sent only on lines opposite the party (`gstSide`); the server would add tax to a party-side line, never subtract it.
  - Tax is added on top of the line amount.
  - `taxId` is not defaulted by the server, so it is pre-filled from ledger-pick's `defaultTaxId`.
- **Place of supply:** INPUT types default to the company state. A purchase therefore books IGST only when `posStcd` is sent.
- **Raised bill:** a RAISE type raises a bill for the whole party leg. Its due date is the voucher date plus `newBill.dueDays` (the party's credit days if not sent). Sales and Purchase need a docRefno (`VCH_DOC_REFNO_REQUIRED`).
- **Journal OPTIONAL:** a bill-by-bill party line raises a JOURNAL bill. Any other line leaves the rest nowhere — never an ADVANCE.
- **Menu 262:** `/types` without `menuId` returns every type, each with its own menu's rights. Grid 118 has no rights filter.

**Verified without login:**
- Bodies built by the client domain code pass the compiled DTOs.
- The server's `validate`, run through [[erp-readonly-service-harness]]: Sales GST, Purchase IGST / RCM / TDS, the missing-ref refusal, notes, and Journal raise+settle all came out right.
- `get` read back through `readPosted` for saa00001, pua00121, pua00117, crn00037, jrl00303 and a PurA draft.
- Render smoke tests, routes return 200, tsc clean, 3110 tests pass.

**Departure from Qt:** a note's ref field is captioned "Their ref". Qt's side-based caption calls a credit note's customer a "Supplier".
