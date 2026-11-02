---
name: error-codes-notes-119
description: "Notes 119 (2026-10-15) — every refusal needs errors[].code (Qt words the box by code only); parts 1 + 4 built; receipts/payments/cheques coded 10-15 (notes 124: VCH reuse + ACC/CHQ catalogues); ~700 literal sites elsewhere still uncoded"
metadata:
  node_type: memory
  type: project
  originSessionId: 1e7ecb9a-55b9-4694-8a97-3ac095580f2b
  modified: 2026-10-15T05:17:26.072Z
---

Built 2026-10-15:
- §4: `src/common/validation/house-validation.ts` — `createHouseValidationPipe()` (main.ts, test/helpers/payment-e2e + identity-app, validateDto util, ledger-masters controller). One error per field, `code` = VALIDATION_ + constraint in UPPER_SNAKE (whitelist → VALIDATION_WHITELIST), nested path `lines.0.qty`. House UUID decorators are now a constraint named `isUuid` (was `matches`) → VALIDATION_IS_UUID. `frameworkValidationCode()` codes Nest param-pipe 400s ("uuid is expected") in both filters; 500s carry INTERNAL_ERROR, numeric overflow VALUE_TOO_LARGE.
- **Shape change:** AllExceptionsFilter now passes a house body through at the TOP level (it used to nest it under `message`) — modules without their own filter (reports, hsn …) changed; day-summary / party-outstanding e2e helpers updated.
- §1: TxnHoldErrorCode (HOLD_*, all 37 sites), TenderDetailErrorCode (TENDER_LINE_*), SALES_ERROR_CODES gained CHEQUE_* / FIELD_REQUIRED / LINE_*; `throwNotFound` and `normalizeRequiredText` take an optional `code`.

**Why:** cashiers could not read server sentences; Qt (`api_error_text.cpp`) shows the user's simple wording keyed on `errors[0].code`, the server text small and grey.

**How to apply:** measure what is left by temporarily making `code` required on BOTH `ModuleErrorDetail` (module-shared.utils) and `ModuleApiErrorDetail` (common/types) and counting TS2741/TS2345/TS2322 — TS2344 alone is a module type with optional code, not a missing site. Never reuse a code for a new meaning; keep state fields (`status`) on the detail. accountVoucherHeader's throws are all commented out (nothing to code). See [[till-safe-notes-118]].

Notes 124 (2026-10-15, later): receipt / payment / cheques done. `code` is now REQUIRED on Receipt/Payment/ChequeErrorDetail. Codes come from the user's wording doc: VCH_* reused (vouchers/vouchers.errors.ts) plus `ACC` (receipt/types/acc-error-codes.ts, shared by all three) and `CHQ` (cheques/types/cheque-error-codes.ts). AllocationError details carry code (AllocationErrorDetail). requireRoleLedgers / collectRoleLedgerGapErrors take an optional `code` (ACC_ROLE_UNMAPPED). ACC_OTHER_LINE_UNFUNDED was my own pick (not in the doc). The wording rows live in prisma/seed/Error_Messages.json, written by another session.
