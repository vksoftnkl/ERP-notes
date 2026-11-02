# Sale Bill — live end-to-end test, 2026-09-24

Suite: `tests/salebill/run-full.sh` + `test_full.cpp` in the Qt repo. It drives the
bill screen through its own code, posts for real against its own ZT- data
(customers ZT-CUST-CREDIT / ZT-CUST-CASH, items ZT-ITEM-A 18% / ZT-ITEM-B 5%), and
checks every effect in the database, not in the response.

Final run: **123 pass, 8 fail, 5 skipped.** Every remaining failure is below.
Three client defects the run found were fixed in the Qt client the same day (listed
at the end, for reference).

Each item is written against the API contract, with the file:line the behaviour
comes from in the server source.

---

## 1. The item-price lookup returns 0% GST for every item

`master-lookup/lookups/item-price.lookup.ts:272` still reads `prisma.itemTaxMaster`
(the retired `item_tax_master`, 10 rows). `item_master.item_default_tax_id` now
points at `inventory.tax_rate_master`, so no item's tax id is found and the
lookup answers `gst_rate: 0`, `cgst_perc / sgst_perc / igst_perc: 0`. Checked on
A001 (GST 18%) and on the ZT items.

The bill screen fills its tax columns from this lookup, so **every bill posts at 0%
GST**. The test put the real rate on the row by hand to go on checking the GST legs.

**Ask:** read `tax_rate_master` (the id on `item_default_tax_id`) in the lookup.

## 2. The item-price lookup returns no HSN

The lookup's `data` has no HSN field, so the client sends `sbiHsnCode` blank on
every line. `/bills/validate` then says `HSN_DIGITS` (INFO) on every line, and the
GST register gets no HSN.

**Ask:** return `item_hsn_code` in the lookup.

## 3. A bill with any set-off can never be cancelled (or amended)

A bill that set off an advance was cancelled through `/bills/cancel`:

> 409 — A receipt or credit note has been allocated against this bill — cancel that first

The allocation guard (`sales/posting/sales.guards.ts:605-611`) counts the bill's
OWN invoice-side set-off row (`abj_bill_id` = the invoice, `abj_against_bill_id` =
the advance) as somebody else's allocation. Amend runs the same guards, so it is
blocked the same way.

**Ask:** exclude the bill's own `ADVANCE_ADJUST` / `NOTE_ADJUST` rows, the ones
this bill's `/post` wrote, from the "allocated against this bill" count.

## 4. A bill that sets off a credit note cannot be saved (contract decision)

> 400 — Bill cannot be saved • sbAdvanceAmt — adjustments total 618.00 but sbAdvanceAmt says 500.00

- **The client** sends `sbAdvanceAmt` = the ADVANCE-typed set-offs only. Its reason:
  `sbAdvanceAmt` feeds the order's advance ledger, and a credit note relieves a
  different liability (design §5.6).
- **The server** requires `sbAdvanceAmt` = the total of `adjustments[]`
  (`bill.service.ts:1091`). At post it also adds every adjustment into one "advance
  adjusted" figure and posts the whole of it DR `ADVANCE_RECEIVED`, credit notes
  included (`bill-lifecycle.service.ts:1116`).

**Needs one decision, then both sides follow it:**
- (a) `sbAdvanceAmt` is the total of all set-offs. The client changes one line, but
  a credit-note set-off keeps hitting `ADVANCE_RECEIVED`.
- (b) A separate note amount (say `sbNoteAdjAmt`), with the credit-note set-off
  posted to its own ledger role as §5.6 designs.

## 5. `/bills/amend` answers 500

This was predicted from the code, and the test confirms it: every amend of a POSTED
bill returns 500. The re-post reuses `avh_voucher_no = sb_bill_slno`, and the GST
register's source key, while the CANCELLED originals still hold them
(`ux_avh_voucher_no`, `uq_acc_voucher_doc_register_source`). No bill on the box has
ever been amended (there are no AMENDED status rows).

## 6. A second re-tender of the same bill answers 500

The first re-tender of a POSTED bill works: the card is voided, a cash row replaces
it, one balanced type-22 contra voucher is written, and the sale voucher and stock
are untouched. A second re-tender of the same bill returns 500. The new contra
voucher collides with the first one on `ux_avh_src` / `ux_avh_doc_refno`.

## 7. Sale order: `/sale-orders/post` answers 500, and `/create` 500s on an advance tender

- `/sale-orders/create` works without tenders and returns 500 with any tender row,
  even one shaped exactly as the Qt order screen sends it (tender ledger, amounts,
  settle status). No `SALE_ORDER` tender row has ever been stored on the box.
- `/sale-orders/post` of a plain DRAFT order returns 500, so no order can be
  CONFIRMED. Because of this, `/bills/open-sources?kind=ORDER` never lists one.

## 8. A WARN that needs an override can never be overridden on a new bill

A backdated bill gets `SALES_BACKDATE`, and `/bills/validate` answers it twice: once
as a WARN and once as a **REFUSAL**. `warn()` in `sales.guards.ts:46-74` pushes a
refusal unless the code is already in `overrides[]` and the user holds the override
right. The client treats any refusal as final (Back only), and on a NEW bill it has
no override right loaded yet. So the override is impossible from the screen. This
is part client and part server; the client half is written up on the Qt side.

**Ask:** keep refusals for what cannot be overridden, and leave an overridable WARN
as a WARN only. The server re-checks `overrides[]` at `/post` anyway.

---

## Observations (not defects, worth knowing)

- **Cancel uses up a bill number.** The mirror voucher is type 3 and draws on the
  bill series, so every cancel leaves a gap. (bil00153's reversal was bil00154.)
- **A bill can post lines from a DRAFT order** through the whole-order import route.
  `open-sources` only lists CONFIRMED / PARTIAL orders, but nothing on `/post` stops
  a bill against an unconfirmed one.
- **A null bucket gives a 500, not a 400.** `sbiBucket: null` explicitly sent
  overrides the column's NOT NULL DEFAULT 'SALEABLE', and the server reports it as a
  bare 500. The client no longer sends null (fixed), but a 400 naming the field
  would have saved an hour.

## What passed (the contract works)

- **Draft:** numbering; no voucher, stock or receivable on a draft; transport saved
  with the draft; delete removes the draft and its transport row.
- **Cash bill:**
  - stock out equals the quantities and COGS is posted perpetual;
  - SALES = taxable, CGST = SGST, no IGST on an intra-state bill;
  - the voucher balances from its legs;
  - receivable CLOSED; a GST register row with one detail per line; the status trail
    is written.
- **Credit bill:** the typed due days (45) and the due date are saved; the
  receivable is OPEN with those credit days; the party stays debited in full; a bill
  over the 5,000 limit is refused and stays a DRAFT.
- **Tender:**
  - three-way split (cash, card, UPI), change handed back;
  - 269ST cash limit refused and named;
  - a temp credit writes an OPEN `acc_temp_credit` row, stays in the pending amount,
    and a second temp credit on the same mobile is refused.
- **Advance set-off:** the advance's pending falls, the bill's pending nets, and
  `ADVANCE_RECEIVED` DR equals the set-off (once the client sends `adjustments[]`
  on `/post`; see below).
- **Cancel:** stock returned, sale rows plus reversals net to zero, mirror voucher
  balanced, register CANCELED, receivable closed out, status trail written; a return
  against a bill locks it from cancel.
- **Re-tender (first):** only the tender side moves.
- **Transport on a posted bill:** its own `PUT /bills/transport`; the bill does not
  change.
- **Challan import:** the challan's lines do not move stock a second time on the bill.

## Fixed in the Qt client the same day

1. **`sbiBucket: null` on every line.** Since the 23-09 client commit, every bill
   with a line was refused. The client now sends SALEABLE when the bucket is empty,
   as the challan already did.
2. **The operator's chosen set-offs were not sent on `/bills/post`**, so the server
   took the party's oldest credits instead of the ones picked. The client now sends
   `adjustments[]` on `/post` (the DTO already accepted it).
3. **A loaded credit bill forgot the party was credit-allowed**, so the client
   refused every amend of a credit bill. It is now read from party-context's
   `party.creditAllowed`.

---

# Re-run after the fixes (same day, afternoon)

Suite re-run unchanged against the updated server: **45 pass, 8 fail, 8 skip.** Almost
all of the drop is ONE new failure; every later scenario depends on a posted bill.

## NEW — `/bills/post` answers 500 for every bill

Reproduction (no client involved):
1. `/bills/create` any bill with one item line → 200, a DRAFT.
2. `/bills/post` with only the four keys `{sbId, sbCompanyId, sbBranchId, sbAccYear}` → **500 Internal server error**.

It posted fine before the change. Two facts that look related:
- every line is stored with `sbi_tax_id` **NULL** (the server does not fill it at create);
- the client cannot send it — `sbiTaxId` is not on the save DTO (`items.0.property sbiTaxId should not exist`).

Guess: the fix now resolves the output-tax / sales ledgers by the line's tax id, which is
null. Either fill `sbi_tax_id` from the item's `item_default_tax_id` at create (the lookup
already knows it: `tax_id`), or accept `sbiTaxId` on the DTO and the client will send it.

## Confirmed fixed

- **#1 GST**: the lookup now returns `gst_rate 18`, `cgst/sgst 9`, `tax_id`, and the
  output ledgers.
- **#2 HSN**: the lookup returns `hsn_code`. The client read the wrong key
  (`item_com_code`) on all four sales screens; fixed on the client.
- **#7 order post**: a plain order now confirms. (The advance-tender 500 is not re-tested:
  the suite creates orders without a tender.)
- **#8 overridable WARN**: a backdated bill now posts once the override is ticked in the
  popup.

## Not re-tested (blocked by the post 500)

#3 set-off cancel, #5 amend, #6 second re-tender. #4 (credit-note `sbAdvanceAmt`) still
waits on the decision.

---

# Final re-run, after all fixes — 145 pass, 1 fail, 4 skipped

Every item above is confirmed fixed on the live box: GST/HSN from the lookup, the
credit-note figure (`sbNoteAdjAmt`), amend (in place: revision-1 legs soft-deleted,
revision-2 legs live, balanced), re-tender twice (`<bill>/RT1`, `/RT2`), a set-off
bill cancels (advance only), order confirm, the overridable WARN. Set-offs no longer
write an `ADVANCE_RECEIVED` leg — correct, since the advance receipt already credited
the party; the set-off is a bill-wise allocation only.

## NEW — cancelling a bill that set off a CREDIT NOTE answers 500

- **bil00476:** an advance of 500 and a credit note (srt00081) of 118 are set off,
  and the bill is POSTED with `sb_advance_amt` 500 and `sb_note_adj_amt` 118.
  `/bills/cancel` with a reason returns **500 Internal server error**, and the bill
  stays POSTED.
- **A bill with only an advance set off** cancels cleanly in the same run: the
  set-off pair is reversed and the advance's pending comes back.

So the credit-note half of the cancel unwind throws. The adjustment rows look
ordinary: the CN side is `NOTE_ADJUST` 118, row 4, not reversed. Worth stepping
through `reverseAll` → `addToAllocation` on the SALES_RETURN receivable, and anything
the `sb_note_adj_amt` change added to the cancel path.

## Client fix from this round (for reference)

The bill always sent `sbiSrcDocLineNo: null`, and names the ORDER (`so_id`) in
`sbiSrcDocId`. So the fulfilment recompute could not tell which order line was
delivered, and a fully billed order stayed CONFIRMED. The client now sends the order
line number. The server's other accepted form (`soi_id` in `sbiSrcDocId`) was left
alone, because the client groups lines by source document id.

---

# Still open — please pick up (2026-09-24, end of day)

The suite stands at **145 pass / 1 fail / 4 skipped**, and all eight original
items are verified fixed. Two things remain:

1. **Cancelling a bill that set off a CREDIT NOTE answers 500.** See "NEW — cancelling
   a bill that set off a CREDIT NOTE" above; reproduce with bil00476 (advance 500 +
   credit note 118). A bill with only an advance cancels fine.
2. **`/sale-orders/create` still answers 500 with any advance tender row** (item #7).
   Confirming an order now works, but an order still cannot take an advance, so the
   suite's "order advance is set off on the bill" check stays SKIPPED.

**Deploy order, please:** run the migration BEFORE restarting the API. The last deploy
went live about two hours ahead of `20260924100000_sale_bill_note_adj_amt`, and every
bill read and write failed with a bare 500 until the migration ran.

When both are done, tell Viji; the suite (`tests/salebill/run-full.sh`) re-checks both
in one run.

## 3. `sb_modified_by` is stamped with the user's UUID on every server-side verb

The client sends the login NAME as `sbCreatedBy` / `sbModifiedBy` on `/bills/create`,
and `resolveActor(dto, userId)` keeps it. But post, cancel, amend, retender, transport
and update-remarks take no modified-by from the client, and fall back to the request's
user **id**. So a posted bill's audit line reads "modified 019e441b-…" instead of a name
(bil00155 shows it).

**Ask:** in those verbs, resolve the actor from the logged-in user's NAME (the JWT or
`user_master.usr_login_name`), the same text `/create` stores, so created-by and
modified-by read alike.

---

# Re-run after the second round — 147 pass, 1 fail, 3 skipped

Confirmed fixed: cancelling a bill that set off a credit note (bil00476 case), and
`/sale-orders/create` with an advance tender row.

## NEW — an order's advance is booked, but its header does not know

- Order sor00100 was created with a CASH tender of 100. The advance **is** posted:
  voucher `tdVoucherId` 01a0d358-0767…, and an `ADVANCE` receivable of 100,
  pending 100, `abl_src_doc_type = SALES_ORDER`.
- But `/sale-orders/get` still returns `soAdvanceRecdAmt 0`, `soAdvanceBalanceAmt 0`,
  `soAdvanceStatus NONE`, before and after the order is confirmed.
- A bill importing the order carries `soAdvanceBalanceAmt` as its `sbAdvanceAmt`, so it
  asks the server to set off **0**. The server's automatic "order's own advance first"
  applies up to `sbAdvanceAmt`, so nothing is set off and the customer is billed in
  full while the 100 stays open on the order.

**Ask:** when the order's advance tender posts (`order-advance-posting.helper.ts`),
update `so_advance_recd_amt` / `so_advance_balance_amt` / `so_advance_status`, or
derive them in `/sale-orders/get` from the order's live `ADVANCE` receivable. Either way
`soAdvanceBalanceAmt` must equal what is still open.

---

# CLOSED — final run 149 pass / 0 fail / 3 skipped (2026-09-24, night)

Every item in this file is verified fixed on the live box, including the order
advance: a bill importing an order with a 100 advance now sets the 100 off
automatically. The only observation left is by design: a cancel draws one bill number
for its mirror voucher. The 3 skips are loyalty (off for the company), promotions (no
scheme applies to the test items) and IRN / e-way (needs the GST sandbox).
Thank you.

---

# NEW — every bill from the desktop app fails to post: SALES_DEVICE_UNREGISTERED

> Sale bill bil00597 cannot move stock: its device (44c01ccc-a162-8314-34b0-1dbf2f5c7bab)
> is not a registered device and the session has no counter

The chain:
1. The Qt app sends its hardware fingerprint as `device_id` at `/auth/login`
   (`AppConfig::deviceId()`, a SHA-256 of host name and MAC).
2. `auth.service.ts:52-60`: device validation is commented out (`const device = null`),
   so the login answers `device_id: null` and the token carries no device or counter.
3. With no id from login, the client falls back to the fingerprint and sends it as
   `sbDeviceId`.
4. `/bills/post` needs a registered `device_master.dev_id` (or a session counter) to move
   stock, and refuses.

`/bills/validate` does not check the device, so the draft is created first (bil00597 is
a DRAFT now). The client now says so plainly, and a re-save posts that same draft.

This machine's rows in `fixed.device_master` are also inconsistent:

| dev_id | dev_device_uid | name | company / branch |
|---|---|---|---|
| 019e4e4c-… | 44c01ccc-…-1dbf2f5c7bab | — | **019cc885… / 019d9b39…** (another company) |
| 019ea5d1-… | 44c01ccc-…-1dbf2f5c7bab**1** | MY DESKTOP | none |
| 019e72db-… | 44c01ccc-…-1dbf2f5c7bab**m** | MAINS | none |

**Ask (pick one):**
- **(a)** Re-enable the login device match (`findAndUpdateDeviceOnLogin`), so the login
  returns the registered `dev_id` for the fingerprint and the app sends that; or
- **(b)** make the stock post resolve the device by `dev_device_uid` as well as `dev_id`.

With either, the machine needs one clean row for the company and branch it bills in
(Acme Foods 019c8ea6… / Coimbatore 019c8ea7…). And please make `/bills/validate` report
an unregistered device as a refusal, so no draft is created for a bill that cannot post.
