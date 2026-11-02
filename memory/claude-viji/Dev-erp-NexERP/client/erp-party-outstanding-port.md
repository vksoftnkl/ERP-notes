---
name: erp-party-outstanding-port
description: "Party-wise Outstanding report (menu 279, /reports/party-outstanding) — built 2026-10-09 from the React plan; forward fields the drill waits on, no-grant state, replaceState URL trick"
metadata:
  node_type: memory
  type: project
  originSessionId: 0b73ab1f-c7a9-4e96-ab8e-ef67bead65d9
  modified: 2026-10-09T08:55:22.103Z
---

Built 2026-10-09 under `features/reports/party-outstanding/` from the user's React plan
(`Prathap/report/plan-react-party-outstanding.md` on the SMB share
`/run/user/1000/gvfs/smb-share:server=192.168.0.103,share=shared/Prathap/report/`, with the
backend plan + mockup PNG beside it). No Qt screen. A peer session wrote the server module
(`ERP server/src/modules/reports/party-outstanding/`) the same day; `wire/types.ts` mirrors
ITS types file, not the plan (ReportHead on every payload incl. `accYear`; nullable `creditDays`).

**Menu (user's call, 2026-10-09):** 279 moved from Financial Statements to straight under
6 Reports as **"Party Outstanding"** (position 7.50). The user had hand-made a menu 280 there and
granted it; that grant (user VIJAY) was moved to 279 and 280 deleted from the dev DB. Server
seed now has 279 under 6; follow-up migration `20261009150000_party_outstanding_menu_reports`
moves it on existing boxes (guarded on the original parent/name). The server's check stays
`PARTY_OUTSTANDING_MENU_ID = 279` — never point a screen at a hand-made dev-only menu id.

**Still open when built (phase 4):**
- No live response was read; the server's e2e (`test/party-outstanding-http.e2e-spec.ts`)
  grants 279 to tester1 temporarily.
- Drill to keyed voucher screens needs FORWARD fields the server does not send:
  `branchId` + `voucherTypeId` on bill rows, `voucherBranchId` + `voucherTypeId` on
  history rows / lastSettlement. Parsed as optional; until then Enter says "no screen yet".
  VOUCHER bills map by `vchr_type_id` (24 Jrl, 26 DrN, 27 CrN, 28 PurA, 29 SalA, 30 RcpV,
  31 PmtV); RECEIPT_ADVANCE = 12, PAYMENT_ADVANCE = 33; SALE_BILL/RETURN/ORDER have no keyed route.
- `/parties` has no search key, so Ctrl+F is only a hint and More filters › Party offers
  the focused party. Opening Balances / Received Cheques take no focus param (drill lands unfocused).

**Things learned:**
- The focused party lives in the URL via `window.history.replaceState(null, "", href)`:
  Next 16 syncs `useSearchParams` for a null state. Passing `window.history.state` (it
  carries `__NA`) BYPASSES the sync.
- The global `AllExceptionsFilter` nests a refusal: `{statusCode, message:{message, errors:[{field, code, count}]}}`.
  Ledger Statement's `errors.ts` missed every code because of it — fixed (unwraps `message`).
- Report helpers now live in `features/reports/shared/` (money, dates, abort, generation,
  xlsx-writer, Bal/Side) — both reports import them; don't copy.
- Ctrl+R hand-off `lib/navigation/party-handoff.ts` (`?new=1&partyId&partyName`) is consumed by
  the receipt AND payment pages (`openOnParty` → `api.pickParty`).
- `view/cells.ts` + `view/tiles.ts` hold every cell's text, so the mockup is asserted to the
  character in node (`cells.test.ts`) — no jsdom here. Verified headless per
  [[erp-client-ui-browser-verification]] with every call fulfilled from `testing/fixtures.ts`.

Related: [[erp-ledger-statement-port]], [[erp-menu-permission-gating]].

**2026-10-15:** Print / PDF / Ctrl+P now print the Party-wise list (A5) — see
[[erp-party-outstanding-print-template]]. WhatsApp reminder (one party's statement) stays disabled;
the export's PARTY_STATEMENT shape is ready for a second design.

**2026-10-29 — Party combo + bill-wise print (client only).** Party moved OUT of More filters into
the strip (row 1, after Area): a `SearchableSelect` filled from the report's OWN `/parties`
(`sort=name`, pageSize 500, up to 10 pages) with the DRAFT's filters minus the party and default
buckets (`query/party-choices.ts`, `use-party-choices.ts`), asked on first open/focus. A pick runs
Show at once; Ctrl+F opens it (the old "no party search" hint is gone). Print/PDF/Ctrl+P follow the
tab — Bill-wise tab → `PARTY_OUTSTANDING_BILLWISE_A5`; the other tabs keep the configured
(party-wise) design. A "Print bill-wise" button shows on the other tabs. The design is picked
through the new `PrintOptionsDialog` prop `preferredTemplateCode` (starts as if chosen under Format;
"use the default" drops it; if the code isn't installed, the dialog says so and prints the default).
Verified headless with fixtures (CDP driver, deleted afterwards).

**2026-10-30 — user's calls:** the strip's Party combo is labelled **"Ledger name"** (texts say
ledger; the list is `led_name` A–Z — the server's party `name` IS `l.led_name`). The user chose
relabel over a type-to-narrow text filter (that would need a `/parties` name search server-side).
The grid now OPENS on `sort=name` `dir=asc` (`parseFilters` default; a sort with no `dir` gets
its first-click direction: name ↑, amounts ↓). The grid's column heading still says "Party".
