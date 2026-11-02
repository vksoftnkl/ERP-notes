# Audit: Claims (section A of purchase_claims_shortage_ui_mockup.png)

Audited 2026-10-29 by qt-tester. Built as `PurchaseClaimsDialog` (`src/modules/purchase/return/purchase_claims_dialog.{h,cpp}`, grid ui_table 60), opened from the **Claims...** button on the Return screen (menu 25). Mockup: `/home/vk/Shared/Prathap/purchase/purchase_claims_shortage_ui_mockup.png`, section "A. CLAIMS - money expected" + the "Settle claim" box + "What Settle writes" box + callouts 1, 2, 5.
Snapshots: `/home/vk/Shared/Prathap/purchase/audit/claims_*.png`, `claims_modal_*.png`. Rendered 1560 x 560, live :3011, ZT return PRN/2026-2027/D14/00129 (cancelled after).

Layer: **Q** Qt only, **S** server route / field, **D** DB. CD = `purchase_claims_dialog.cpp`, CC = `purchase_return_cols.h` (PurchaseClaimCols).

## Summary

Rows audited: 40 (39 judged).

| Verdict | Count |
|---|---|
| YES | 10 |
| PARTLY | 9 |
| DIFFERENT | 5 |
| NO | 15 |

The money logic works and is tested (Credit part / whole, Short-settle, Reject, Undo last, reopen). What does NOT exist is the screen the mockup draws: the mockup is ONE page "Supplier dues" holding the claims list, a settle form, a posting preview, AND the shortage section. The build is a bare list dialog with five buttons whose inputs come from one-line pop-up prompts.

### The 10 biggest gaps

1. **No "Settle claim" panel.** The mockup form (Action, Received as, Amount, Balance, Cancel / Settle) is replaced by modal one-line prompts: "Credit the claim - Amount" then "Credit note - no (optional)" (claims_modal_01/02). Qt only.
2. **No "What Settle writes" posting preview** (Dr supplier / Cr CLAIM_RECEIVABLE 150.00; Dr WRITE_OFF / Cr CLAIM_RECEIVABLE 30.00, plus the three rule notes). Qt only (server amounts are deterministic).
3. **Credit + write-off of the balance in one step** (Balance combo "30.00 -> write off (reason: short-settled)") does not exist. Today it is two separate button presses, and Short-settle always writes off the whole open amount. Qt (two calls) or S (one call).
4. **Received as is fixed to Supplier CN.** The mockup offers credit note / cash / bank / adjust in next bill, with CN no and CN date. Qt sends `receivedAs: CN` always and never the CN date. The server DTO accepts only `CN` and `ADJUST`; cash / bank are not accepted. Qt for ADJUST + cnDate, S for cash / bank (or leave to Receipt, see the memory note).
5. **Grid lacks Age, Reason, Written off, Status chip, Last action** and the tick box. `/purchase/claims/open` returns no reason, no written-off split, no status, no last action text (and no age; it can be computed from `docDate`). The Settled column in the dialog already counts write-offs: after a Reject the claim shows Settled 100.00 / Open 0.00 (claims_04) and nothing says it was written off. Q + S.
6. **No multi-select settle** ("settle one at a time or several"): no tick column, selection is one row. Qt only.
7. **Not one screen with the Shortage section**; no "Supplier dues - {supplier}" title with the "3 OPEN" chip, no A / B section headings. The plan (frontend F6) said one view `dues/claims_shortage_view` with two sections; folder and class do not exist. Qt only + a decision (see question below).
8. **Reachable only from the Return screen.** No menu entry of its own. Menu 179 "Claim Management" (parent 5, Accounts) exists in `fixed.menu_master` and is not wired; the build note says it is "not ours to take". Needs your decision.
9. **Against column is "SUPPLIER - ZT-PUR Supplier" text**; the mockup shows the plain party word (Supplier / KPN Parcel in amber when it is not the supplier). The column is also truncated at the dialog width. Qt only.
10. **Claim reference reads "Return No | Ln"** in two columns instead of one "PRN/.../00003 - r2"; Kind wording differs ("Goods sent" ok, "Destroyed" vs "Destroyed here"). Qt only.

## Table

| # | Mockup element / callout | Built? | What is there now | What is missing | Layer | Evidence |
|---|---|---|---|---|---|---|
| **Frame** | | | | | | |
| 1 | Page title "Supplier dues - Sri Murugan Agencies" | DIFFERENT | Dialog title "Open claims - ZT-PUR Supplier" (small label) | "Supplier dues" heading, big style | Q | claims_01_open_rows.png; CD:30-40 |
| 2 | Sub-title "Acme Foods - Coimbatore" | NO | not shown | company / branch | Q | claims_01 |
| 3 | Status chip "3 OPEN" (amber) | NO | Count only in the text line "2 claim row(s) - Open 218.00" | chip | Q | claims_01 |
| 4 | Section heading "A. CLAIMS - money expected" + caption "from Purchase Return rows with outcome Claim; settle one at a time or several" | NO | No heading | heading + caption | Q | claims_01 |
| 5 | One page shared with section B (shortages) | NO | Two separate screens; claims is a dialog under the Return screen | shared screen | Q (decision) | frontend plan F6; no `dues/` folder |
| **Claims grid** | | | | | | |
| 6 | Tick-box column | NO | Row gutter "#" only | check column | Q | claims_01 |
| 7 | Claim ("PRN/.../00003 - r2") | DIFFERENT | Two columns: Return No (full refno) + Ln | one combined cell | Q | claims_01; CC:46 |
| 8 | Date | YES | dd/MM/yyyy | - | - | claims_01 |
| 9 | Age (days) | NO | Not shown | computed from doc date | Q | claims_01 |
| 10 | Item | YES | item name | - | - | claims_01 |
| 11 | Kind (Goods sent / Destroyed here) | PARTLY | "Goods sent" / "Destroyed" | wording | Q | claims_01; CD:104 |
| 12 | Against (Supplier / KPN Parcel amber) | PARTLY | "SUPPLIER - ZT-PUR Supplier", "TRANSPORTER - ZT Loadman ..." (truncated) | party name only, amber when not the supplier | Q | claims_01 |
| 13 | Reason (Near expiry / Breakage / Damage) | NO | Not shown; return rows never carry a reason (see audit-return rows 31) | column; `/claims/open` has no reason | Q + S + D | `openClaims` select list has no reason |
| 14 | Value | YES | claim value (CLAIM_SENT total incl. GST; CLAIM_DESTROYED taxable) | - | - | claims_01 |
| 15 | Settled | PARTLY | Sum of ALL settlement rows, including write-offs | credit-only figure | S | claims_04 (Settled 100 after Reject) |
| 16 | Written off | NO | Not shown | server field (sum of SHORT_SETTLED / REJECTED rows) | S | `/claims/open` fields |
| 17 | Open (bold) | PARTLY | shown, not bold | bold | Q | claims_01 |
| 18 | Status chip (OPEN amber / PART blue) | NO | Not shown | chip derived from open / value | Q | claims_01 |
| 19 | Last action ("CN SMA/CN/88 - 1,000.00") | NO | Not shown; `lastSettlementId` is fetched but no text | server field (action, cn no, amount) | S | CC:46 `LastSettlementId` hidden |
| 20 | Row selection highlight / red outline on the focused row | YES | Row tint on the current row | - | - | claims_01 |
| 21 | Callout 1: a claim is born on a Purchase Return row (Claim - sent / destroyed here); ITC already reversed at return post | YES | Rows come only from posted return rows with those outcomes | - | - | claims_01; test_purchase_return C |
| **Settle claim box** | | | | | | |
| 22 | Box title "Settle claim - PRN/.../00001 - r1 (open 180.00)" | NO | No box | box | Q | claims_modal_01 |
| 23 | Action combo (Credit received / Short-settled / Rejected) | DIFFERENT | Three separate buttons "Credit... / Short-settle... / Reject..." | combo inside the form | Q | claims_01 buttons |
| 24 | Received as combo (Supplier CN - no - date) | NO | Always `CN`; the CN no is a pop-up prompt, optional; CN date never sent | combo; CN date; ADJUST (server supports); cash / bank (server does not) | Q (+S for cash / bank) | CD:160-172; `SettleClaimDto.receivedAs` = CN, ADJUST |
| 25 | Amount (red-bordered input) | PARTLY | Pop-up prompt "Amount the party credited (open 118.00)" prefilled with the open amount | inline field | Q | claims_modal_01_credit_amount_prompt.png |
| 26 | Balance combo "30.00 -> write off (reason: short-settled)" | NO | Credit with amount < open just leaves the rest open; to write it off the user presses Short-settle afterwards (always the whole open amount) | one-step credit + write-off; reason | Q (two calls) / S | CD:152-185 |
| 27 | Cancel / Settle buttons | NO | none (action fires on the button press) | buttons | Q | claims_01 |
| 28 | "What Settle writes" box: Dr / Cr lines with amounts + 3 note lines | NO | Not present | preview panel | Q | claims_modal_01 vs mockup |
| 29 | Callout 2: settle in parts, any number of settlements until Open = 0 | YES | Credit 100 of 118 -> open 18; further settlements allowed (tested) | - | - | test_purchase_return C |
| 30 | Callout 2: short-settled or rejected -> WRITE_OFF with a reason | YES | Reason prompt for both; server writes WRITE_OFF; Reject re-avails GST on goods sent | - | - | claims_modal_03 (re-open guard); test_purchase_return C |
| 31 | "Undo = a reversing settlement row (never delete)" | YES | "Undo last..." asks a reason and posts settle-undo | - | - | claims_01 buttons; CD:196-229 |
| 32 | Callout 5: Against whom (supplier / transporter / insurer); Received as (CN, cash, bank, adjust) | PARTLY | Against shown; received-as limited (row 24) | see row 24 | Q / S | claims_01 |
| **Buttons / keys** | | | | | | |
| 33 | Actions are in the settle form (Settle / Cancel) | DIFFERENT | Bottom row: Credit... (primary), Short-settle..., Reject..., Undo last..., Close - Esc | form buttons | Q | claims_01 |
| 34 | Opening the screen | DIFFERENT | Only via "Claims..." on the Return screen; scoped to the picked supplier, else all | its own menu (179 exists, unused) or a menu-195-style entry | Q + decision | form_purchase_menu.h:24-29; fixed.menu_master 179 |
| **Warnings / messages** | | | | | | |
| 35 | Settling a fully settled row | YES | "Nothing open - This claim is fully settled - Undo last to reopen it." | - | - | claims_modal_03_nothing_open.png |
| 36 | Server refusals shown | YES | plain text in the hint line + warning box | - | - | CD:187-195 |
| 37 | Totals line | PARTLY | "2 claim row(s) - Open 218.00" | the mockup has none for claims; fine | - | claims_01 |
| **Data checks** | | | | | | |
| 38 | `GET /purchase/claims/open` fields | PARTLY | refno, docDate, lineNo, itemName, outcome, claimAgainst, partyName, claimValue, settled, open, lastSettlementId | age, reason, writtenOff, status, lastAction, kind label | S | purchase-return.ts:567-585 |
| 39 | `POST /purchase/claims/settle` can take cnDate, docDate, receivedAs | PARTLY | server accepts; Qt sends none of cnDate / docDate and fixes CN | Qt wiring | Q | `SettleClaimDto` (swagger) |
| 40 | Credit that arrives with the return ("Supplier CN no / date if given now") matched here | NOT RENDERED | `supplierCnNo` is saved on the return header; whether it auto-creates a CREDIT settlement was not tested | - | - | - |

## Open question for you (menu 179)

Menu 179 "Claim Management" exists under Accounts (parent 5). The plan's claims screen is "one page with the shortage section" (mockup) or at least its own menu. Which do you want: (a) claims list + settle form inside the Return screen, (b) a new Claims menu in the Purchase group, (c) the page "Supplier dues" holding A and B on menu 195, or (d) take over menu 179?

## Update 2026-10-29 — rebuilt as the Supplier Dues page (menu 297)

Built by qt-screen: `SupplierDuesPage` (`src/modules/purchase/dues/supplier_dues_page.{h,cpp}`, `supplier_dues_cols.h`) + `ShortageCloseDialog` (`dues/shortage_close_dialog.{h,cpp}`); menu 297 under Purchase; the Return screen's "Supplier dues" button opens it filtered to its supplier. The old `PurchaseClaimsDialog` (ui_table 60) is retired. Routes: `/purchase/supplier-dues/{open,settle,settle-undo}` + `/purchase/shortages/{close,close-undo}`. Theme block: share `theme/seed-template-v46.qss` (unpublished). After-snapshots at 1560 px: `audit/dues_after_01..06*.png`. Test: `SRC=test_purchase_supplier_dues tests/purchase/run.sh`.

| # | Was | Now | Note |
|---|---|---|---|
| 1 | DIFFERENT | YES | "Supplier dues — <supplier>" (all suppliers when unfiltered) |
| 2 | NO | YES | company · branch city |
| 3 | NO | YES | "N OPEN" amber (totals.openCount); "NOTHING OPEN" green |
| 4 | NO | YES | heading + caption (caption also names shortage claims on a transporter / insurer) |
| 5 | NO | YES | one page: A. claims + B. shortages (the shortage RECEIPT part is the next job) |
| 6 | NO | YES | ✓ box column — click or Space; several settle in one call |
| 7 | DIFFERENT | YES | "PRN/…/00206 · r1" (full ref + source bill in the tooltip) |
| 8 | YES | YES | |
| 9 | NO | YES | server ageDays |
| 10 | YES | YES | |
| 11 | PARTLY | YES | Goods sent / Destroyed here / Shortage claim |
| 12 | PARTLY | YES | "Supplier", else the party name in amber (tooltip: TRANSPORTER · name) |
| 13 | NO | YES | reasonName ("—" when the return row has none) |
| 14 | YES | YES | |
| 15 | PARTLY | YES | heading "Credited" (credit-only figure, server `credited`) — see deviations |
| 16 | NO | YES | server writtenOff |
| 17 | PARTLY | PARTLY | not bold: no fonts in code and the QSS cannot reach one grid column |
| 18 | NO | YES | chip OPEN amber · PART blue · SETTLED green · WRITTEN OFF grey |
| 19 | NO | YES | "CN SMA/CN/88 · 1,000.00" / "Adjust in next bill · …" / "Written off · …" / "Rejected · …"; date + reason in the tooltip |
| 20 | YES | YES | |
| 21 | YES | YES | |
| 22 | NO | YES | "Settle claim — <claim> (open X)"; "Settle N claims (open X)" for several |
| 23 | DIFFERENT | YES | Action combo: Credit received / Short-settled / Rejected |
| 24 | NO | YES (shape differs) | Supplier CN / Adjust in next bill + CN no + CN date as three widgets; cash / bank = the Receipt link (user 10-29) |
| 25 | PARTLY | YES | inline, theme-bordered; one row only (several = each row's whole open) |
| 26 | NO | YES | "18.00 stays open" / "18.00 → write off" + reason — ONE call (writeOffRest) |
| 27 | NO | YES | Undo last · Alt+X, Cancel, Settle · Ctrl+Enter (asks first) |
| 28 | NO | YES | live dryRun legs (debounced 350 ms) + credited / written off / GST re-availed + the 3 notes |
| 29 | YES | YES | |
| 30 | YES | YES | reason required on the page for Short-settled / Rejected |
| 31 | YES | YES | Undo last on the current claim; "Show settled" lists fully settled ones so their last action can be undone |
| 32 | PARTLY | YES | cash / bank deliberately not on Settle (Receipt link) |
| 33 | DIFFERENT | YES | |
| 34 | DIFFERENT | YES | menu 297 + the Return's shortcut (filtered) |
| 35 | YES | YES | Settle disabled, tooltip "… is fully settled" |
| 36 | YES | YES | box worded by errors[].code + grey server line; a refused dry run shows in red in the preview |
| 37 | PARTLY | YES | totals strip: claims, money open, credited · written off, shortages, goods value open, GST held |
| 38 | PARTLY | YES | /supplier-dues/open carries age, reason, writtenOff, status, lastAction, kindText, claimRef |
| 39 | PARTLY | YES | cnDate, docDate (app date), receivedAs CN / ADJUST sent |
| 40 | NOT RENDERED | NOT TESTED | a supplierCnNo typed on the return header still does not create a settlement here |
