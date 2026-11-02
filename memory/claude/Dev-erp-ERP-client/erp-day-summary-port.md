---
name: erp-day-summary-port
description: "Account Day Summary report (menu 284 + 181, /reports/day-summary) — built 2026-10-12 from the React plan; wire mirrors the SERVER types (not the backend plan's examples); open server asks"
metadata:
  node_type: memory
  type: project
  originSessionId: 68f4cb69-a132-4c6a-8ec4-0cde993741e7
  modified: 2026-10-12T05:54:47.670Z
---

Built 2026-10-12 under `features/reports/day-summary/` from the user's React plan
(`Prathap/report/plan-react-account-day-summary.md` on the SMB share, backend plan +
mockup beside it). No Qt screen. Route `app/reports/day-summary`; `DEFAULT_PRIMARY_MENU`
has "Account Day Summary" (284) and "Day Book Detail" (181, hidden) → `?tab=vouchers`
under Financial Statements (the guard strips the query, so either grant opens it).

A peer server session wrote `ERP server/src/modules/reports/day-summary/` the SAME day
(migration `20261012120000_account_day_summary_menu` applied: 284 exists, 0 grants).
The client was first written to the backend plan's examples, then re-aligned to the
server's `types/day-summary.types.ts` + the service's check facts. Differences that bit:
- `natureFilter` is a `vchr_nature` (`@IsIn`) — tile codes are refused. Tiles map
  SALE_RETURNS→CREDIT_NOTE etc.; EXPENSES drills by the ExpV type id found by CODE in
  `/voucher-types`; PAYMENTS = nature PAYMENT includes ExpV (server has no split).
- Money / Day-wise / cash-tile balances are `null` under a voucherTypeId filter → "—";
  `AMOUNT_TIES.facts.moneyChecked=false` keeps the tie-out neutral.
- `hasChildren` = has child GROUPS, so group Enter is NOT gated on it (leaf groups have ledgers).
- `/voucher-types` = `{items}`, no query; `/daily` = `{cashOpening, bankOpening, days}`;
  `/vouchers` payload carries `draftsOnly`; rows carry `branchId`; `/export` =
  `{summary, vouchers[], totalRows}` (no amount total → no Total row in the sheet).

**Still open:** no live response read (needs a session; route answers 401). Server asks:
`totals.cancelledPairs` on `/vouchers` (footer clause waits on it); draft/pair voucher
numbers in DRAFTS / CANCELLED_PAIRS facts (the plan's wording; parsed when sent).
Drill map is by voucher TYPE onto the keyed screens (Voucher Register has no keyed view;
src rows like SALE_BILL have no keyed screen → "No screen opens this voucher type yet").

Verified headless (52 checks) with the CDP recipe of [[erp-client-ui-browser-verification]]:
fixtures imported straight from `testing/fixtures.ts` via `node --experimental-strip-types`;
snap chromium can only download into `/home/vk/snap/chromium/common/…`, not `/tmp`;
`innerText` applies CSS `text-transform` (section titles read UPPERCASE).
Related: [[erp-ledger-statement-port]], [[erp-party-outstanding-port]], [[erp-menu-href-mapping]].
