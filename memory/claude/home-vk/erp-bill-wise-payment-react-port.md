---
name: erp-bill-wise-payment-react-port
description: 2026-10-03 Bill-wise Payment (menu 100) ported from Qt PaymentEntry to React features/accounts/payment; uncommitted; receipt engine now policy-driven and room-capped; never run logged-in
metadata:
  node_type: memory
  type: project
  originSessionId: e05d84f3-470d-4bf0-bac8-1b311496d7c1
  modified: 2026-10-03T05:35:04.192Z
---

Built 2026-10-03 in `/home/vk/Dev/erp/ERP client`, UNCOMMITTED: `features/accounts/payment/`, `store/api/paymentApi.ts`,
`app/accounts/payment/**` (route `/accounts/payment`, menu label "Bill-wise Payment"), registries (ui tables 42/43,
grid 123, dropdown 60 `paymentPayee`), `"Payment"` cache tag. Reuses the receipt port's types/engine/components.

Shared receipt changes (also uncommitted): `receipt/domain/roles.ts` `SettlementPolicy` (payment passes
`PAYMENT_SETTLEMENT`: no leg split, BANK_CHARGES is an ADDITION); new `receipt/domain/deductions.ts`
`planDeductions` = server notes-62 A1 rule (unpinned deduction fills only the sent bills' ROOM, overflow on
account). This CHANGED receipt behaviour: its "worked example" test claimed 70,750 on a 70,000 bill, which the server
409s — test corrected. Receipt TDS-on-advance was a latent 400 before this.

Verified without login: bodies pass the server's compiled DTOs with forbidNonWhitelisted; 10 reducer-built scenarios
run through the server's own `allocate()` (OUT) + `computePaymentTds()` agree to the paisa. Never clicked through in a
browser, never posted against a real DB.

Server facts the Qt client got wrong or that bite a port:
- a typed BANK_CHARGES line is always 400 unless Σ = Σ tdMdrAmt → not offered in the Type picker (Qt offers it)
- posted `/payments/get` returns `otherLines: []` → lines rebuilt from `legs[]` roles
- `creditsApplied[]` on a posted payment holds BOTH rows of each spend pair (DR on bill, CR mirror on held item) —
  read only `drCr === "DR"`
- summary keys are still `creditsHeld` / `totalCredits`; cheque row must not send `tdRefNo` (leaf taken at post)
- dropdown 60 and grid 112 exist only in the dev DB (no seed/migration) → see [[erp-live-vps-deploy]] before deploying

Related: [[erp-sale-bill-react-port]], [[nexerp-qt-client-source]].
