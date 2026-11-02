# Notes 134: Cash Position (S11) — menu, HO review routes (2026-10-17)

**STATUS:** done 2026-10-17. Menu **289** Cash Position (271, 2.60, VIEW/CREATE/EXPORT) and `POST /till/reviews/create` + `GET /till/reviews/list` are live. The Qt screen is registered on 289; Add HO review posts to the route (a 403 TILL_RIGHT_CREATE without 289 Create was checked live). §3's one-call `/till/cash-position` is still open (not needed now).

The S11 head-office screen is approved (mockups `till/till_ho_cash_position_rev2_ui_mockup.png` and `till/till_ho_review_ui_mockup.png`) and is being built on our side (Qt). It reads registered grids, so most of it needs nothing from you. Two things do.

## 1 · The ask: one new menu row

| Field | Value |
|---|---|
| Parent | **271** Till |
| Name | **Cash Position** |
| Position | 2.60 (after Till Reports 288) |
| Visible | yes |
| Verbs | `{VIEW, CREATE, EXPORT}` — CREATE = may add an HO review |

Please tell us the id.

## 2 · The ask: HO review routes (`accounts.till_session_review`)

The table exists (with `ck_tsr_outcome` and `ck_tsr_target`), but nothing writes to it. Please add:

1. **`POST /till/reviews/create`**:
   - **Body:** `companyId, branchId, accYear`, **either** `dayId` **or** `sessionId` (the table's `ck_tsr_target`), `outcome` (one of `ck_tsr_outcome`), `note` (required unless OK), and `refVoucherId` + `refVoucherAccYear` (only for WRITTEN_OFF / SUSPENSE_CLEARED).
   - **Who:** `reviewedBy` is the caller and `reviewedOn` is now.
   - **Right:** the new menu's CREATE.
   - **Refuse:** a day or session that is not this branch's.
   - **Insert only:** a review is never edited; a new one supersedes it.
2. **(Optional) `GET /till/reviews/list?companyId&branchId[&dayId|sessionId]`**. We read history from a grid (174), so this route is only needed if you want it for the web screen.
3. **A `till_event` `HO_REVIEW`** carrying the outcome, if you journal reviews like everything else. This is optional too.

Until the route exists, the screen's **Add HO review…** button is greyed with "waits for the server (notes 134)".

## 3 · What we built on our side (no action)

Grids registered 2026-10-17 from `grids/till_cash_position_registration.py`:

| Grid | What |
|---|---|
| **170** TILL CASH POSITION | One row per branch with a counter or safe, as at the end of the "as on" day. It shows day status, an older day still open, live sessions (and over max hours), awaiting receipt (AWAITING handovers / drops), safe book (the Cash Office rule of grid 158), insured limit, oldest safe count, in transit (REMIT not confirmed, oldest age), CLOSE cash variance (day / month to date), approvals waiting, last `till_event`, last review, and flags. |
| **171** TILL CASH POSITION LIVE | Sessions still holding a drawer. The screen asks your **`/till/sessions/expected`** for each one's cash, as Till Sessions does. |
| **172** TILL CASH POSITION EVENTS | The day's exceptions, every branch. |
| **173** TILL CASH POSITION CASHIERS | Month-to-date cash shorts per cashier, every branch. |
| **174** TILL CASH POSITION REVIEWS | One branch's reviews (for the popup's history). |

**Checked:** grid 170's safe book for ZTS on 17-10 (−8,000) equals your `/till/safes/balance` `expected`.

**One question:** cash in tills costs one `/sessions/expected` call per live session, across every branch. For a chain with 40 live drawers that is 40 calls a minute while the screen is open. Would you rather give one route, `GET /till/cash-position?companyId&date`, answering each branch's live expected cash, so the screen makes one call? It is not needed now; we will switch when it exists.

## 4 · For information: wrong-PIN events, who they name

A `PIN_FAIL` from a **change** (and any movement witness) has `tev_user_id` = the **caller**, and the person whose PIN was wrong is in `payload.witnessBy`. A `PIN_FAIL` from a force close has the approver in `tev_user_id`.

Our grids now read `COALESCE(payload.witnessBy, tev_user_id)`. Grids 161–163 were updated today for this.

If you would rather `tev_user_id` always named the person whose PIN failed, tell us and we will drop the COALESCE.
