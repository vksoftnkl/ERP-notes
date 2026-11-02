---
name: erp-qt-425fa80-port
description: "Port of Qt NexERP bec3f98..425fa80 (2026-10-17) — till approval gate (428 → Needs approval dialog, inbox, decide), Contra tender picker, receipt surcharge inside the tender, user roles notes 125, Business Day books checks, Day Book by tender tabs; what was NOT ported and server traps"
metadata:
  node_type: memory
  type: project
  originSessionId: 86c1f51d-739f-45e6-b6ae-7e4747e8aa57
  modified: 2026-10-17T06:20:04.839Z
---

Ported 2026-10-17 by six parallel subagents on a shared layer written first. Follows [[erp-qt-bec3f98-port]].
Not committed by me; peers staged the files mid-task, I unstaged only this port's paths ([[erp-workspace-layout]]).

- **Approval gate (server notes 124 §5):** shared `features/sales/till/shared/approval-gate.ts` (`approvalNeededOf`
  reads a 428 TILL_APPROVAL_REQUIRED from ANY module's body, flat or nested) + `approval-request-dialog.tsx`
  ("Needs approval": approver here + PIN, or Send to inbox). Wired on Open Till (FLOAT_MISMATCH, TILL_COUNTER),
  Expense post (EXPENSE + `approvalId` on /expenses/post) and Payment (notes 131: the 428 KEEPS a draft — its id is
  adopted first so the retry never posts a second copy). Inbox + decide dialog live in `features/sales/till/sessions`
  ("Waiting for approval (n)" on menu 273; count via the QUIET read). Cockpit is a route, so a cash-gap row navigates.
- **Expense deviation:** /expenses/validate pushes TILL_APPROVAL_REQUIRED into `refusals` under REFUSE; the web
  leaves it out of what greys Post (else the 428 dialog is unreachable). Qt has that dead end — told the user.
- **NexDropdownSingle `onChange(selection, row?)`** — the picked row (Qt `selectedRow()`); grant dialog uses
  `row.usr_type` → `levelForUserRole`.
- **Receipt surcharge** is INSIDE the tender now: `amount × p/(100+p)`; 200 @1% → 202.00. Sale bill/testbill/order
  were already equivalent (they add pct AND flat; receipt is pct OR flat). Server rounds the master pct to 2 dp,
  client 3 — a 1.125% master would be refused (not fixed).
- **Contra (Con):** ledger cell = tender picker (cash + BANK-type tenders with a ledger), `lines[].tenderId`
  REQUIRED by the server (VCH_MONEY_LINE_NO_TENDER); posted Contras don't return it. Server fed6b770 drops all
  cash/bank ledgers from the ledger picker → Receipt Dr / Payment Cr money side lists nothing (expected).
- **Business Day:** Books = drawers / Books = safe checks, green checks folded, sessions grid min 6 rows.
- **Day Summary:** `?tab=tender` / `?tab=no-tender` (Ctrl+4/5). SERVER BUG: `withRows=false` is ignored on
  /reports/tender-day-book (DTO @Transform reads the converted value) — client sends `limit=1` as a workaround;
  drop it once the DTO uses `OptionalQueryBoolean`. Menu 284 still defaults to Summary (Qt now opens by tender).
- **NOT ported:** dd 43 → Ledger master Alt+C/Alt+A on the tender master (needs an inline mode in the 3000-line
  `account-ledger/page.tsx`, busy with another session's edits). `scripts/sql/reset_transactions.sql` (DB tool).
- Gap found: Posting Ledger is required on the tender form (web + Qt) but the server wants none on RRN / TEMP_CR /
  CREDIT / LOYALTY tenders (notes 126) — those can't be saved from either form.
