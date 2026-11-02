# Sales error messages, short and simple (REV 1, 2026-10-15)

**STATUS:** sent by the user and LOADED 2026-10-15 7:30 PM (catalogue 2026-10-15.3), verified row for row. The two questions below were not answered: the wording went in as drafted. Prathap loads the rows into `fixed.error_message` (notes 120 catalogue; notes 123 §5). Qt shows them from the server's `user` block.

Same rules as `till-error-wording.md`:
- a title of 2–4 words, one short line, then "Do:" and one step;
- everyday words;
- the server's text stays small and grey underneath, with the bill number, line and amounts in it.

The catalogue rows are fixed text, so these lines name no figures. The grey line carries them ("Line 3: rate 90 is below the minimum 100").

**Shared rows** (as in `counter-error-wording.md`):
- **[APP]:** Something went wrong · The app sent a wrong request. · Do: Send a photo of this to support.
- **[SETUP]:** Setup missing · Something is not set up. · Do: Call manager.

Of the 90 `SALES_…` / sales `GST_…` codes, only `SALES_DAY_CLOSED` is in the catalogue today ("Day closed · Today is closed for billing. · Call manager."). Everything below is new.

## 1 · The document's state

| Code | Title | Line | Do |
|---|---|---|---|
| `SALES_BILL_POSTED` | Already posted | This bill is already posted. | To change it, use Amend or Cancel. |
| `SALES_BILL_CANCELLED` | Bill cancelled | This bill is cancelled. | Make a new bill. |
| `SALES_DOC_POSTED` | Already posted | This document is already posted. | To change it, use Amend or Cancel. |
| `SALES_DOC_CANCELLED` | Cancelled | This document is cancelled. | Make a new one. |
| `SALES_DOC_NOT_DRAFT` | Not allowed now | This document is not in the right state for this. | Refresh and check its status. |
| `SALES_NOT_FOUND` | Not found | This document is not there now. | Refresh the list. |
| `SALES_REVISION_STALE` | Changed meanwhile | Someone changed this document after you opened it. | Close it, open it again, and redo your change. |
| `SALES_AMEND_OFF` | Amend is off | Changing a posted document is switched off. | Cancel it and make a new one, or call manager. |

## 2 · Rights

| Code | Title | Line | Do |
|---|---|---|---|
| `SALES_RIGHT_VIEW` | Not allowed | You have no right to open this. | Call manager. |
| `SALES_RIGHT_CREATE` | Not allowed | You have no right to make new ones. | Call manager. |
| `SALES_RIGHT_EDIT` | Not allowed | You have no right to change this. | Call manager. |
| `SALES_RIGHT_DELETE` | Not allowed | You have no right to delete this. | Call manager. |
| `SALES_RIGHT_PRINT` | Not allowed | You have no right to print this. | Call manager. |
| `SALES_RIGHT_EXPORT` | Not allowed | You have no right to export this. | Call manager. |
| `SALES_RIGHT_POST` | Not allowed | You have no right to post. | Save as draft, or call manager. |
| `SALES_RIGHT_CANCEL` | Not allowed | You have no right to cancel. | Call manager. |
| `SALES_RIGHT_AMEND` | Not allowed | You have no right to amend. | Call manager. |
| `SALES_RIGHT_OVERRIDE` | Not allowed | You have no right to override this warning. | Call supervisor. |
| `SALES_RETENDER_RIGHT` | Not allowed | You have no right to change how a bill was paid. | Call supervisor. |

## 3 · Something else holds this document

| Code | Title | Line | Do |
|---|---|---|---|
| `SALES_RETURN_LOCKS_BILL` | Return exists | A sale return is made against this bill. | Cancel the return first. |
| `SALES_ALLOCATION_LOCKS_BILL` | Payment linked | A receipt or credit note is set against this bill. | Cancel that first. |
| `SALES_CN_APPLIED` | Credit note used | This credit note is already used or refunded. | Make a new document to correct it. |
| `SALES_BILL_PDC_MOVED` | Cheque moved | This bill's cheque has already gone to the bank. | Handle it in Received Cheques. |
| `SALES_RETENDER_PDC_MOVED` | Cheque moved | This cheque has already gone to the bank. | A bounce is done in Received Cheques. |
| `SALES_CHEQUE_LOCKED_IN_REGISTER` | Cheque moved | This cheque is already deposited or settled. | Settle it in Received Cheques first. |

## 4 · E-invoice and e-way bill

| Code | Title | Line | Do |
|---|---|---|---|
| `SALES_IRN_LIVE` | E-invoice live | This bill has a live e-invoice. | Cancel the e-invoice first, or make a credit note. |
| `SALES_EWB_LIVE` | E-way bill live | This bill has a live e-way bill. | Cancel the e-way bill first. |
| `GST_DECLARED_LOCKED` | Already declared | Transport details are already sent to GST. | Change the vehicle on the e-way bill instead. |
| `SALES_IRN_WINDOW_PASSED` | Too late | The time to cancel this e-invoice has passed. | Make a credit note instead. |
| `SALES_EWB_WINDOW_PASSED` | Too late | The time to cancel this e-way bill has passed. | Call manager. |
| `GST_IRN_CANCEL_FAILED` | Cancel at portal | The e-invoice could not be cancelled from here. | Cancel it on the GST portal first. |
| `GST_EWB_EXPIRED` | E-way bill expired | This e-way bill has expired. | Make a new e-way bill. |
| `GST_CREDIT_NOTE_CUTOFF` | Too late | The GST date for a credit note on this bill has passed. | Call manager. |
| `SALES_EWAY_TRANSPORT_MISSING` | Transport needed | E-way bill needs the transport details. | Fill in the vehicle and transport details. |

## 5 · Orders, challans, returns

| Code | Title | Line | Do |
|---|---|---|---|
| `SALES_ORDER_DELIVERED` | Already delivered | Some lines of this order are delivered. | Cancel only the open lines. |
| `SALES_ORDER_LINE_DELIVERED` | Already delivered | This line is already delivered. | Keep the delivered quantity. |
| `SALES_ORDER_CONFIRMED` | Order confirmed | A confirmed order is not deleted. | Cancel it instead. |
| `SALES_ORDER_NOT_OPEN` | Order not open | This order cannot be billed now. | Check the order's status. |
| `SALES_ORDER_LINE_OVER` | Too much | More than the order's pending quantity. | Bill the pending quantity or less. |
| `SALES_RESERVE_SHORT` | Not enough stock | Not enough stock to keep for this order. | Lower the quantity. |
| `SALES_DC_PURPOSE_NOT_ALLOWED` | Not allowed | This challan type is not used here. | Pick another type, or call manager. |
| `SALES_DC_REQUIRES_ORDER` | Order needed | This challan needs a sale order. | Pick the order first. |
| `SALES_DC_LINE_OVER_ORDER` | Too much | More than the order line allows. | Lower the quantity. |
| `SALES_DC_LINE_OVER` | Too much | More than the challan line has left. | Lower the quantity. |
| `SALES_DC_BILLED` | Already billed | A bill is made against this challan. | Cancel the bill first. |
| `SALES_DC_RETURNED` | Goods returned | Goods are returned against this challan. | Cancel that return first. |
| `SALES_DCR_OVER_OPEN` | Too much | More than the challan has open. | Lower the quantity. |
| `SALES_DCR_DC_NOT_POSTED` | Challan not posted | The challan is not posted. | Post the challan first. |
| `SALES_RETURN_OVER_QTY` | Too much | More than was sold on the bill. | Lower the return quantity. |
| `SALES_RETURN_BILL_NOT_POSTED` | Bill not posted | The bill is not posted. | Post the bill first. |
| `SALES_RETURN_ITEM_NOT_ALLOWED` | No returns | This item cannot be returned. | Remove it from the return. |
| `SALES_FREE_RETURN_OFF` | No free returns | Free items cannot be returned. | Remove the free line. |
| `SALES_RETURN_WINDOW` | Too late | The time to return this bill has passed. | Call manager. |

## 6 · Dates and periods

| Code | Title | Line | Do |
|---|---|---|---|
| `SALES_YEAR_LOCKED` | Year closed | This financial year is closed. | Use a date in the open year. |
| `SALES_BACKDATE` | Old date | This date is too far back. | Use today's date, or call manager. |

## 7 · Customer, credit, price, payment

| Code | Title | Line | Do |
|---|---|---|---|
| `SALES_CUSTOMER_REQUIRED` | Customer needed | Pick a customer for this bill. | Pick the customer. |
| `SALES_CREDIT_LIMIT` | Credit limit crossed | This customer's credit limit is crossed. | Take payment now, or call manager. |
| `SALES_RATE_BELOW_MIN` | Rate too low | The rate is below the lowest allowed. | Raise the rate, or call supervisor. |
| `SALES_DISC_CAP` | Discount too big | The discount is more than allowed. | Lower the discount, or call supervisor. |
| `SALES_TENDER_DAILY_LIMIT` | Limit crossed | This payment type's daily limit is crossed. | Take it another way. |
| `SALES_TENDER_MIN_MAX` | Wrong amount | Amount is too small or too big for this payment type. | Change the amount, or pay another way. |
| `SALES_LOYALTY_CAP` | Too many points | More points than allowed on this bill. | Use fewer points. |
| `SALES_PROMO_NOT_LIVE` | Offer ended | This offer is not on now. | Remove the offer. |
| `SALES_CHARGE_OVER_CARRY` | Charge too big | This charge is more than was collected before. | Lower it, or call supervisor. |
| `SALES_STOCK_NEGATIVE` | Not enough stock | Not enough stock for this quantity. | Lower the quantity. |
| `SALES_DELIVERY_ORDER` | Wrong step | Delivery steps must go in order. | Do the next step first. |
| `SALES_CASH_LIMIT` (named party) | Cash limit crossed | Cash from this customer today is ₹2,00,000 or more. | Take the rest by bank, card or UPI. |
| `SALES_CASH_LIMIT` (walk-in, variant) | Cash limit crossed | One bill cannot take ₹2,00,000 or more in cash. | Take the rest by bank, card or UPI. |
| `SALES_PAN_REQUIRED` | PAN needed | A cash sale this big needs the customer's PAN. | Enter the PAN, or a Form 60 reference. |
| `HSN_DIGITS` | HSN too short | An HSN code is too short. | Check the item's HSN. |

The `SALES_CASH_LIMIT` / `SALES_PAN_REQUIRED` rows are for when notes 123 is picked up (on hold; warning only).

## 8 · Temporary credit

| Code | Title | Line | Do |
|---|---|---|---|
| `SALES_TEMP_CREDIT_DETAILS_MISSING` | Details needed | Temporary credit needs the person's name and mobile. | Fill in name and mobile. |
| `SALES_TEMP_CREDIT_DAYS` | Too many days | Temporary credit is for fewer days. | Pick fewer days. |
| `SALES_TEMP_CREDIT_AMOUNT` | Amount too big | Temporary credit amount is too big. | Lower the amount. |
| `SALES_TEMP_CREDIT_OPEN` | Credit already open | This mobile already has temporary credit. | Collect it first, or call supervisor. |

## 9 · Cheques

| Code | Title | Line | Do |
|---|---|---|---|
| `SALES_CHEQUE_NUMBER_MISSING` | Cheque number | The cheque number is missing. | Fill in the cheque number. |
| `SALES_CHEQUE_DATE_MISSING` | Cheque date | The cheque date is missing. | Fill in the cheque date. |
| `SALES_CHEQUE_BACKDATED` | Wrong date | The cheque date is before the bill date. | Check the cheque date. |
| `SALES_CHEQUE_STALE` | Old cheque | This cheque is more than 3 months old. | The bank will not take it: ask for a new one. |
| `SALES_CHEQUE_DATE_TOO_FAR` | Wrong date | The cheque date is more than a year away. | Check the year on the cheque. |
| `SALES_CHEQUE_REPEATED` | Same cheque twice | The same cheque is entered twice. | Remove one. |
| `SALES_CHEQUE_ALREADY_REGISTERED` | Cheque already taken | This cheque is already taken on another bill. | Check the cheque number. |

## 10 · Re-tender

| Code | Title | Line | Do |
|---|---|---|---|
| `SALES_RETENDER_AMOUNT_MISMATCH` | Amounts don't match | New payments must add up to what you remove. | Check the amounts. |

## 11 · The app's own mistakes → [APP]

These mean the app sent something wrong, not that the user did. All take the [APP] row:

`SALES_FIELD_REQUIRED`, `SALES_LINE_FIELD_REQUIRED`, `SALES_LINE_NO_DUPLICATE`, `SALES_LINE_NOT_FOUND`, `SALES_SALESMAN_INVALID`, `SALES_STOCK_QTY_MISMATCH`, `SALES_AMOUNT_MISMATCH`.

## 12 · Setup → [SETUP]

`SALES_DEVICE_UNREGISTERED` → **This PC is not set up** · This PC is not registered. · Do: Call manager.

## Questions

1. **`SALES_CREDIT_LIMIT`:** at the counter, is "Take payment now, or call manager" right? Or should the cashier be told to make it a cash bill?
2. **`SALES_STOCK_NEGATIVE`:** some stores allow negative stock with a warning. Is "Lower the quantity" right where it refuses?
