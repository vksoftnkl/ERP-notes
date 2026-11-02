# Till: review and fine-tune of every plan, SQL and mockup (REV 2, 2026-10-07)

> **User ask, 2026-10-07:** *"do double check all till plans, sql mockups once again compare with
> other erps fine tune things."*
>
> **A separate plan, per house rule.** `TILL_DESIGN.md`, `plan-till-receipt-payment-expense.md`,
> `plan-noncash-tender-control.md` and `analysis-counter-screen.md` are not edited; §0 lists what this file
> overrides. **Where a plan and this file disagree, this file wins.**
>
> The SQL files 47 / 48 / 49 were **unapplied anywhere**, so they were corrected in place. §1 lists every
> change; the originals are kept in the session scratchpad.
>
> Mockup scripts were corrected and every PNG regenerated (§4). `47_TABLES_GUIDE.md` and `47_session_flow.png`
> were updated in place, because they describe the SQL.

**How it was reviewed:** four independent passes, each reading every file in full.
1. Plans ↔ SQL consistency.
2. Mockups ↔ plans ↔ each other.
3. Comparison with Odoo POS, D365 Commerce, Oracle Xstore, Lightspeed, Ginesys and GoFrugal, plus the Indian payment and tax facts.
4. Logic audit: offline sync, accounting, edge cases, with 30 SQL probes on a throwaway PG 18 loaded with the live schema.

Every finding was then decided here, and the SQL re-validated (§6).

## 0 · What this overrides

| In | Section | Now |
|---|---|---|
| TILL_DESIGN.md | §5.2 lifecycle | + §2.6 (sync check at end billing), §2.7 (cut-off ends billing), §2.8 (movement void only while live) |
| TILL_DESIGN.md | §5.3 day close checklist | §2.11: SUSPENSE is a decision, so close stays green; "open suspense" is informational. §2.10: what a reopened day allows |
| TILL_DESIGN.md | §5.4 expected cash | errata §3.1 (`DR`/`CR`, `SALES_ORDER`); §2.9 (closed sessions read the frozen figures) |
| TILL_DESIGN.md | §5.6 shipped rules | DAY_REOPEN channel **EITHER**; §2.12 single-shop preset |
| TILL_DESIGN.md | §5.7 variance rows | §3.2: the Cr side is **that tender's ledger**; Till cash only for CASH |
| TILL_DESIGN.md | §7.3 error codes | §3.5 adds codes |
| TILL_DESIGN.md | §8 S3, S4, S5 | §2.14 till-to-till change; S4 per terminal (noncash §4.1); S5 sketch figures §3.3 |
| TILL_DESIGN.md | header / §14.3 mockup status | §4: every screen has a mockup, none reviewed yet |
| plan-till-receipt-payment-expense.md | §4 ExpV party / GST | §2.16 |
| plan-till-receipt-payment-expense.md | §5 269ST / 40A(3) | §2.17, §2.18 |
| plan-noncash-tender-control.md | §0 row on `till.variance_default` | §3.4 |
| plan-noncash-tender-control.md | §3 point 3 (PENDING at save) | §2.4 |
| plan-noncash-tender-control.md | §4.2 slip check | §2.2: the outcome is stored (`tvr_rows`) |
| plan-noncash-tender-control.md | §5.2–§5.5 import / TSet | §2.3 (TSet legs), §2.5 (BANK source) |
| plan-noncash-tender-control.md | §6.1 write-off, §6.2 LINKED, §6.3 suspense | §2.1, §2.2 |
| 47 / 48 / 49 | as listed | §1 |

## 1 · SQL changes (47 / 48 / 49, corrected in place)

| File | Change | Why (finding) |
|---|---|---|
| 47 | `ck_tbd_year`, `ck_tss_year`: `acc_year` must be the business date's April–March year | a 02-04 date could be filed in the old year's partition |
| 47 | `uq_tbd_id_date` + `fk_tss_day` on (id, year, **date**) | a session could carry another date than its day |
| 47 | `ux_tct_final` now covers CLOSE **and RECOUNT** as one close-side final | two "final" close counts per session were accepted |
| 47 | `ck_tcm_pickup_witness` | §5.6 says a pickup always has a witness; nothing enforced it |
| 47 | `ck_tvr_cash_terms`: open / moved_in / moved_out only on CASH | a card variance could carry float or drawer moves |
| 47 (restated in 48, 49) | `ck_tap_event`, `ck_tap_role` on `till_approval` | the request table accepted any event / role (e.g. CASHIER) |
| 47 (restated in 48, 49) | events `SYNC_PENDING_AT_CLOSE`, `LATE_ARRIVAL`, `SESSION_DAY_ENDED` | §2.6, §2.7 |
| 47 | DAY_REOPEN channel `EITHER` | REMOTE-only meant no reopen with the store's internet down |
| 47 | setting `till.single_operator` | §2.12 |
| 48 | `ck_td_src_doc_type` + `EXPENSE` | **every expense voucher's tender insert would have failed** |
| 48 | `ck_tap_event` dropped around the PAID_OUT → EXPENSE rename, re-added | the rename would violate 47's new list |
| 48 | 40A(3) row remark: IT Act 2025 renumbering, verify | §2.18 |
| 49 | `till_variance.tvr_rows jsonb` | §2.2 |
| 49 | `acc_settlement_line.asl_tvr_id / _acc_year`, `ck_asl_linked` rewritten, `ck_asl_rule_set`, `ck_asl_tvr_pair`, `ck_asl_tvr_linked` | §2.1: a LINKED line names its bill and the close variance it pairs with |
| 49 | `asi_source` + `BANK`, `ck_asi_bank` | §2.5 |
| 49 | ledger role `GST_ON_CHARGES_PENDING` | §2.3 |

**Running the files:** each file re-runs cleanly on its own. Running an **earlier** file after a later one is
not supported, and never was. Re-running 47 after 48 would re-seed the retired PAID_OUT rule, and 48 after 49
would narrow the event lists 49 widened. Re-run the chain in order, or only the latest file.

## 2 · Decisions (logic)

### 2.1 Money with no bill, whose bill sits in a CLOSED session: pair, never reverse

The case is a payment keyed CASH that was really UPI or card. That session's drawer came up short and the variance was decided. At that point:
- Till cash is already right: the sale gave +x and the close TVar −x.
- The close TVar left Dr Tender suspense x.
- The settlement TSet left Cr Tender suspense x.

**LINKED therefore records the pair and posts nothing** when the close variance went to SUSPENSE:
`asl_match_status = RESOLVED`, `asl_resolution = LINKED`, `asl_td_id` = the bill's tender row, and
`asl_tvr_id` = the close variance. If the close variance went to EXPENSE or RECOVER, one journal posts:
Dr Tender suspense / Cr Cash short & excess, or Cr the cashier's recovery ledger (which pays the cashier back).

The variance row is **not** reversed. Reversing it would put a till-cash leg into a closed session and break
the §5.7 invariant. While the session is still OPEN, LINKED re-tenders the bill instead (the drawer is live).

### 2.2 The slip check's outcome is stored: `tvr_rows`

A close-time non-cash variance lists the tender rows behind it:
`[{td_id, td_acc_year, amount}]`. The close then posts TVar Dr Tender suspense / Cr that tender's clearing
ledger.

**Later, for each listed row:**
- **The statement pays it:** the TSet credits **Tender suspense** (not clearing) for it. Without this, clearing ends −x and suspense +x forever.
- **It's written off** (`NONCASH_WRITE_OFF`): credit Tender suspense. For a row the close never covered, credit the clearing ledger.
- **The write-off request** carries `tap_src_doc_type = 'TILL_VARIANCE'` and `tap_src_doc_id` = the variance.

The **suspense report** has three parts:
- open close-variance rows, from `tvr_rows`, less what was settled, paired or written off;
- unmatched statement lines;
- chargebacks.

Every clearing voucher carries `avh_session_id` of the session it clears.

### 2.3 TSet legs (one per import), complete

| Leg | Amount |
|---|---|
| Dr Bank | net. A negative net (rental-only file) flips it to Cr |
| Dr Bank charges (`BANK_CHARGES`) | fee |
| Dr **GST on charges, invoice pending** (`GST_ON_CHARGES_PENDING`) | GST on the fee |
| Dr Tender suspense | chargebacks + unmatched refunds |
| Cr the tender's clearing ledger | matched sales − matched refunds, **except** rows a close variance parked (§2.2) |
| Cr Tender suspense | §2.2 rows paid now + unmatched sale lines |

**GST on the fee is not input credit yet.** ITC is claimed on the acquirer's monthly tax invoice, which may be
IGST, entered as ExpV or PurA. That entry moves the amount from the pending ledger to `INPUT_*` by the
invoice's own split. Posting CGST/SGST per payout would mismatch GSTR-2B.

**UPI and RuPay debit carry zero MDR** (PSS Act s.10A, still in force, so no GST either). The fee is always
read per line, never as a flat rate.

### 2.4 Which rows wait for a statement

At save, `td_settle_status = PENDING` only when:
- the tender **type** is CARD, UPI or WALLET;
- `ttm_close_mode IN ('SLIPS','STATEMENT')`;
- the tender master has a settlement ledger.

CHEQUE keeps its PDC flow, and BANK transfers are reconciled as bank. Otherwise the Not-received list would flood.

### 2.5 UPI straight to the bank: `asi_source = 'BANK'`

A merchant's own-account UPI has **no payout file and no fee**. The bank credits each UTR in real time, so
the proof is the bank statement.

A BANK import is one statement period: each line is one credit, `asi_payout_ref` is NULL, and fee and tax are 0
(`ck_asi_bank`). Its TSet has no fee legs. "One import = one payout" applies to CARD / UPI / WALLET aggregator
files only.

### 2.6 End billing waits for the device's unsent bills

Expected cash is computed on the store server. A till that billed while cut off from the server still holds
bills in its outbox, so counting now would show a false excess, and later a false shortage.

**At End billing:**
1. The device sends its outbox count and its last `client_seq`.
2. The server refuses COUNTING with `TILL_DEVICE_UNSYNCED` until they have arrived, and logs `SYNC_PENDING_AT_CLOSE`.

**Late arrivals:** a force close may run while a device is unreachable. A bill that arrives later for that
closed session is **accepted, never refused** (the sale happened), with `LATE_ARRIVAL` logged. The session lands on
the cockpit's "explain" list. Its cash was in the drawer at the count, so the excess it caused is re-decided
the 47 way: REVERSED + new row. That reversal is in the same session, so the invariant holds.

### 2.7 The business-day cut-off ends billing

At `till.day_cutoff` (04:00) a live session can no longer bill (`TILL_SESSION_DAY_ENDED`, logged as
`SESSION_DAY_ENDED`). It is ended and counted, or force-closed. A bill after the cut-off needs a session on the new
business date.

Without this, a session running past midnight on 31 March files bills in the wrong financial year.
`ck_tss_year` now also refuses a mismatched year.

### 2.8 Voiding a movement only while its session is live

`MOVEMENT_VOID` only applies while the movement's session is OPEN, SUSPENDED or COUNTING
(`TILL_MOVEMENT_SESSION_CLOSED`). After that, the fix is a new movement in the current session (D7) or a session
reopen. A void after close would change a frozen session's till-cash legs.

### 2.9 Closed sessions read their frozen figures

A re-tender voids the original tender row, so recomputing an old session's expected cash would drift away
from `tss_cash_expected`. Reports read the frozen columns for a CLOSED session. When `expected()` must
recompute (reopen), it counts rows voided **after** `tss_closed_on` as live.

### 2.10 What a reopened day allows

A reopened day takes **corrections only**: money documents, re-tender, and session reopen. New bills keep the
current business date. DAY_REOPEN is now channel EITHER.

### 2.11 SUSPENSE is a decision

A variance sent to SUSPENSE is decided, so day close stays green. The day-close screen and HO show an
informational **"open suspense"** row (session, tender, amount, age). Only PENDING variances, approvals and
movements without vouchers are AMBER or RED.

### 2.12 Single-shop preset: `till.single_operator`

`ck_tap_self` blocks self-approval unless the rule allows it, and every shipped rule says no. An owner who is
also the cashier would be stuck in PENDING_APPROVAL. With `till.single_operator = true` (branch scope):
- the service snapshots `tap_self_allowed = true` on every request;
- the set-up screen proposes `till.blind_close = false` and `till.float_mode = CARRIED`.

Nothing else changes. Chains keep the second person. None of the systems compared forces a second person by
default.

### 2.13 Wording: "hidden-expected count"

D365 uses "blind close" for *parking a shift uncounted* and counting it later elsewhere. Our `till.blind_close`
means "the cashier counts without seeing expected". The setting key stays, but labels and docs say
**"hidden-expected count"**. `count_place = CASH_OFFICE` is our "park and count later".

### 2.14 Till-to-till change

There is still no direct till-to-till movement: cash goes counter → safe → counter, so every rupee passes the
safe book. S3 offers one action, **"Change from C02"**, which writes the TDrp + TFlt pair in one call. There is no
schema change.

### 2.15 Considered and not adopted

- **Xstore's manager reconcile count after the cashier's:** CASH_OFFICE count place plus witness already cover it.
- **A paid-in linked to a paid-out:** retired with PAID_OUT in 48.
- **Fewer session states:** the six states are justified for chains, and the §2.12 preset keeps a single shop to open, sell, count, close.

### 2.16 Expense voucher (ExpV): party and GST (overrides 48 plan §4)

ExpV is posted by its own route family (`/api/v1/expenses/*`) with its own derive, **not** through the voucher
register engine. The register engine's `partyMode` is NONE/ONE/MANY, and ONE makes the party **mandatory**
(`voucher-register.service.ts:206`), which does not fit an optional supplier.

So the seeded type keeps `vchr_party_mode = 'NONE'`, `vchr_in_register = false`, and the expenses service:
- accepts an optional supplier ledger;
- writes the GSTR-2 rows itself, the same rows PurA writes, when a GST bill is entered;
- writes tender rows with `td_src_doc_type = 'EXPENSE'` (now legal, §1).

### 2.17 Cash limits: whom the sum is per

- **40A(3)** sums cash to one **party ledger** per day. A cash expense with only free-text "paid to" is not
  summed; a single one over the limit WARNs and asks for a party.
- **269ST** on a walk-in cash customer can only be checked **per bill** unless a mobile number or PAN is
  captured. The plan says so; the guard sums across bills only for a known party.

### 2.18 Indian-law facts to re-check before go-live

- **The Income-tax Act 2025 is in force from 1 April 2026** and renumbers 269ST and 40A(3); the limits are
  unchanged. The statutory rows keep their codes. Add effective-dated rows citing the new section numbers
  **after confirming them against the Act text**. The mapping found so far is from secondary sources.
- **TCS 206C(1H) on sale of goods was dropped from 1 April 2025.** The live `statutory_limits` still has
  active 206C(1H) rows. This is outside the till module; **it still needs raising with Prathap** (not yet in any notes file).

## 3 · Errata in the plans (read these as corrected)

1. **TILL_DESIGN §5.4:** the values are `td_dr_cr = 'DR' / 'CR'` and the source `SALES_ORDER` (not 'D'/'C', ORDER_ADVANCE).
2. **TILL_DESIGN §5.7 and the 47 guide `tvr_treatment`:** a variance's Cr / Dr side is **that tender's ledger**. Till cash is used only for CASH. A card variance to SUSPENSE is Dr Tender suspense / Cr Card clearing.
3. **TILL_DESIGN S5 sketch:** open **1,990** (the counted float, §5.4), out **30,000**, expenses **250** (48), expected **13,720**, counted **13,640**, attempts 13,590 → 13,640.
4. **Non-cash plan §0:** `till.variance_default` is cash-only (its seed says so); 49 adds `till.noncash_variance_default`.
5. **Error codes** (TILL_DESIGN §7.3 lists the 47 set): see each plan's list, plus `TILL_DEVICE_UNSYNCED`, `TILL_SESSION_DAY_ENDED`, `TILL_MOVEMENT_SESSION_CLOSED`, `TILL_SESSION_CLOSED`, `TILL_CASH_LEDGER_DIRECT` (48) and the non-cash plan §7 codes.
6. **Settings count:**
   - 47 seeds 17 + `till.single_operator` = 18;
   - 48 adds 3;
   - 49 adds 6;
   - total **27**.
7. **TILL_DESIGN header** ("no PNG mockups yet") and §14.3 ("the visual spec"): mockups exist for every screen, and **none is reviewed yet**. They become the spec only once approved.
8. **The worked story's paid-out:** RAVI K's 250 is an **Expense voucher** (cash tender), not PAID_OUT. moved_out = **30,000**, expenses = **250**, expected unchanged at **13,720**.

## 4 · Mockups

Scripts corrected and **15 PNGs regenerated + 1 new** (`till_slip_check_ui_mockup.png`), each checked
visually. None is reviewed by the user yet. The story now holds across every mockup:

| Item | Was | Now |
|---|---|---|
| MEENA T's 07-10 session | C01-261007-01, live while RAVI opened C01 (impossible: `ux_tss_counter_live`) | **C04-261007-01** 08:58–14:02. C01-…-01 was **voided** (opened in error), which is why RAVI's is -02 |
| PRIYA M (C03) | COUNTING in one mockup, waiting approval in another | **PENDING_APPROVAL** everywhere: 48,980 / 47,730 / −1,250. Sessions-list variance total −2,310 |
| The 500 with no bill | a card line on T2 (the close would have caught it) | a **UPI-QR2** payment (STATEMENT mode, never counted at close). It is SURESH B's −500 cash shortage, paired by `asl_tvr_id`, no voucher, no reversal |
| HDFC card payout of 08-10 | 22 lines, gross 18,944.00, fee 474.56, GST 85.43, net 18,384.01 | **21 lines, gross 18,444.00, fee 470.06, GST 84.62, net 17,889.32**, clearing 16,445.00, suspense 1,999.00 (the unconfirmed suggestion) |
| TSet GST legs | Input CGST + SGST per payout | one leg **GST on charges, invoice pending** (§2.3) |
| bill 160 on T2 (bil00611) | "no ref" | keyed auth Q1W2E3 · 0000 that is on no slip. The close had a **card variance −160 SUSPENSE** (`tvr_rows`), shown on day close and the sessions list |
| bill 2,100 on T2 (bil00628) | short at the statement | the machine charged 2,100 (the close matched); the **provider** paid 2,010 → PARTIAL, PROVIDER_ERROR |
| Slip check | no mockup | **new**: T2 expected 6,354 (card 6,804 − refund 450), batch 6,194, short 160 |
| Day close | SUSPENSE shown as "undecided" (AMBER) | an **INFO "open suspense"** row (C02 cash −500, card −160); AMBER only for PENDING; header 08:58 auto by MEENA T (C04) |
| RAVI's paid-out 250 | PAID_OUT tpo00012 | **Expense voucher exp00011** (cash); Out 30,000 + Expenses 250; expected 13,720 unchanged |
| Approval gate / Inbox / HO | PAID_OUT | the Payment form, **CASH_PAYMENT** (shipped 1,000); EXPENSE 2,400 |
| Approval Rules master | PAID_OUT, DAY_REOPEN remote | EXPENSE + CASH_PAYMENT, a NON-CASH group (NONCASH_WRITE_OFF, SETTLEMENT_RESOLVE), DAY_REOPEN **EITHER** |
| Reason master | 39 shipped, PAID_OUT category | 55 shipped: EXPENSE (supplier / staff advance inactive "use Payment") + NONCASH (10) |
| Cash Movement S3, F11 menu rev 1 | a Paid out panel / item | removed; S3 shows a Top-up; F11 uses the rev-2 list |
| End Shift S4 | one "Card" line | **Card T1 (HDFC) · batch 0031 · 14 · 12,340.00**; a mismatch opens the slip check |
| Cockpit tabs | Approvals 2 | RAVI **4** (float mismatch, 2 pickups, variance); KAVYA **3** |
| Cockpit gate text | "over tolerance 10" | "over the RS Puram rule 50.00 (tolerance 10.00)" |
| Bill numbers | B-00405 … | the house format **bilNNNNN** (RAVI bil0165x…, SURESH bil005xx–006xx), returns srtNNNNN |

**Not yet drawn** (plan screens with no mockup):
- the sale-bill tender popup with the mandatory reference and the duplicate refusal;
- Settlement's Suspense and Duplicate-refs tabs;
- Day Open;
- the X / Z report slips;
- a receipt (menu 99) taken on a till.

## 5 · Comparison summary

| Capability | Ours | Odoo POS | D365 Commerce | Oracle Xstore | Lightspeed | Ginesys |
|---|---|---|---|---|---|---|
| Session per cashier / drawer | ✔ 6 states | ✔ | ✔ fixed / floating / shared | ✔ register or till | ✔ | ✔ |
| Opening float counted | ✔ issued / carried | ✔ | ✔ | ✔ | ✔ | ✔ |
| Hidden-expected count | ✔ | ✘ | (blind close = park the shift) | back-office count | ✘ | ? |
| Pickup / safe drop / bank drop | ✔ | cash in/out | ✔ | ✔ | payout type | ✔ |
| Till-to-till change | via safe, one action (§2.14) | ✘ | ✔ | ✔ | ✘ | ✔ |
| Tolerance per tender | ✔ | ✔ | ✔ | reasons | ✘ | ? |
| Rule × authority approvals | ✔ richest | add-ons | permissions | manager override | ✘ | ✘ |
| Sync check at close | ✔ **added (§2.6)** | ✘ | ✔ | ✘ | n/a | ✘ |
| Card/UPI statement matching | ✔ (49) | bank reco | statements | ✘ | ✘ | report |
| Store day close | ✔ checklist | ✘ | statements | ✔ | ✘ | ✔ |

Sources: Microsoft Learn (D365 Commerce cash management, shift and drawer management, financial reconciliation),
Oracle Retail Xstore till management, Odoo POS cash control, Lightspeed pay-ins and payouts, Ginesys store
settlement, NPCI / PSS Act s.10A zero-MDR coverage, Income-tax Act 2025 section mappings.

## 6 · Validation (2026-10-07, after the changes)

A fresh throwaway PG 18 was loaded with the live schema and reference rows, then 47 → 48 → 49 were applied. Each file
re-runs cleanly on its own. Every probe gives the expected result, and every refusal comes from the intended constraint:

| Probe | Result |
|---|---|
| final CLOSE + final RECOUNT · two final RECOUNTs | refused `ux_tct_final` |
| final OPEN + non-final CLOSE + final RECOUNT | accepted |
| CARD variance with moved_in | refused `ck_tvr_cash_terms` |
| CARD variance with `tvr_rows` | accepted |
| PICKUP with no witness / with the cashier as witness | refused `ck_tcm_pickup_witness` / accepted |
| unknown event · role CASHIER · PAID_OUT request | refused `ck_tap_event` · `ck_tap_role` · `ck_tap_event` |
| EXPENSE + NONCASH_WRITE_OFF requests | accepted |
| 02-04-2027 filed in 2026-2027 | refused `ck_tbd_year` |
| 31-03-2027 and 01-04-2026 in 2026-2027 | accepted |
| session dated ≠ its day | refused `fk_tss_day` |
| RESOLVED/LINKED naming its bill + paired variance | accepted |
| RESOLVED/INCOME naming a bill | refused `ck_asl_linked` |
| MATCHED pointing at a variance · MATCHED without a rule | refused `ck_asl_tvr_linked` · `ck_asl_rule_set` |
| BANK import with a payout ref / without one | refused `ck_asi_bank` / accepted |
| tender row with source EXPENSE / unknown source | passes `ck_td_src_doc_type` / refused by it |

The probes the first pass ran (16 on 49) still hold. **Nothing was applied to 192.168.0.106.**
