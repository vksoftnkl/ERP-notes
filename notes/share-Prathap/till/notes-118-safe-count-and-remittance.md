# Notes 118: The safe: balance, count, and remittance to the bank (2026-10-14)

**STATUS:** server DONE (migration 20261015100000) and verified live 2026-10-15 from the Qt Cash Office (menu 287): balance, exact SAFE count, CIT remit crm00001 100.00, remit-confirm, refused remit TILL_REMIT_OVER_BALANCE. Decide-variance not live-tested yet. Open: the safe figures on the till.day.header print data (§ day Z). Was: asked. TILL_DESIGN §13 phase 5 (S9). On Business Day (274) the "Safe counted" row is grey and the Remittance row's "Remit →" button is disabled, both because these routes are missing.

What 47 already has:
- `till_safe` (ledger, insured limit, drop safe);
- `till_count.tct_kind` SAFE, with `tct_safe_id`;
- `till_cash_movement` kinds REMIT and SAFE_TRANSFER, with `tcm_bag_no`, `tcm_seal_no`, `tcm_cit_agency`, `tcm_ref_no/date` and `tcm_ledger_id` required for REMIT;
- the CRem "Cash Remittance" voucher type;
- the ledger roles SAFE_CASH and CASH_IN_TRANSIT;
- the rules SAFE_VARIANCE (STORE_MANAGER) and REMITTANCE (CASH_OFFICE, COUNTER);
- the setting `till.day_close_safe_count` = true;
- the `till_business_day` columns `tbd_safe_opening`, `tbd_safe_expected`, `tbd_safe_counted` and `tbd_remitted`.

`movements/create` takes only DROP, PAID_IN, EXCHANGE, PICKUP and TOP_UP today.

## 1 · `GET /till/safes/balance?companyId&branchId[&safeId]`

Per safe:
- **opening:** the last SAFE count, or 0 before any count;
- **in:** hand-overs and drops received (notes 113/114: only once RECEIVED; NULL receipt status = straight in), pickups, safe transfers in;
- **out:** float issues, top-ups, remittances, transfers out, and exchanges netted;
- **expected now**, the insured limit, and the last count (when, by whom, the difference);
- the ledger balance of `tsf_ledger_id`, so any drift between the book and the movements shows.

Rights: Till Sessions (273) VIEW, or the new Cash Office menu below.

## 2 · `POST /till/safes/count` `{ companyId, branchId, safeId, lines[], notes, witnessPin? }`

- **Blind:** the caller does not see the expected figure before the count, as for the cashier's close. The response then shows expected, counted and difference.
- Writes a `till_count` of kind SAFE (`tct_expected`, `tct_total_counted`) and logs `SAFE_COUNT`.
- **The difference:**
  - **within `till.cash_tolerance`:** post at once (TVar: Cash Short & Excess against the safe ledger);
  - **beyond it:** the SAFE_VARIANCE rule (STORE_MANAGER). Decide it in the same step when the caller's grant covers it (treatment + reason, as approve-cash does since notes 114), else leave it pending for another approver.
- **Where the variance lives:** `till_variance.tvr_session_id` is NOT NULL, so a safe variance needs either a nullable session plus a `tvr_safe_id` with a new stage SAFE, or its own small table. Your choice; tell me which.
- **The day:** the count of the business day feeds `tbd_safe_counted` and satisfies notes 117's `SAFE_NOT_COUNTED`. More than one count a day is allowed; the last one is the day's.

## 3 · `POST /till/movements/remit` (or REMIT on `movements/create`)

`{ companyId, branchId, safeId, amount, bankLedgerId, mode: 'BANK_DEPOSIT' | 'CIT', bagNo?, sealNo?, citAgency?, slipNo?, slipDate?, lines[]?, notes, approverPin? }`

- The REMITTANCE rule as seeded (CASH_OFFICE, COUNTER). Refuse above the safe's expected balance.
- **Posting:**
  - **Bank deposit by staff:** CRem Dr Bank, Cr Safe.
  - **Through a CIT agency:** Dr CASH_IN_TRANSIT, Cr Safe now; then, when the bank statement shows it, a **confirm** step (`POST /till/movements/remit-confirm { tcmId, bankDate, bankRef }`) posts Contra Dr Bank, Cr Cash in transit. A short credit by the bank becomes a variance (the SAFE_VARIANCE rule).
- **The day:** feeds `tbd_remitted`, and clears notes 117's `SAFE_OVER_INSURED` when the safe falls under its limit.
- Void: as other movements, with reason and approval, refused once confirmed.

## 4 · A menu for it

A **Cash Office** screen comes back here, now with real work in it: the safe's balance, Count safe, Remit, and the remittances in transit. Please give it a **new** menu id, not 286: that one stays retired, since its grants were soft-deleted in notes 114. Rights: VIEW for the balance, CREATE for count / remit, OVERRIDE for deciding a safe variance.

## 5 · Reads for the list

A grid of remittances (date, safe, mode, bag / seal, amount, status IN_TRANSIT / CONFIRMED / VOIDED, age) is mine to register once the columns are settled. Tell me if REMIT stays on `till_cash_movement` (I'd say yes) or gets its own table.
