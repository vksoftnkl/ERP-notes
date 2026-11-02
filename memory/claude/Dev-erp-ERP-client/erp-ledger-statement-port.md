---
name: erp-ledger-statement-port
description: "Ledger Statement report (menu 258, /reports/ledger-statement) — built 2026-09-25 against the server's types file before its routes existed; what to re-check when they land"
metadata:
  node_type: memory
  type: project
  originSessionId: f22d285a-769c-4b47-9970-bd9005165d2e
  modified: 2026-09-25T08:36:33.349Z
---

Built 2026-09-25 under `features/reports/ledger-statement/` from the user's
React plan (no Qt screen; `plan-backend-ledger-statement.md` was not on disk).
Route `app/reports/ledger-statement`, bound in `DEFAULT_PRIMARY_MENU` under
Reports › Financial Statements (menu 258 existed locally with 2 grants; server
seed + `20260925140000_ledger_statement_prereqs` migration ship it).

State when built: another session was writing the server module
(`ERP server/src/modules/reports/ledger-statement/`) — only types + DTOs
existed and every route 404'd. `wire/types.ts` mirrors THAT types file, which
differs from the plan: `side` never null (zero = `0.00 DR`), `creditLimit`
exists, `src.*`/branch/voucherNo nullable, `/voucher-legs` needs `ledgerId`.

**When the routes land (plan phase 4):** read one real response per route and
fix `wire/parse.ts`; narrow `wire/errors.ts` `findCode` to the refusal shape
the new exception filter really sends. `testing/fixtures.ts` stays: the unit
tests and any browser re-check are built on it.

Server asks the screen still depends on: `src` for receipts (type 12 leaves
`avh_src_*` NULL, so Enter says "no screen yet"); `groupId` on
`header.ledger` (PgUp/PgDn looks it up via `/ledgers?search=name` meanwhile);
`pairVoucherId` (hover-pairing matches voucherNo in narration); a print
purpose `LEDGER_STATEMENT` (Print/PDF/WhatsApp disabled). Only Receipt Entry
has a keyed route to drill into.

Excel uses an in-repo stored-zip xlsx writer (`export/xlsx-writer.ts`) — no
SheetJS dependency; LibreOffice opens it. Verified headless over CDP with
every `/api/v1` call faked from the fixtures. See
[[erp-rtk-pending-query-dedupe-abort]], [[erp-client-ui-browser-verification]].

**2026-10-09:** `wire/money.ts`, `wire/dates.ts`, `api/abort.ts`, `query/generation.ts` and
`export/xlsx-writer.ts` moved to `features/reports/shared/` for Party-wise Outstanding
([[erp-party-outstanding-port]]). `wire/errors.ts` now unwraps the global filter's nested
`message` object — before that it mapped NO server code (every refusal read "refused").

**2026-10-15:** the print purpose ask is DONE — `LEDGER_STATEMENT` + the A5 design, and the
footer's Print / PDF / F11 are wired ([[erp-ledger-statement-print-template]]). WhatsApp stays
disabled. Open server bug: `coreRows`' user_master join still uses the AND-guarded
`created_by::uuid` cast, so `/vouchers` and `/export` 500 for ledgers touched by non-uuid
`avh_created_by` ('VKPOS') — use `createdByJoinSql` as day-summary does.

