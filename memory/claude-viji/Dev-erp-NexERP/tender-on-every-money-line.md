---
name: tender-on-every-money-line
description: User goal 2026-10-16 — Day Book by tender to find missing txns; decided rule "every CASH/BANK ledger line carries a tender", NO tender field on ledger master; notes-126 asked
metadata:
  type: project
---
User's goal (2026-10-16): accounts by tender type, so a missing transaction can be found (Day Book tender-wise). After a long back-and-forth (user was confused by "till-controlled ledger" talk), the agreed model, matching Tally bank-allocation / Odoo payment methods / SAP payment means:
- ledger = where, tender = how; tender is a LABEL on money legs, accounts unchanged; bill-wise Receipt/Payment unchanged.
- NO tender list on the ledger master (the user first proposed one, then mandatory tender on ledger — rejected: two maps drift). Tender master stays the only tender↔ledger map; `led_ledger_type` CASH/BANK (derived + locked from group) is the money flag.
- Rule: every leg on a CASH/BANK ledger carries a tender row; CASH auto, BANK → user picks BANK or CHEQUE (Kvb → two tenders OK; issued cheques use the cheque book's bank).
- Till transfers (TFlt/TDrp/TRcv/TVar/CRem) get CASH tender rows; no-device calls are back office, not a pass; Jrl/Con on till cash/safe refused.
- Reports: Day Book by tender + "money lines with no tender" (always empty).
- Clean-up: CARD/WALLET → Card/Wallet Clearing ledgers; TEMP.CR/CREDIT/LOYALTY → no ledger (posting ignores it).

**Why:** evidence rcv00140/rcv00146 (15-10) Dr Cash In Hand 15,000 in a till branch with no session or tender row; register skips routeMoneyDoc when no instrument.

**How to apply:** Shared/Prathap/accounts/notes-126-every-money-line-has-a-tender.md. Qt work after it's live: register auto-CASH / Bank-or-Cheque, two reports, Business Day "Books = drawers" row. Explain in plain words with a concrete example — the user said they were "big confused" by abstract framing. Related: [[till-management-design]], [[voucher-register-screen]].

**10-16 later — AGREED screen design (user's idea, notes-126 §6):** the money side of every voucher is picked from the TENDER MASTER list, never a ledger (like Receipt/Payment 99/100). RcpV/PmtV: instrument mandatory, tender picker. Contra: from/to tenders (server: Contra has no instruments yet). Jrl/DrN/CrN/PurA/SalA: no cash/bank ledgers at all; bank charges/interest go on Payment/Receipt. One BANK tender per bank account; one CHEQUE tender (receive → Cheques In Hand, pay → cheque book's bank). Built: Alt+C on the tender master's ledger field (dropdown 43 → LedgerEntry menu 53, dropdown_masters_accounts.cpp). User asked "am I thinking right" — confirmed yes; don't re-litigate.

**2026-10-16 PM:** notes 126/128 LIVE (checked: no-tender report, Jrl/Con/PmtV refusals, tender-master save codes). Qt: Contra ledger cell = tender menu (openContraTenderMenu/applyContraTender, lines[].tenderId), tests/register `contra126` 7/7. Grid 119 still offers cash/bank → notes-130 asked. Leftovers → till/notes-129.

**2026-10-17 — notes-130 is LIVE (grid 119 changed 10-16 17:02):** grid 119 now lists 0 ledgers for RcpV DR, PmtV CR and Con, so the By line of a receipt had an empty popup (user bug report). The server takes RcpV/PmtV money ONLY as the party line's instrument (a typed Dr cash leg with tenderId → VCH_INSTRUMENT_NOT_ALLOWED). Fix (user picked "By line = tender list"): `isMoneyRow` / `openTenderMenu` / `applyMoneyTender` in voucher_register_entry.cpp. The pick becomes the instrument of the party lines above that have none (cash goes straight on; cheque/needsRef, and any non-cash on a payment, open the instrument popup pre-set). The money line is then removed, because the server's generated leg shows it. typedDifference counts an instrumented line as balanced. Test `run.sh money126` 14/14. pickcursor + many FAIL both before and after the fix, because they pick Cash from grid 119: outdated by notes-130, not yet rewritten.

**2026-10-17 later:** the user said the PmtV Dr side was still empty. The REAL cause is that Prathap's notes-130 grid 119 SQL has `--` comments, and the runner refuses it (400 GRID_RUN_SQL_INVALID "Comments are not allowed"), so EVERY type/side is empty. A psql count of the SQL looks fine (556), so always test a grid through `/configured-grid-sql/run`. notes-132 asks for the comment-free SQL (accounts/notes-132-grid-119.sql). claude has no UPDATE on fixed.grid_details.

**2026-10-17 — notes-132 DONE:** Prathap saved the comment-free SQL. Verified `run.sh grid119` 16/16 (runner, all 8 types × side; money sides 0; no CASH/BANK anywhere; limit max 100 per page) and `popup119` 3/3 (real key path, Records 238 for "k"). money126 14/14.
