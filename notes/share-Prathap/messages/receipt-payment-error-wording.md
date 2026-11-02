# Receipt, payment and cheque error messages, short and simple (REV 1, 2026-10-15)

**STATUS:** draft for the user's review. Once approved, Prathap adds the missing codes and loads the rows (notes 124).

Same rules as the till / sales wording:
- a title of 2–4 words, one short line, then "Do:" and one step;
- everyday words;
- the server's sentence (with the bill, cheque and voucher numbers and amounts) stays small and grey underneath.

**Shared rows:**
- **[APP]:** Something went wrong · The app sent a wrong request. · Do: Send a photo of this to support.
- **[SETUP]:** Setup missing · Something is not set up. · Do: Call manager.

**What exists today:**
- The **Voucher Register** (JV, Contra, Receipt / Payment vouchers), **issued cheques** and **cheque books** raise **61 `VCH_…` codes**, none of them worded yet. That's Part A.
- The **Receipt (99)**, **Payment (100)** and **Received Cheques (51)** screens raise about **190 refusals with no code at all**, only `{field, message}`. They can't be worded until they carry a code. Part B maps each one to a code, reusing a `VCH_…` code where it means the same and proposing new `ACC_…` / `CHQ_…` codes for the rest.

---

## Part A · The `VCH_…` codes (they exist; wording only)

### Rights

| Code | Title | Line | Do |
|---|---|---|---|
| `VCH_RIGHT_VIEW` | Not allowed | You have no right to open this. | Call manager. |
| `VCH_RIGHT_CREATE` | Not allowed | You have no right to make new ones. | Call manager. |
| `VCH_RIGHT_EDIT` | Not allowed | You have no right to change this. | Call manager. |
| `VCH_RIGHT_DELETE` | Not allowed | You have no right to delete this. | Call manager. |
| `VCH_RIGHT_POST` | Not allowed | You have no right to post. | Save as draft, or call manager. |
| `VCH_RIGHT_CANCEL` | Not allowed | You have no right to cancel. | Call manager. |
| `VCH_RIGHT_AMEND` | Not allowed | You have no right to amend. | Call manager. |
| `VCH_RIGHT_OVERRIDE` | Not allowed | You have no right to override this warning. | Call supervisor. |

### The voucher's own state

| Code | Title | Line | Do |
|---|---|---|---|
| `VCH_NOT_FOUND` | Not found | This voucher is not there now. | Refresh the list. |
| `VCH_NOT_DRAFT` | Already posted | Only a draft can be changed. | Cancel it and enter it again. |
| `VCH_POSTED` | Already posted | This voucher is already posted. | Cancel it and enter it again. |
| `VCH_NOT_POSTED` | Still a draft | This is still a draft. | Delete the draft instead. |
| `VCH_CANCELLED` | Already cancelled | This voucher is already cancelled. | Nothing to do. |
| `VCH_UNBALANCED` | Doesn't balance | Debit and credit are not equal. | Check the amounts. |
| `VCH_NO_LINES` | No lines | The voucher has no lines. | Add the lines. |
| `VCH_LINE_AMOUNT` | Wrong amount | A line's amount is wrong. | Check each line's amount. |

### Dates and periods

| Code | Title | Line | Do |
|---|---|---|---|
| `VCH_PERIOD_LOCKED` | Books locked | The books for this date are locked. | Use a later date, or call manager. |
| `VCH_YEAR_CLOSED` | Year closed | This financial year is closed. | Use a date in the open year. |
| `VCH_DATE_OUTSIDE_YEAR` | Wrong year | The date is not in this financial year. | Check the date. |
| `VCH_BACKDATED` | Old date | The date is before today. | Use today's date, or call manager. |

### Accounts and party

| Code | Title | Line | Do |
|---|---|---|---|
| `VCH_LEDGER_SIDE` | Wrong side | This account cannot be on that side. | Check debit and credit on the line. |
| `VCH_LEDGER_INACTIVE` | Account switched off | This account is switched off. | Pick another account, or call manager. |
| `VCH_LEDGER_NOT_FOUND` | Account not found | This account is not there. | Pick the account again. |
| `VCH_INSTRUMENT_LEDGER` | Wrong account | A cheque or card needs a bank or payment account. | Pick the right account. |
| `VCH_PARTY_MODE` | Party needed | This voucher needs one party. | Pick the party. |
| `VCH_PARTY_NOT_FOUND` | Party not found | This party is not there. | Pick the party again. |
| `VCH_DUP_DOC_REFNO` | Number used | This reference number is already used. | Check the reference number. |
| `VCH_DOC_REFNO_REQUIRED` | Reference needed | The bill or reference number is missing. | Fill in the reference number. |

### Bills

| Code | Title | Line | Do |
|---|---|---|---|
| `VCH_BILLWISE_SHORT` | Bills not matched | The amount is not fully set against bills. | Set the rest against bills, or as advance if allowed. |
| `VCH_BILL_OVERSPENT` | Too much on a bill | More than the bill's pending amount. | Lower the amount on that bill. |
| `VCH_BILL_NOT_FOUND` | Bill not found | This bill is not there now. | Refresh the bills list. |
| `VCH_BILL_WRONG_PARTY` | Wrong party | This bill belongs to another party. | Pick this party's bills. |
| `VCH_BILL_WRONG_SIDE` | Wrong bill | This bill cannot be settled here. | Pick another bill. |
| `VCH_ALLOCATED_ELSEWHERE` | Already used | A receipt or payment is set against this. | Cancel that first. |
| `VCH_ADVANCE_SPENT` | Advance used | The advance from this is already used. | Reverse that use first. |

### GST, TDS, e-invoice

| Code | Title | Line | Do |
|---|---|---|---|
| `VCH_GST_RATE_MISSING` | GST rate needed | The GST rate is missing. | Pick the rate on the line. |
| `VCH_GST_LEDGER_TYPED` | Wrong account | GST accounts fill in by themselves. | Remove the GST account line. |
| `VCH_GST_NOT_ALLOWED` | No GST here | This voucher cannot carry GST. | Remove the GST. |
| `VCH_TDS_RATE_MISSING` | TDS not set | This party's TDS is not set up. | Call manager. |
| `VCH_TDS_BELOW_THRESHOLD` | No TDS needed | The amount is below the TDS limit. | Remove the TDS. |
| `VCH_TDS_DEPOSITED` | TDS already paid | This TDS is already paid to the government. | Call accounts: it needs a TDS return correction. |
| `VCH_IRN_LIVE` | E-invoice live | This voucher has a live e-invoice. | Cancel the e-invoice first. |

### Cheques, banks, cheque books

| Code | Title | Line | Do |
|---|---|---|---|
| `VCH_CHEQUE_DETAILS` | Cheque details | Cheque number or date is missing or wrong. | Fill in the cheque details. |
| `VCH_CHEQUE_MOVED` | Cheque moved | This cheque has already gone to the bank. | Handle it in Cheques first. |
| `VCH_CHEQUE_NOT_FOUND` | Cheque not found | This cheque is not there now. | Refresh the list. |
| `VCH_CHEQUE_STATE` | Not allowed now | This cheque is not in the right state for this. | Refresh and check its status. |
| `VCH_INSTRUMENT_NOT_ALLOWED` | Not allowed | A cheque or card cannot be on this line. | Move it to the bank line. |
| `VCH_INSTRUMENT_TENDER` | Wrong payment type | This payment type does not fit the line. | Pick another payment type. |
| `VCH_BANK_REQUIRED` | Bank needed | Pick the bank account. | Pick the bank. |
| `VCH_BOOK_REQUIRED` | Cheque book needed | Pick the cheque book. | Pick the book. |
| `VCH_BOOK_FINISHED` | Book finished | This cheque book has no leaves left. | Start a new cheque book. |
| `VCH_BOOK_BANK` | Wrong book | This cheque book is for another bank. | Pick this bank's book. |
| `VCH_BOOK_NOT_FOUND` | Book not found | This cheque book is not there. | Refresh the list. |
| `VCH_BOOK_OVERLAP` | Leaves clash | These cheque numbers are in another book. | Check the leaf numbers. |
| `VCH_BOOK_INVALID` | Wrong numbers | The cheque leaf numbers are wrong. | Check the first and last leaf. |

### The app's own mistakes / setup

| Code | Row |
|---|---|
| `VCH_INVALID`, `VCH_ALLOCATION_LINE`, `VCH_TYPE_NOT_REGISTER`, `VCH_TYPE_INVENTORY` | [APP] |
| `VCH_GST_LEDGER_UNMAPPED`, `VCH_TDS_UNMAPPED` | [SETUP] |

---

## Part B · Receipt (99), Payment (100), Received Cheques (51): codes to add

Each refusal today is a bare sentence. The table names the code it should carry. ✱ = a new code, with its wording. A `VCH_…` code = reuse it (worded in Part A).

### Receipt and Payment (both screens raise these)

| Server sentence today (shortened) | Code | Title · Line · Do (new codes only) |
|---|---|---|
| Receipt / Payment not found | `VCH_NOT_FOUND` | |
| X is POSTED / CANCELLED … only a DRAFT may be posted / changed | `VCH_NOT_DRAFT` / `VCH_CANCELLED` | |
| A cancelled receipt / payment is closed to edits | `VCH_CANCELLED` | |
| … is out by X (posting or its reversal) | `VCH_UNBALANCED` | |
| Accounting year X is CLOSED / was locked on … | `VCH_YEAR_CLOSED` / `VCH_PERIOD_LOCKED` | |
| X falls in year Y, but the receipt says Z | `VCH_DATE_OUTSIDE_YEAR` | |
| "Party" is inactive and cannot receive / be paid; a line ledger is inactive | `VCH_LEDGER_INACTIVE` | |
| Bill X belongs to another party / company | `VCH_BILL_WRONG_PARTY` | |
| Bill X is a … bill / sits on the … side | `VCH_BILL_WRONG_SIDE` | |
| Bill X has P pending, but … is allocated | `VCH_BILL_OVERSPENT` | |
| Cheque X is DEPOSITED / … once an instrument has left … | `VCH_CHEQUE_MOVED` | |
| The on-account balance / advance from X has already been used | `VCH_ADVANCE_SPENT` | |
| The TDS on X was deposited under challan … | `VCH_TDS_DEPOSITED` | |
| A tender row must carry more than zero; an other-ledger line must carry more than zero | `VCH_LINE_AMOUNT` | |
| A cheque needs … (number / date / bank) | `VCH_CHEQUE_DETAILS` | |
| Bill moved while this receipt was being entered | ✱ `ACC_BILL_MOVED` | Changed meanwhile · This bill changed while you were entering. · Refresh the bills and try again. |
| Credit X has P left, but … | ✱ `ACC_CREDIT_OVERSPENT` | Too much credit · More than the credit note has left. · Lower the amount. |
| Credit moved while this receipt was being entered | ✱ `ACC_CREDIT_MOVED` | Changed meanwhile · This credit changed while you were entering. · Refresh and try again. |
| Editing a posted receipt / payment is switched off | ✱ `ACC_AMEND_OFF` | Amend is off · Changing a posted voucher is switched off. · Cancel it and enter it again. |
| This is a post-dated cheque voucher, not a receipt / payment | ✱ `ACC_PDC_VOUCHER` | Open the main voucher · This is the cheque's own voucher. · Open the receipt or payment it belongs to. |
| X was received from / paid to "P", and an amend cannot change it | ✱ `ACC_PARTY_LOCKED` | Party can't change · A posted voucher's party can't change. · Cancel it and enter it again. |
| This receipt / payment has been amended since you opened it | ✱ `ACC_REVISION_STALE` | Changed meanwhile · Someone changed this after you opened it. · Close it, open it again, and redo your change. |
| A receipt / payment with no instruments is a journal | ✱ `ACC_NO_TENDER` | Payment type needed · Add how the money was paid. · Add a cash, cheque or UPI line. |
| Tender "X" is inactive / of another company / not at this branch / of another type | ✱ `ACC_TENDER_NOT_USABLE` | Payment type off · This payment type can't be used here. · Pick another payment type. |
| Received / Paid X less change Y is … | ✱ `ACC_CHANGE_WRONG` | Change wrong · Received minus change does not match. · Check received and change. |
| Tender "X" does not allow its ledger to be changed | ✱ `ACC_TENDER_LEDGER_FIXED` | Account fixed · This payment type's account can't change. · Leave the account as it is. |
| "X" is the party of this receipt and cannot also be a line | ✱ `ACC_PARTY_AS_LINE` | Same account twice · The party can't also be a line. · Remove that line. |
| Writing off / back X on a bill needs an approver | ✱ `ACC_WRITEOFF_APPROVER` | Approval needed · Writing off needs an approver. · Pick the approver. |
| ON_CLEARING moves allocation into the cheque register | ✱ `ACC_PDC_ALLOCATION` | Bills later · Bills are matched when the cheque clears. · Match them in Cheques after clearing. |
| (fields) cannot be changed on a posted receipt / payment | ✱ `ACC_POSTED_FIELDS` | Can't change that · Money details of a posted voucher can't change. · Use Amend, or cancel and enter it again. |
| receipt_salesman_mandatory: who collected it | ✱ `ACC_COLLECTOR_REQUIRED` | Who collected? · Pick who collected this money. · Pick the person. |
| payment_salesman_mandatory: who made it | ✱ `ACC_PAYER_REQUIRED` | Who paid? · Pick who made this payment. · Pick the person. |
| X has no voucher partitions | ✱ `ACC_YEAR_NOT_SET_UP` | [SETUP] |
| Receipt / Payment voucher type is not configured | ✱ `ACC_TYPE_NOT_SET_UP` | [SETUP] |
| Nothing maps "ROLE" to a ledger (incl. BANK_CHARGES, TDS_PAYABLE) | ✱ `ACC_ROLE_UNMAPPED` | [SETUP] |
| Row number appears twice; give the line a role or a ledger; ROLE is not a role…; ROLE posts DR not CR; ROLE cannot settle a bill; bank-charge figures disagree; onAccount is required; the allocation names a voucher with no instrument; a draft with legs; one person, not an array; "X" is not an accounting year | ✱ `ACC_INVALID` | [APP] |

### Payment only

| Server sentence today (shortened) | Code | Title · Line · Do (new codes only) |
|---|---|---|
| A cheque row must name the book its leaf comes from | `VCH_BOOK_REQUIRED` | |
| Book X is used up / closed; Book X has no leaf left | `VCH_BOOK_FINISHED` | |
| A cheque we issue is dated the payment or later | ✱ `ACC_CHEQUE_BACKDATED` | Wrong date · Our cheque can't be dated before the payment. · Use the payment date or later. |
| Cheque book X belongs to another company / is not usable at this branch | ✱ `ACC_BOOK_NOT_USABLE` | Wrong book · This cheque book can't be used here. · Pick another book. |
| "X" is a cash / bank ledger: moving money between them … | ✱ `ACC_PARTY_IS_CASH` | Wrong party · A cash or bank account can't be paid here. · Use a Contra voucher instead. |
| X is not TDS-applicable / is TDS-applicable under … | `VCH_TDS_RATE_MISSING` | |
| The client keyed TDS of X, but SECTION @ … | ✱ `ACC_TDS_MISMATCH` | TDS differs · The TDS typed differs from the worked-out TDS. · Use the worked-out TDS. |
| Tender row X is already settled with the bank | ✱ `ACC_TENDER_SETTLED` | Already settled · A payment line is already settled with the bank. · Undo the settlement first. |
| A cheque we issue takes its number from the book (leave tdRefNo empty) | `ACC_INVALID` | |

### Received Cheques (51)

| Server sentence today (shortened) | Code | Title · Line · Do (new codes only) |
|---|---|---|
| Cheque not found | `VCH_CHEQUE_NOT_FOUND` | |
| Bank ledger not found | `VCH_LEDGER_NOT_FOUND` | |
| X is STATUS (wrong state for this action); posted ON_CLEARING / ON_RECEIPT, so … | `VCH_CHEQUE_STATE` | |
| "X" is inactive and cannot take a deposit | `VCH_LEDGER_INACTIVE` | |
| … is out by X | `VCH_UNBALANCED` | |
| A cheque has to be for more than nothing | `VCH_LINE_AMOUNT` | |
| The allocations come to X but the cheque is Y; it settled X against bill … | ✱ `CHQ_ALLOCATION_WRONG` | Bills don't match · The bills don't add up to the cheque. · Check the bill amounts. |
| A bounce has to say why the bank sent it back | ✱ `CHQ_REASON_REQUIRED` | Reason needed · Say why the bank returned it. · Pick the reason. |
| X names no bank ledger | ✱ `CHQ_BANK_MISSING` | Bank needed · This cheque has no bank account. · Pick the bank. |
| X has already bounced once | ✱ `CHQ_BOUNCED_ONCE` | Bounced before · This cheque bounced once. · Re-present it, or ask for a new cheque. |
| The date is more than three months old | ✱ `CHQ_STALE` | Old cheque · This cheque is more than 3 months old. · The bank will not take it: ask for a new one. |
| The date is more than a year away | ✱ `CHQ_DATE_TOO_FAR` | Wrong date · The cheque date is more than a year away. · Check the year on the cheque. |
| … on or after DATE (too early for this action) | ✱ `CHQ_DATE_TOO_EARLY` | Wrong date · This date is too early for this cheque. · Use a later date. |
| … cannot be dated DATE: that is in the future | ✱ `CHQ_DATE_FUTURE` | Wrong date · This date is in the future. · Use today or earlier. |
| "X" is a … ledger (not a bank) | ✱ `CHQ_BANK_WRONG` | Wrong account · Deposit only into a bank account. · Pick a bank account. |
| Already done | ✱ `CHQ_ALREADY_DONE` | Already done · Someone already did this. · Refresh. |
| X is an ISSUED cheque, not a received one; no / deleted tender row; every leg came to zero | `ACC_INVALID` | |
| Voucher type is not configured | `ACC_TYPE_NOT_SET_UP` | |

## Questions

1. **`VCH_BACKDATED` / "Old date":** for receipts at the counter, is "Use today's date, or call manager" right?
2. **`VCH_TDS_DEPOSITED`:** "Call accounts: it needs a TDS return correction". Do your users know what that means, or should it just say "Call accounts"?
