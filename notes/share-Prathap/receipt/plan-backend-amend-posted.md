# Receipt — `POST /receipts/amend`, editing a POSTED receipt whole (2026-09-17)

**For Prathap.** One route, behind one company setting, default OFF.

**Why it exists.** The current model assumes the person who keys a receipt is
not the person who approves it, so a mistake is corrected by cancelling and
re-entering. That is right for a client with an accounts department and wrong
for a one-person shop: the owner keys the receipt, spots a wrong cheque number
a minute later, and has to unmake and re-key the whole document — party,
tenders, allocations — to change six characters. Asked for by client request
2026-09-17.

**What it is NOT.** Not a second way to post. Not a way around the instrument
rules in §4 below. Not available to a client who leaves the setting off.

---

## 1. Why the full payload, and not field-by-field routes

The first shape proposed was a tier of small routes — one for header text, one
for instrument detail. It was rejected, correctly: the operator does not think
in tiers. They think "this receipt is wrong, here it is again, right this
time." A route per field also multiplies the number of places the posting
rules have to be re-implemented, and each one drifts.

So amend takes **exactly the payload `/create` and `/post` take together** —
the same object the screen already assembles — plus the four keys of the
posted receipt and an `editRemark`.

```jsonc
POST /api/v1/receipts/amend
{ "avhVoucherId": "…", "avhCompanyId": "…",
  "avhBranchId": "…",  "avhAccYear": "2026-2027",

  // …every field of save-receipt.dto  (header + tenders[] + otherLines[])
  // …every field of post-receipt.dto  (allocations[] + creditsApplied[]
  //                                    + otherLineBills[] + onAccount)

  "baseRevision": 1,                                      // mandatory
  "editRemark": "cheque no keyed 55491, actual 55419" }   // mandatory
```

`editRemark` is required for the same reason `/update-header`'s is: the trail
has to say **why**, and nobody writes it afterwards.

`baseRevision` is the revision the client LOADED — the `avh_revision_no` that
came back on `/receipts/get`. It must equal the header's current value or the
amend is refused (§4). It is the optimistic lock, and it is not optional:

> Two people have rct00015 open. A amends the cheque number and saves. B, who
> loaded it before that, amends the party's name and saves. Without the check
> B's payload is the whole receipt, so it silently puts the wrong cheque number
> back — and the audit log faithfully records that B did it on purpose.

A last-writer-wins race is survivable on a master record. On a posted voucher
it rewrites ledger legs, so it is not.

---

## 1a. Why this is its own route, and not a mode on `/create`

Asked and settled 2026-09-17, because it is the obvious first thought:
`/receipts/create` is already an upsert on `avhVoucherId`, so why not let a
POSTED id mean "amend"?

**Because `/create` is the route the screen calls on every draft save** —
every save, every retry after a timeout, every double-submit. Today a POSTED
voucher there is a harmless 409:

> `rct00015 is POSTED. Only a DRAFT may be edited here; use
> /receipts/update-header for narration, or cancel and re-enter to change the
> money.`

Make POSTED mean "amend" and every one of those becomes a silent restatement
of a posted document that nobody asked to change. **Keep that 409 exactly as
it is.**

There is also an asymmetry that does not go away: the normal flow is
`/create` then `/post`, two calls. An amend is inherently ONE call that does
both. Folding it in would make `/create` sometimes write ledger legs and
sometimes not, decided by a status the caller may not know it has.

A middle option was considered — one URL with an explicit `mode: "AMEND"`
rather than an inferred status — and it is sound; it was not taken for one
practical reason. **Amend is the only route in this module gated by a company
setting.** A route that can be switched off, or simply not built yet, is
easier to reason about than a mode flag inside a DTO every client already
sends and that must be checked against a setting on every request. It also
keeps this route independently schedulable from route 8.

---

## 2. The transaction

One transaction. Steps 2–4 are code that already exists — this route
orchestrates §5.3 and §5.2, it does not re-implement them.

1. **Lock** the header `FOR UPDATE`; **refuse unless POSTED**. Lock every bill
   the existing adjustments name, its `acc_pdc_register` rows, and its ADVANCE
   bill.
2. **Refuse** per §4 below, before anything is written — including the
   `baseRevision` check, which is cheap and catches the case where the client
   has been looking at a stale document for the last ten minutes.
3. **Unwind, in place** — §5.3's work on the money, with two differences:
   - one negative row per existing adjustment row (the trigger reopens the
     bills, exactly as cancel does);
   - ADVANCE bills soft-deleted; register rows and their PDC vouchers removed;
     old legs and old tender rows deleted;
   - **NO reversal voucher is written, and the status does NOT become
     CANCELLED.** That is the whole difference from cancel. The document is
     not being unmade, it is being restated.
4. **Re-apply** — §5.2 steps 2 through 12 from the NEW payload, against the
   same header. Re-validate everything 5.1 validates; re-run the identity to
   the paisa; re-run the allocation engine; write new register rows, new legs,
   new adjustments, new ADVANCE, new tender rows.
5. **Keep the identity of the document**: the same `avh_voucher_id`, the same
   `avh_voucher_no`, the same `avh_voucher_refno`, status still POSTED.
   `avh_revision_no = avh_revision_no + 1`.
6. **Trail**: `txn_status_log` POSTED → AMENDED → POSTED carrying
   `editRemark`, and `audit.audit_log` per row touched (§5).

**Numbering is not re-allocated.** A receipt is not a GST document; the
customer is holding a slip with that number on it, and issuing a second number
for the same money is how a shop ends up explaining two receipts for one
payment. The revision counter carries the change instead.

---

## 3. What has to be added

| # | what | why |
|---|---|---|
| 1 | `acc_voucher_header.avh_revision_no int not null default 0` | so the slip can print "rev 2" and a dispute is answerable. One column, no backfill — every existing row is rev 0 |
| 1b | **`/receipts/get` returns `avhRevisionNo`** on the header | without it the client has no `baseRevision` to send back and the lock cannot work. It is the same header object §4.5 already returns — one more field |
| 2 | Register `acc_vouchers`, `acc_bill_adjustment`, `acc_pdc_register` in `audit.audit_log` | §5 — the trail is what makes an in-place edit acceptable, and it is half-built already |
| 3 | `accounts.allow_posted_amend` — BOOL, **COMPANY** scope, default **false** | same shape as `accounts.receipt_salesman_mandatory` in `public.app_setting_def` |

Nothing else. No new table, no new voucher type, no schema change to the money
tables.

---

## 4. What amend REFUSES — the same list as cancel, no exceptions

A setting cannot make these safe, because they are facts about the world
rather than rules about the software.

| refuse when | why | message shape |
|---|---|---|
| any cheque of the receipt is **DEPOSITED or later** | the bank has acted on it | verified live 2026-09-17: *"Cheque 55491 is CLEARED. Once an instrument has left the drawer the receipt behind it cannot be unmade — unwind it on the Received Cheques screen first."* |
| the **on-account ADVANCE has been spent** by a later receipt | the money is already somewhere else | as §5.3 |
| `baseRevision` **≠ `avh_revision_no`** | somebody amended it since this client loaded it | 409, and say what it is now: *"This receipt has been amended since you opened it (now revision 2). Reload it and make the change again."* |
| the receipt is **not POSTED** | a DRAFT is edited by `/create`; a CANCELLED one is history | 409 |
| the **period is locked / the year is closed** | add this one now even though the lock does not exist yet — amend is the route that will need it first | 409 |

The window amend actually serves is the one that matters: a cheque still in
the drawer, minutes after entry. That is where the reported pain is.

---

## 5. The trail, which already exists

`audit.audit_log` is live — 10,777 rows on 2026-09-17 — and already stores
per row touched:

    log_original_record  jsonb    the before
    log_modified_record  jsonb    the after
    log_changed_fields   jsonb    just what moved
    log_user_id · log_branch_id · log_device_name · log_ip · log_screen_id

It already covers `receipt tender` (60 inserts, 1 cancel), `sale_bill`,
`stock_voucher` and the masters. The three receipt money tables in §3 item 2
are what is missing.

**This is the point on which amend was originally argued against and then
accepted.** An in-place edit of posted money is unacceptable when nothing
records what changed; with a before/after snapshot per row, the same edit is
better evidence than cancel-and-re-enter, which leaves two documents and makes
the reader infer the difference.

---

## 6. The client, for reference

`src/modules/accounts/receipt/receipt_entry.cpp`. With the setting on and
`canEdit` on menu 99, a loaded POSTED receipt unlocks its fields and
**Post & Print** becomes **Save changes**, calling `/amend` instead of
`/create` + `/post`. The screen already assembles both payload halves, so
there is no new payload code and no new dialog.

The client holds the `avhRevisionNo` it loaded and sends it straight back as
`baseRevision` — it never computes or increments it. On the 409 from §4 it
reloads the receipt and tells the operator what happened, rather than retrying
with the new number, which would defeat the whole point of the lock.

`btnCancelVoucher` and the register's F3 are unaffected — cancel still means
cancel. Amend does not replace it: a receipt taken from the wrong party is
cancelled, not restated.

---

## 7. Decision to record

**R20 — correcting a posted receipt.** Cancel-and-re-enter for every client
(today), or in-place amend behind a company setting?

**Taken 2026-09-17: both.** `accounts.allow_posted_amend` off is the current
model unchanged, and is the default. On, the owner-operator restates the
document and the audit log carries the before and after. The refusal list in
§4 is identical either way, so no client can amend past an instrument that has
left the drawer.

Two sub-decisions taken the same day, both recorded above so they are not
re-opened:

- **Its own route, not a `mode` on `/create`** (§1a). `/create` keeps
  answering 409 for a POSTED voucher.
- **`baseRevision` is mandatory** (§1, §4). An amend carries the whole
  document, so last-writer-wins would silently undo somebody else's
  correction — on ledger legs, not on a master record.

