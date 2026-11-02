# Issued Cheques — Qt client plan (menu 52)

**For Viji. Written 2026-09-14; code after the payment's §2 and the
`/issued-cheques/*` + `/cheque-books/*` routes answer.** The mirror of
`../cheques/plan-qt-received-cheques.md` on `apd_tra_type = 'P'` — same
row entity, same History dialog, same badge painting, its own `.ui` on
`NexDialog` (no `NexCrud*`). The screen is `issued_cheques_ui_mockup.png`.

## 1. File set

```
src/modules/accounts/cheques/
  issued_cheques.h / .cpp / .ui          // NexDialog: filters, summary strip, register, action bar, panel stack
  issued_cheque_panels.h / .cpp          // Print, Presented, Returned unpaid, Stop / Void, Replace — one class each
  cheque_register_entity.h               // SHARED with menu 51: ChequeRow + allowedActions(status, traType)
  cheque_history_dialog.h / .cpp         // SHARED
src/modules/accounts/payment/cheque_book_dialog.*   // REUSED: the Cheque books button opens it
src/core/api/endpoints/accounts_end_points.h   // ApiEndpoints::IssuedCheques::{List, Get, History, Print, Presented, Returned, Stop, Void, Replace}
```

## 2. The `.ui`

1. **Title strip:** company · branch · year · today · "POSTED AT ISSUE"
   chip (no posting-mode setting on this side).
2. **Filters:** ISSUED, NOT PRESENTED (HELD) · PRESENTED (CLEARED) ·
   RETURNED UNPAID (BOUNCED) · STOPPED / VOID (CANCELLED) · REPLACED — the P
   side's labels for the same statuses; cheque date range; own bank
   (dropdown 25); payee (dropdown 43); search (leaf / favouring / payment
   no). Grid params as menu 51's.
3. **Summary strip:** ISSUED, NOT PRESENTED · POST-DATED, OUT · PRESENTED
   THIS MONTH · RETURNED UNPAID · STOPPED / VOID · **LEAVES <from–to> · next
   <n> · <free> free** (from `/cheque-books/list` for the filtered bank;
   "all banks" shows the count of active books) · the tie "bank book
   uncleared = ISSUED, NOT PRESENTED + POST-DATED, OUT" · buckets FUTURE /
   DUE TODAY / OVERDUE / STALE.
4. **The register** `tblRegister` (`NexApiTableView` on the issued grid,
   multi-select for Print): Issued · Favouring · Leaf no · Our bank ·
   Cheque date · Due bucket · Amount · Status (P-side words) · Payment ·
   voucher (drill-through → `PaymentEntry::openFor(key)`) · Presented ·
   Printed (yes / no / count) · note. A VOID leaf row (amount —, no
   payment) is painted dim.
5. **Action bar:** Print F6 · Presented F7 · Returned unpaid F8 · Stop /
   Void F3 · Replace · Cheque books · History Ctrl+H; enabled per
   `allowedActions(status, 'P')`: ISSUED → Print / Presented / Stop /
   Replace; RETURNED → Replace; PRESENTED · STOPPED · VOID · REPLACED →
   History only (a VOID leaf has nothing else).
6. **Panel stack:**
   - **Print:** N selected · Format (combo of the bank's layouts; default
     the book's `chb_format`) · Date on cheque (blank = as on the row) · A/c
     payee (crossed) · Amount in words preview → the print runner (cheque
     layout) then `POST /issued-cheques/print` (counts + logs). A spoilt
     print → the Stop/Void panel with the row pre-selected.
   - **Presented:** bank date · remarks → `POST /issued-cheques/presented`
     (NO voucher; `av_recon_date` is the server's).
   - **Returned unpaid:** return date · reason (the bounce reasons setting)
     · charged by bank · (charge to party n/a — greyed with the reason) · a
     preview of the ChqBnc legs and the bills that reopen (from `/get`) →
     `POST /issued-cheques/returned`.
   - **Stop / Void:** STOP (had a payment — reversal voucher) vs VOID (a
     blank spoilt while printing — book + reason; or the selected HELD row
     spoilt) with the difference stated on the panel → `/stop` or `/void`.
   - **Replace:** book (default the old row's), new date, amount (defaults
     to the old, may differ), allocation note → `POST /issued-cheques/
     replace`; the answer's new leaf shown.
   - **Cheque books:** opens `ChequeBookDialog` (the payment's) — list per
     bank with used / void / free, open a new book, close one.
7. **Footer:** hint bar (Space selects · F6 · F7 · F8 · F3 · Ctrl+H · Enter
   opens the payment) · `btnClose` · `btnPrint`.

## 3. Behaviour

Open → defaults ISSUED + RETURNED ticked, cheque-date range wide enough to
include post-dated (year end + 3 months); selection → `/get` detail strip
(payment, bills it settled, the book, the replacement chain); action OK →
confirm naming the leaf and money → call → `hardReload()` → reselect.
Refusals from the server (a PRESENTED cheque returned, a second Presented,
an R row) shown as is. `LEAVES` figures refresh after every action that
consumes a leaf.

## 4. Tests

1. Status → buttons per the P map; VOID row → History only; 2. Print
payload carries every checked key + format; count increments on reload;
3. Presented on ISSUED → PRESENTED with the bank date, no voucher in the
detail; 4. Returned preview lists the reopened bills; 5. Stop vs Void
chooses the right endpoint; 6. Replace shows the new leaf and the old row
REPLACED; 7. LEAVES strip = the books' figures; 8. drill-through opens the
payment with its own scope.
