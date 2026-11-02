---
name: erp-qt-6f15977-port
description: "Port of Qt NexERP 6f15977 (2026-10-13) — one-call post (notes 106) on bill/receipt/payment/physical stock, draft-rights gating, receipt UPI surcharge, till witness PIN / supervisor moves / variance decide / shell open-check, expense F8 picker, user-menu ancestor sync; notes 110 NOT ported"
metadata:
  node_type: memory
  type: project
  originSessionId: dfeba9f8-3b6c-4e34-98d1-3543846d593a
  modified: 2026-10-13T05:52:09.383Z
---

Ported 2026-10-13 from NexERP 5e25760..6f15977 by seven parallel subagents, one folder each (follows
[[erp-qt-d5c3637-till-expense-settlement-port]] and [[erp-expense-voucher-qt-d97cdd4-port]]).

- **Server notes 106 — save AND post in ONE create call**, a refusal leaves nothing (no draft, no number):
  sale bill + testbill `/bills/create` `post:true` + `overrides` (after `/validate` + popup) — see
  [[erp-sale-bill-lifecycle-port]]; receipt / payment `/…/create` = draft body + the POST-shape settlement keys
  (`allocations`, `creditsApplied`, `otherLineBills`, `onAccount`; payment REMOVES ones the post builder lacks)
  + `post:true`; physical stock `/stock/physical/create` with `header.status:"POSTED"` after a confirm
  (Ctrl+Enter posts, F5 = Save Draft). The old `/…/post` mutations were deleted from the client.
- **Draft rights** via `lib/permissions/document-rights.ts`: Save draft greyed (never hidden) without
  `maySaveDraft(saved)`, handlers re-check (F5 can't go round it); Post-only users may now key + post.
  Sale order: `sale-order.rights.ts`; it already sent `soStatus:'CONFIRMED'` on auto_post (≡ `post:true`).
- **Receipt surcharge**: `features/accounts/receipt/domain/surcharge.ts` (paise, half-up, 200@1% → 202.02);
  typed figure = for the bills, gross collected; `SURCHARGE_RECOVERED` seeded like BANK_CHARGES and sent
  (server seeds it when omitted, refuses one that disagrees). Payment shares the tender row type (fields optional).
- **Till**: `session/supervisor-moves.tsx` (grid 147 PICKUP/TOP_UP; DB writes `VOIDED`, Qt checks `VOID`);
  cash movement redesign + `witnessPin`; `sessions/variance-decide*` → `/till/sessions/decide-variance`;
  shell `components/layout/use-till-session-check.tsx` (open-check once per user|branch per login, kept in
  sessionStorage; navigates to Open Till / warns).
- **Expense** F8 = register picker modal over grid 137; posting legs hidden. **User admin** rights tree syncs ancestors.

**Server notes 110 — since PORTED (a449b8c port, commit 7b0546a, 2026-10-14); kept for history:** `witnessPin` is "no longer read";
a PICKUP/TOP_UP is entered by the session's own cashier with `supervisorId` + `supervisorPin`
(`GET /till/movements/supervisors`). Web (and Qt 6f15977) still send witnessPin and forbid the cashier — the
movement dialog will be refused (422 `supervisorId`) once that server change runs. Needs its own port when Qt
catches up. Also: server now names the approval grant menu "Till Approval Authority (285)"; UI text still says 276.
