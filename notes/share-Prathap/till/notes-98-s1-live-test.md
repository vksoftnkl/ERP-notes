# Notes 98 — S1 Open Till Session: live test findings

**From:** Qt client (Viji / Claude) · **Date:** 2026-10-09 · **For:** Prathap

> **STATUS 2026-10-09 (re-tested live): §1 FIXED with option (b).** Session ZT1-261009-02 (1,990 counted
> against 2,000): the OPEN variance posts at open (EXPENSE, POSTED, Dr Cash Short & Excess 10 / Cr till
> cash); after the close (Z 2) Cash In Hand nets **0.00**. Still open: `tss_variance_status` reads
> `NONE`; the pre-fix session ZT1-261009-01 still leaves +10 on Cash In Hand (test data — reverse or
> ignore).

The Qt S1 screen (menu 272, `TillOpenSessionDialog`) was run end to end on the test box as VKPOS
on device VKPOS1, against counter **ZT1** and safe **ZTS** (ledger *Store Safe Cash*), both made
for this test in Acme Foods / Coimbatore. The story is the mockup's own: float 2,000 issued,
1,990 counted at open, reason *Safe gave less*.

| Step | Route | Result |
|---|---|---|
| open-check | GET `/till/sessions/open-check` | ✔ free list, day auto-open, carriedFrom |
| open (from the dialog) | POST `/till/sessions/open` | ✔ `ZT1-261009-01` OPEN; TFlt `tfl00059` Dr Cash In Hand 2,000 / Cr Store Safe Cash 2,000; OPEN-stage variance −10 PENDING; notes kept |
| re-open the dialog | open-check → `deviceSession` | ✔ Resume panel, open blocked, Resume adopts the session |
| end billing | POST `/till/sessions/end-billing` | ✔ |
| count 1,990 | POST `/till/sessions/count` | ✔ ACCEPTED (attempt 1) |
| close | POST `/till/sessions/close` | ✔ Z 1; TDrp `tdr00067` Dr Store Safe Cash 1,990 / Cr Cash In Hand 1,990 |

Session id `01a11f23-37ee-7aba-b871-e80ddd202e0b` (2026-2027) if you want to look at the rows.

---

## 1. A closed session leaves the till cash ledger off by the opening difference (verified)

After the close, the session's own legs net to:

| Ledger | Net |
|---|---|
| Cash In Hand (till cash) | **+10.00 Dr** |
| Store Safe Cash | −10.00 |

The drawer is empty (all 1,990 went to the safe), yet the till ledger still holds 10. The −10
OPEN-stage `till_variance` row is `PENDING` with no voucher, and **`close` does not look at it**:
the session closed with Z 1 and `tss_variance_status = NONE`.

Phase 3 (the FLOAT_MISMATCH approval) is what would treat it, but until then every opening
difference silently stays in the till ledger after the close. Suggest one of:

- **a.** `close` refuses (428 `TILL_APPROVAL_REQUIRED`), or the session goes `PENDING_APPROVAL`,
  while any variance of the session is `PENDING` — the same gate as a close-stage variance; or
- **b.** until phase 3, post the OPEN-stage variance at open with the default treatment
  (Dr Cash short & excess / Cr till cash, TVar) and let the approver re-treat it later.

**Order with notes 97:** (a) sends the session to `PENDING_APPROVAL`, which notes 97 §1.2 shows
is a dead end today (it locks the counter, the cashier and the device for good). So do 97 §1.2
first, or take (b), which needs nothing from 97.

Also: `tss_variance_status` stays `NONE` while an OPEN-stage variance is pending. It should
probably read `PENDING`, so the cockpit / sessions list can find it.

## 2. Small things

- **`open` does not take a reason.** `OpenTillSessionDto` has no `reasonId`, so the client puts
  `FLOAT_MISMATCH −10.00: <reason name>` in `notes`. When the phase 3 gate lands, please accept
  `reasonId` (category FLOAT_MISMATCH) on the open and store it on the OPEN variance.
- **No safe on the open.** The float always comes from the counter's safe or the branch
  default. Fine for now; the mockup's "From safe" picker is shown as text.
- **Force close** has no route yet; the client shows the button disabled.

## 3. Not a server item — for the record

The till needs the login token to carry a device. `/auth/login` matches
`fixed.device_master.dev_device_uid`; the developer PC here logs in with an unregistered uid, so
its open-check answers `TILL_DEVICE_REQUIRED` (correct). The client now says "register this PC in
Device Master" for that answer.
