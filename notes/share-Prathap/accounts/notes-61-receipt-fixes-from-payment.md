# notes (61): three `/receipts` defects your payment module already fixed

2026-09-29. Your payment README ("Fixed while testing") lists three defects you found and fixed
in `/payments`, and says `/receipts` still has each one. I checked all three in `receipt/`. They
are still there, and the live Receipt screen (menu 99) calls all three routes. **The fix in
each case is the one you already wrote in `payment/`. Port it across.**

None of them can be worked around from the client. Details below.

## 1 · `/receipts/post` refuses a post-dated cheque (422) — the one that matters

- **Where:** `receipt/receipt-posting.service.ts:366` still calls `assertBooksReconcile`.
- **What goes wrong:** that check has no allowance for a post-dated voucher. The voucher is
  POSTED today, so the ledger counts it now. The bill's post-dated row counts only when it
  matures. So every receipt carrying a post-dated cheque answers `422 ACC_PARTY_OUT_OF_BALANCE`
  and rolls back.
- **Client side:** the screen shows your error message. There is nothing it could send to
  avoid it.
- **Fix:** what `payment/payment-posting.service.ts:367` does. Call
  `assertVoucherBooksReconcile` from `vouchers/voucher-books.helper.ts`, with `ledgerIds` = the
  party plus every ledger the post's vouchers moved.
- **Please also check the cancel.** `receipt-cancel.service.ts:267` and
  `payment-cancel.service.ts:198` both still use the plain `assertBooksReconcile`. A cancel
  probably nets to zero, but no test cancels a receipt or payment carrying an un-matured
  post-dated cheque. Please add one to each suite.

## 2 · `/receipts/adjacent` compares a DATE with a timestamp

- **Where:** `receipt/open-items.service.ts:915-922`.
  - `fromDate` and `toDate` are bound as `::timestamptz`.
  - So is `current.avhVoucherDate` in the row comparison.
- **What goes wrong:** `avh_voucher_date` is a `date` and the box runs in Asia/Kolkata. So the
  29th is read as 28 September 18:30Z, and it is compared with midnight UTC.
  - Prev and Next skip or misorder receipts on the same day as the one being walked from.
  - A one-day range matches nothing.
- **Client side:** Prev/Next sends only the voucher key (no date range). The server reads the
  date that goes wrong from the stored row, so the client cannot avoid it.
- **Fix:** what `payment/payment-open-items.service.ts:497-532` does. Bind all three as
  `::date`: `fromDate` and `toDate` sliced to `YYYY-MM-DD`, and the current row's date through
  `toDateString()`.
- `/vouchers/adjacent` (notes 52) is already right: it reads the walk key in SQL and binds its
  dates as `::date`. Only the receipt's walk has the defect.

## 3 · `/receipts/amend` answers with the old revision

- **Where:** `receipt/receipt-amend.service.ts:336-342` returns `...posted`, whose `header` was
  read inside the post. That is before the revision bump at `:305-316`, so it still carries the
  old `avhRevisionNo`.
- **What goes wrong:** a client that sends that header's revision back as its next
  `baseRevision` gets a 409, for an amend nobody else made.
- **Client side:** our screen is **not** affected today. After an amend it clears and reloads
  through `/get`. The React port or any other client would be affected, so it is still worth
  fixing.
- **Fix:** what `payment/payment-amend.service.ts:180-184` does. Put
  `header: await this.receiptService.toHeaderPayload(tx, after)` into the returned object, after
  the spread.

## Tests to add to the receipt suites

- a post-dated cheque receipt posts: 201, not 422;
- the same receipt cancels cleanly;
- `/adjacent` over two receipts on the same day, both directions, and with a one-day
  `fromDate`/`toDate`;
- `/amend` twice in a row, the second sending the first response's `avhRevisionNo` as
  `baseRevision`: it must not get a 409.
