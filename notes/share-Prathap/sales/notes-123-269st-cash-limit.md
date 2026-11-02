# Notes 123: 269ST, PAN / Form 60 and HSN checks are switched off (2026-10-15)

**STATUS:** §1–4 (269ST etc.) ON HOLD; §5 DONE, verified live 2026-10-15 7:30 PM: 90 sales rows loaded exactly as the seed (catalogue 2026-10-15.3, 202 rows); SALES_BILL_POSTED answered with its words. ON HOLD (user, 2026-10-15): "we can work on backend later — we can't make the customer wait at the counter." Leave the checks as they are for now. Server only. Follows notes 122 §4.

**The user's decision:** nothing may hold up a bill at the counter. When this is picked up later, it starts as a **warning only** (stl_enforce = WARN for the shipped 269ST row; PAN / Form 60 a warning too), never a refusal at the counter. Whether a company may switch it to REFUSE is decided then. The design below is kept for that day.

## What happens

Cash sales of **2,00,000 or more to one person** post without any refusal or warning. Two of them, from the sale bill suite on 15-10, with the ZT customer:
- bil01256: cash 2,12,400.00, posted;
- bil01271: cash over 2,00,000, posted.

The limit itself is set up correctly in `public.statutory_limits`:

| stl_code | value | enforce | applies to | from |
|---|---|---|---|---|
| CASH_TXN_LIMIT_269ST | 2,00,000.00 | REFUSE | ALL | 2017-04-01 |

## Why

In `modules/sales/bill/bill-lifecycle.service.ts`, step **"7 · the law"** (around line 700) is **commented out**:
- `statutory.assertCashLimit(...)`, the 269ST refuse / warn;
- `statutory.assertPanOrForm60(...)`, the PAN / Form 60 requirement on a large cash sale;
- `statutory.hsnDigits(...)`, the HSN-digits INFO warning.

The query that adds up the party's cash for the day still runs, but its result is never used. Nothing else calls these three: receipts, sale-order advances and sale returns never had them. The file was last changed on 10-12, so the checks have been off since at least then.

If they were switched off on purpose (to unblock something during testing), please say what blocked them. Otherwise:

## First: walk-in "cash customers" (the user's point, 15-10)

In a supermarket almost every bill goes to the one walk-in party (`sales.default_customer_id`, "Cash customer"). Adding up **that party's** cash for the day would cross 2,00,000 within hours and refuse ordinary shoppers. That must never happen.

269ST limits cash from **one person**. A walk-in party is not one person; it stands for every shopper who gave no name. So:

- **Walk-in bill:** count **this bill's cash only** (per transaction). Refuse only when one bill takes 2,00,000 or more in cash. The commented code already does this (`isWalkIn ? cash : cash + cashToday`): keep it.
- **Named party** (a real customer, or a walk-in where the cashier typed the buyer's PAN): add the day's cash for that party, as the commented code does.
- **"Walk-in" must be the same answer party-context gives** (`isWalkIn`, which the Qt bill already uses), so the screen and the server never disagree. If a store keeps more than one cash-customer party (one per counter, say), each must count as walk-in. Please say whether party-context already covers that (a customer flag?) or only the one `sales.default_customer_id`.
- **The same rule applies to §2 below:** a cash receipt or a cash advance against a walk-in party counts alone, never added up across the day.

## Ask

### 1 · Switch the three checks back on in bill posting

As the commented code already reads:
- **269ST:** cash tendered on this bill, plus this party's cash on POSTED bills of the same date (a walk-in counts this bill alone). At or above the limit:
  - REFUSE → `SALES_CASH_LIMIT` refusal on `tenders`, with the `statutory` reference;
  - WARN → warning.

  The section bites **at** the limit, not above it.
- **PAN / Form 60:** above its threshold, no `sbCustPan` and no `sbForm60Ref` → `SALES_PAN_REQUIRED` on `sbCustPan`.
- **HSN digits:** INFO only, never blocks.

They must run on **both** paths: `/bills/validate` (so the screen's Validation popup shows them before posting) and `/bills/create` with `post: true` / `/bills/post`.

### 2 · The other ways cash comes in from the same person

269ST counts **all** cash received from one person in a day / for one transaction / for one event, not only bills. For a **named** party, please count, and check against the same limit (a walk-in counts each document alone, as above):
- **receipts** (`/receipts/create`) with a CASH tender from that party;
- **sale-order advances** taken in cash;
- **temp-credit receipts** (TEMP_CR) settled in cash.

So the day's total for a **named** party is: cash on posted bills + cash receipts + cash advances. Refunds and returns do not reduce it; 269ST is about cash *received*.

### 3 · Words for the screens (notes 120 catalogue)

Please add these rows, so the box shows plain words with the law named in the grey line:

| Code | Title | Line | Do |
|---|---|---|---|
| `SALES_CASH_LIMIT` (named party) | Cash limit crossed | Cash from this customer today is ₹2,00,000 or more. | Take the rest by bank, card or UPI. |
| `SALES_CASH_LIMIT` (walk-in, variant) | Cash limit crossed | One bill cannot take ₹2,00,000 or more in cash. | Take the rest by bank, card or UPI. |
| `SALES_PAN_REQUIRED` | PAN needed | A cash sale this big needs the customer's PAN. | Enter the PAN, or a Form 60 reference. |

### 4 · Test

The sale bill suite (`tests/salebill/run-full.sh`, area D) already expects the 2,12,400 cash bill to be refused with the Validation popup naming 269ST and its limit, and to leave no draft. Those three checks fail today and should pass once this is on.

## 5 · Plain words for the sales codes: NOT on hold

This part is separate from the 269ST hold. Please load it once the user approves the wording.

The notes-120 catalogue has only one sales row today (`SALES_DAY_CLOSED`). The other **89** sales codes (`SALES_…`, the sales `GST_…` codes, `HSN_DIGITS`) show the server's developer sentence under a plain title.

- **Wording (for the user to approve first):** `Shared/Prathap/messages/sales-error-wording.md` REV 1.
- **Seed rows:** `Shared/Prathap/messages/sales-error-messages-seed.json`: 90 rows for 89 codes (SALES_DAY_CLOSED is already in), `{ code, variant, lang, title, line, todo }` as in notes 120. Load them into `fixed.error_message` after the approval; the file's version is `2026-10-15.sales-1`.
- **One variant needs a field on the error:** `SALES_CASH_LIMIT` with `isWalkIn=true`. When 269ST comes back (§1, on hold), put `isWalkIn` on that refusal's detail so the walk-in wording ("One bill cannot take ₹2,00,000 or more in cash") is picked.
- **Seven codes are the app's own mistakes** (`SALES_FIELD_REQUIRED`, `SALES_LINE_FIELD_REQUIRED`, `SALES_LINE_NO_DUPLICATE`, `SALES_LINE_NOT_FOUND`, `SALES_SALESMAN_INVALID`, `SALES_STOCK_QTY_MISMATCH`, `SALES_AMOUNT_MISMATCH`). They take the shared "Something went wrong · The app sent a wrong request" row. A `SALES_*` prefix row would cover any code added later.
