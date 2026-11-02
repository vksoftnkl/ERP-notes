---
name: dto-own-undefined-trap
description: "Server bug class found 10-29: after the house ValidationPipe every DTO field is an OWN property (ES2022), so hasOwnProperty ≠ 'sent' and `?? null` wipes stored values; use isPresentField"
metadata:
  type: project
---

Found 2026-10-29 (customer audit, then print templates). tsconfig target ES2022 → class fields are emitted, and `transform: true` makes every declared DTO field an own property, holding `undefined` when the client left it out.

- Bitten live: every Qt customer save reset the ledger TDS flag; collection days wiped; group rename checked the wrong company; publishing a print template with a status-only version row nulled ptv_params (18 prompts) + ptv_note → every render refused PRINT_RENDER_PARAM_UNKNOWN. Repaired PARTY_OUTSTANDING_BILLWISE_A4 by publishing rev 2.
- Fix: `isPresentField(obj, key)` in src/common/utils/module-shared.utils.ts (own AND !== undefined); applyPresentFields uses it. Customer, customer-group and ledger company fixes are COMMITTED (user, 10-29 13:04). The sweep (print-template + frozen compare, print-template-assignment, charge master, items, suppliers, sale agent, tax rate, widget, area, batch prefix, tender, shipping address, bank account, device list, cust rates, staff advance ledger) is STAGED in the server index, not committed; helper `isPresentField` in module-shared.utils.ts. See notes-163.
- Specs must run the body through `createHouseValidationPipe`, or they can't see the bug.
- A PUBLISHED print revision is frozen: re-publish as a NEW revision with a FULL row, never a status-only row.

Related: [[agent-team]], [[crud-blank-number-dropped]].
