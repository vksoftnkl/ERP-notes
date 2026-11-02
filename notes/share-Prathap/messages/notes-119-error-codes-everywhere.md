# Notes 119: An error `code` on every refusal (2026-10-15)

**STATUS:** mostly DONE, live 2026-10-15: §1 held bills + payment lines, §4 VALIDATION_ codes, INTERNAL_ERROR. Still open: §1 voucher header + 2 voucher throws, §3 `code` required, §2 office masters (see the follow-up at the end).

## Why

Cashiers could not understand the server's messages. On 10-15 a cashier tendering a bill got this box:

> **409** · Session VIJI PC-261015-01 is PENDING_APPROVAL: it takes no money
> • sbSessionId — Session VIJI PC-261015-01 is PENDING_APPROVAL: it takes no money

The user's rule now covers **every** error box: short, simple English (many users do not read formal English), plus the server's own text small and grey underneath, so a screenshot still tells us what happened.

The Qt side is built. `src/core/api/api_error_text.cpp` reads `errors[0].code` and shows our wording. For the till it now shows:

```
⚠  Cannot take payment
Your till is waiting for supervisor OK.
Do: Call supervisor.
Bill not lost.
409 · TILL_SESSION_NOT_OPEN · sbSessionId            ← grey
Session VIJI PC-261015-01 is PENDING_APPROVAL: …     ← grey, your text, unchanged
```

The wording is the user's, in `Shared/Prathap/messages/till-error-wording.md`.

**The client switches on `code` only, never on the message text.** So a refusal without a `code` cannot get simple wording. It keeps your sentence, under a plain title.

**Nothing changes in your messages.** Keep writing them for developers: they are the grey line.

## What already has codes: thank you, nothing to do

| Module | Helper |
|---|---|
| till (all 67 codes) | `throwTill` |
| sales posting: bill, return, quotation, order, DC | `throwSalesLocked / Refused / Right / Invalid / Refusals` |
| receipts, payments, vouchers | `throwState / Missing / Right / Refused / Invalid` |
| tender settlement, issued cheques, expense | their own helpers |

## What has none: the ask

Counted from the source on 10-15. These are throws that reach the client with `{field, message}` and no `code`.

### 1 · Counter first (a cashier meets these)

| File | Uncoded throws |
|---|---|
| `sales/txn-hold/txn-hold.service.ts` (held bills: resume / cancel) | 5 |
| `sales/posting/sales.errors.ts` (one generic throw left) | 1 |
| `accountsModule/tenderDetail` | 5 |
| `accountsModule/accountVoucherHeader` | 4 |
| `accountsModule/vouchers` (two left) | 2 |

### 2 · Office masters (staff, not cashiers; lower priority)

| Area | Uncoded throws |
|---|---|
| `settings/*` (company 18, branch 13, employee 11, print template 9, designation 7, department 7, theme 5, users 4, app settings 3) | 85 |
| `sales/loyalty` + `sales/promotion-scheme` (scheme masters) | 62 |
| `Inventory/*` (tax rate 9, qty price 5, price 2, others 13) | 29 |
| `master/*` (widget 7, charge detail 6, charge 5, batch prefix 5 …) | 25 |
| `fixed` 15 · `audit-log` 14 · `items-cust-rates-master` 6 · `purchase` 4 · `users` 4 · `reports` 4 · `stocks` 2 · `gst` 1 | 50 |
| ledger / group / tender masters in `accountsModule` (ledgerRole 4, ledgers 3, shipping 2, bank 2, tender 2, tender type 2, company group 2, acc group 2, ledger map 1, reconcile 1) | 21 |

### 3 · Make `code` required in the generic helpers

`throwBadRequest`, `throwConflict`, `throwNotFound`, `throwForbidden`, `throwUnprocessable` and `throwOnUniqueConstraintError` (`common/utils/module-shared.utils.ts`) take any `{field, message}`.

- If `ModuleErrorDetail` gains `code: string` as a **required** key, `tsc` lists every site still missing one, and no new throw can forget it.
- `throwNotFound(message, field, detailMessage)` needs a `code` parameter too.
- A plain `throw new BadRequestException('text')` (Inventory, stocks, fixed) should go through these helpers, so it gets the same shape.

### 4 · DTO validation (the global `ValidationPipe`)

Today a bad body comes back as `message: ["property x should not exist", "y must be a UUID"]`: no field, no code.

Please give the pipe an `exceptionFactory` that answers in the house shape, one error per field:

```json
{ "success": false, "message": "Validation failed",
  "errors": [ { "field": "sbSessionId", "code": "VALIDATION_IS_UUID", "message": "sbSessionId must be a UUID" } ] }
```

- Make `code` = `VALIDATION_` + the class-validator constraint name in upper snake case: `VALIDATION_IS_NOT_EMPTY`, `VALIDATION_IS_UUID`, `VALIDATION_WHITELIST` (for "should not exist"), and so on.
- Qt can then say "Fill in Customer", or for a client bug, "Something went wrong. Do: Send a photo of this to support."

### 5 · Rules for codes

- **Prefix by module:** `HOLD_NOT_FOUND`, `COMPANY_GSTIN_DUPLICATE`, `TAX_RATE_IN_USE` (as `TILL_` and `SALES_` already do).
- **Never reuse a code for a new meaning.** Add a new code instead; our wording is keyed on it.
- **Keep the extra fields on the detail**, like till's `status`. Qt picks the wording by them (PENDING_APPROVAL vs COUNTING vs SUSPENDED).
- **Name each module's codes in one enum** (as `TillErrorCode`), so we can write the wording file from it.

## Our side, next

- **Wording files:** sales posting (about 101 codes) and receipts / payments, in the same short style as the till file. The user reviews each one before it is coded.
- **The sales `/validate` refusals** (the warning strip on the bill) show the server sentence today; they get the same wording.

Tell us when part 1 and part 4 are live. Those are what the cashier meets.

## Follow-up (2026-10-15 afternoon): checked against your source

Thank you. Done in the source:
- §1 held bills (32 `HOLD_…`) and payment lines (`TENDER_LINE_…` / `TENDER_REF_…`);
- the last generic throw in `sales.errors.ts`;
- §4 `createHouseValidationPipe` with `VALIDATION_…`;
- `INTERNAL_ERROR` in the global filter.

**Not live yet.** The API on 192.168.0.106:3011 still answers the old shape: `till/days/open` with a bad body gives no `code`, and `txn-holds/get` on a missing hold gives no `code`. Notes 118's routes still answer 404 too (`/till/safes/balance`), although the migration `20261015100000_till_safe_notes_118` ran at 09:47. **Please deploy / restart the API** with both.

Still open from this note:
1. **§1 voucher header:** `accountVoucherHeader` (4 throws) and the 2 generic throws left in `accountsModule/vouchers`. These files have not changed since 10-08.
2. **§3 `code` required:** `ModuleErrorDetail` is still `{ field, message }`, and `code` is optional on `throwNotFound`. Making `code: string` required lists every site still missing one at compile time.
3. **§2 office masters** (settings, Inventory, master, fixed, audit-log, ledger masters): lower priority, no change yet.
