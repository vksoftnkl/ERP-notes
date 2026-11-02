# Till — non-cash tender control: card, UPI, wallet (REV 1, 2026-10-07)

> **User asks, 2026-10-07:**
> - *"for cash we can count and differ save in variance but for other payments gate ways if amt
>   missed means how to control / in what way to approve by checking every slip (or) by machine?"*
> - Then: *"write the non-cash tender control plan"*.
>
> **A separate plan, per house rule.** `TILL_DESIGN.md`, `47_till_management.sql`, `48_…sql` and
> `plan-till-receipt-payment-expense.md` are not edited. §0 says which parts this overrides.
> `../49_noncash_tender_control.sql` (run after 48) is the schema side. It was validated on a throwaway
> PG 18 loaded with the live schema; see §11.
>
> Table guide for 47: `47_TABLES_GUIDE.md`. This plan uses its names throughout.

## 0 · What this overrides

| In | Section | Now |
|---|---|---|
| TILL_DESIGN.md | §5.4 last bullets (STATEMENT tenders "reconciled later") | **§4–§6 here**: "later" is the statement matching, with a not-received list and a write-off approval |
| TILL_DESIGN.md | §5.7 "Non-cash at close … the till does not post card settlement" | still true for the **till**; the settlement is posted by **TenderSettlementService** (§5), a TSet voucher per payout |
| TILL_DESIGN.md | §8 S4 step 3 (one card line: slips / amount / batch) | **one line per terminal** (§3.1) |
| TILL_DESIGN.md | §8 S5 cockpit | gains the **slip check** panel (§3.2) |
| 47 / 48 | `ck_trs_category`, `ck_tar_event`, `ck_tev_code` | restated in 49 with NONCASH, NONCASH_WRITE_OFF, SETTLEMENT_RESOLVE, 5 events |
| 47 | `till.variance_default` covers cash **and** non-cash | non-cash has its own `till.noncash_variance_default` = **SUSPENSE** |

## 1 · The decision in one line

**Non-cash money is never in the drawer, so it is proven by machines, not by counting:**
1. A reference is taken at billing.
2. One batch total per terminal is checked at close.
3. Every tender row is matched against the provider's statement the next day.

Slips are checked one by one **only** where a machine total disagrees. Nothing non-cash is written off at
the counter: an unexplained gap waits in SUSPENSE until the statement proves it.

## 2 · What goes wrong, and which layer catches it

| # | What happened | Looks like at close | Real loss? | Caught by | Fixed by |
|---|---|---|---|---|---|
| 1 | Wrong tender keyed (paid cash, keyed card, or the reverse) | cash variance **and** non-cash variance, equal and opposite | no | Layer 2 | **re-tender** the bill: variance `RETENDERED`, no posting |
| 2 | Payment never succeeded (UPI pending / failed, fake screenshot, card declined but billed) | card: the batch total is short. UPI: nothing at close | **yes** | Layer 1 (reference), Layer 3 (no statement line) | `NONCASH_WRITE_OFF`: suspense, recovery or loss |
| 3 | Amount keyed ≠ amount charged on the machine | batch total differs by the gap | sometimes | Layer 2 (card), Layer 3 (UPI) | re-tender the amount, or a PARTIAL settlement |
| 4 | Same UTR / approval code used on two bills | no gap at close (looks paid twice) | **yes** | Layer 1 (refused at save) | refused before it happens |
| 5 | Money arrived with no bill (customer paid, cashier never billed) | nothing | revenue leak | Layer 3 (an unmatched statement line) | `SETTLEMENT_RESOLVE`: link the late bill, refund, or income |
| 6 | Chargeback weeks later | nothing | **yes** | Layer 3 (CHARGEBACK line) | tender row FAILED, recovery from the customer or loss |

## 3 · Layer 1: at billing (prevention)

Changes to the sale bill / return / order-advance / receipt tender save (NestJS + the Qt tender dialog):

1. **A reference is mandatory** on a tender whose `acc_tender_master.tnd_needs_ref` is true. That column
   already exists: CARD and UPI are true on the box today.

   | Tender | Fields | Validation |
   |---|---|---|
   | CARD | `td_auth_code` (approval code on the slip) + `td_card_last4` | auth 6 chars, last4 4 digits |
   | UPI | `td_ref_no` = the 12-digit UTR / RRN (`td_payer_vpa` optional) | 12 digits |
   | WALLET | `td_ref_no` = the wallet transaction id | non-empty |

   Missing → `TENDER_REF_REQUIRED`, on save and on post.
2. **Duplicate reference:** the same `upper(td_ref_no)` (or auth code + last4 + amount) on another live
   row of the same tender is refused, `TENDER_REF_DUPLICATE`, and logs `DUPLICATE_REF_BLOCKED`.
   - The setting `tender.duplicate_ref` = BLOCK (default) or WARN.
   - It is a service check backed by the reconciliation index `ix_td_ref`, **not** a unique index, because rows of two stores meet at the cloud.
3. **Settlement status set at save.** For a tender with a settlement ledger
   (`tnd_settlement_ledger_id IS NOT NULL`), the row is saved with:
   - `td_settle_status = 'PENDING'`;
   - `td_expected_settle_on = bill date + tnd_settlement_days`;
   - `td_settle_ledger_id` = that ledger.

   Today 28 card and 65 UPI rows on the box sit at `NA`, so nothing could ever be reconciled.
4. **Phase 4: integrated payments** (Pine Labs / Paytm / Razorpay EDC or dynamic QR). The bill sends the
   exact amount to the terminal or gateway, and the tender row is written **only** on the gateway's success,
   with the reference filled by the gateway. Cases 2, 3 and 4 then cannot happen. This is a separate plan
   (one integration per provider); nothing here blocks it.

## 4 · Layer 2: at session close (one figure per machine)

### 4.1 One count line per terminal

**Set-up rule:**
- Each card terminal and each UPI QR / VPA is its **own** `acc_tender_master` row (CARD-T1, CARD-T2, UPI-QR1) with `tnd_terminal_id`.
- Today there is one CARD and one UPI tender per company. These are split at go-live.
- The till already counts one line per tender (`till_count_line.tcl_tender_id`), so no schema change.

End Shift (S4) step 3, per terminal (`tender.close_by_terminal` = true):

```
Card  T1  batch no [0031]  slips [14]  batch total [12,340.00]   ← from the terminal's settlement print
Card  T2  batch no [0007]  slips [ 3]  batch total [ 2,150.00]
UPI / Wallet: verified by statement (no input)
```

Stored as SLIPS lines: `tcl_tender_id` = the terminal, `tcl_qty` = slips, `tcl_entered_amount` = the
batch total, `tcl_batch_ref` = the batch no. Expected per terminal = `TillLedgerService.expected(session,
CARD, tenderId)`. That parameter already exists in the design.

- **Match** (difference ≤ `till.noncash_tolerance`, default 0): nothing more to do.
- **Mismatch:** a NONCASH variance row per terminal, and the session goes to PENDING_APPROVAL like a cash one.

### 4.2 The slip check (only on a mismatch): S5 cockpit panel

```
Terminal T1 — expected 12,500.00 · batch 12,340.00 · short 160.00
 time   bill        amount    auth    last4   slip?
 10:12  B-00412    1,250.00  A81K2Z   4432    [✓]
 11:40  B-00431      160.00  —       —        [ ]   ← no slip: payment never taken?
 …
Slips with no bill: [ add slip: amount __ auth __ last4 __ ]   ← paid on the machine, billed as cash?
```

The approver ticks rows against the slips. Each kind of result points at its fix:

| Result | Fix |
|---|---|
| a row with no slip | re-tender that bill (cash / credit) if the customer actually paid otherwise; else it stays a gap |
| a slip with no row | a bill keyed as another tender: re-tender it to card (the cash variance closes too) |
| amount differs | re-tender the amount |

- The ticks are a screen aid and are **not stored**. What is stored is the re-tenders (their own rows) and a `SLIP_CHECK` event with the counts.
- Whatever re-tender could not explain is approved under `NONCASH_VARIANCE` (STORE_MANAGER), with the default treatment **SUSPENSE** (`till.noncash_variance_default`): Dr Tender suspense / Cr the tender's clearing ledger (TVar).
- It is **never EXPENSE at the counter**. The statement decides later (§6).

### 4.3 UPI / wallet at close

- `ttm_close_mode = STATEMENT`: expected is shown, nothing is counted, and no variance is raised at close. This is unchanged from 47.
- The cockpit shows a count of UPI rows **without a reference**. It is 0 once Layer 1 is live, and it is the thing to chase if it isn't.

## 5 · Layer 3: next day, statement matching (the proof)

### 5.1 Where it runs: the store server (one writer per row)

- `acc_tender_detail` rows belong to the store server. If the cloud wrote `td_settle_*`, the store's next push would overwrite them. So **import, match and post go through the store server's API**, as a remote till approval does.
- **HO with a chain-wide file:** HO uploads it at the cloud, the cloud splits it by terminal id / VPA, and forwards each part to its store's API. A store that is offline gets its part when it reconnects.
- **Single shop:** the owner uploads at the store.

### 5.2 The two new tables (49), and why

| Table | Holds | Why no existing row can |
|---|---|---|
| `acc_settlement_import` (asi_) | one **payout** = one bank credit: provider, file + hash, payout UTR and date, gross / fee / tax / net, status, the TSet voucher | a payout spans many bills; nothing of ours represents it |
| `acc_settlement_line` (asl_) | one provider line: kind, time, terminal / VPA, RRN / UTR, auth, last4, gross / fee / tax / net, **what it matched** and how | a line can be money with **no bill** (case 5); it has no tender row to sit on |

Everything else rides on existing columns: `td_settle_status / _settled_on / _settle_amount /
td_mdr_amt / td_settle_ref_no / td_settle_voucher_id`.

**Rules the database enforces:**
- `ck_asi_net`, `ck_asl_net`: net = gross − fee − tax (one row's own arithmetic).
- `ck_asl_linked` / `_matched` / `_resolved`: a line's status agrees with its link.
- `ck_asi_posted` / `_void`.
- `ux_asi_file`: the same file is imported once.
- `ux_asl_td_matched`: one sale line settles a tender row, within a statement year; the service covers a payout crossing 31 March.

### 5.3 Import

`POST /api/v1/tender-settlement/import` takes a multipart file plus the tender.

- The file is read with the tender's `tnd_statement_format`, a new jsonb column on `acc_tender_master`. It is the column map of that provider's CSV / XLSX: which column is the RRN, the amount, the fee and so on.
- **One import = one payout.** A file carrying several payout dates is split into one import per payout ref.
- Each line resolves `asl_tender_id` from the terminal id / VPA. Lines of another store are refused when the file is a single store's.

### 5.4 Matching (TenderSettlementService.match, idempotent, re-runnable)

Candidates are tender rows of the line's tender with `td_settle_status IN ('PENDING','PARTIAL')`, not
voided and not deleted. The first rule that hits wins:

| Rule | Key | Result |
|---|---|---|
| **REF** | `asl_ref_no = td_ref_no` (UTR / RRN) and amount within `tender.match_amount_tolerance` | MATCHED automatically |
| **AUTH** | card: `asl_auth_code = td_auth_code` and last4 equal and amount equal, txn date ±1 day | MATCHED automatically |
| **AMOUNT_TIME** | same tender, same amount, within `tender.match_window_minutes`, **exactly one** candidate on each side | **SUGGESTED**: a person confirms |
| none | | UNMATCHED |

Line kinds:
- **REFUND:** matched to the SALE_RETURN / refund tender row (Cr side).
- **CHARGEBACK:** matched to the original sale row; that row becomes `FAILED` and goes to §6.
- **FEE / ADJUSTMENT:** rows with no customer behind them; they go straight to the voucher.

### 5.5 Post: one TSet voucher per payout

```
Dr  Bank (tnd_settlement_ledger_id)                    net
Dr  Bank charges (role BANK_CHARGES)                   fee
Dr  Input CGST / SGST or IGST (INPUT_* roles)          tax on the fee
    Cr  the tender's clearing ledger (tnd_ledger_id)   gross of the MATCHED sale lines − refunds
    Cr  Tender suspense (TENDER_SUSPENSE)              gross of UNMATCHED sale lines (money with no bill)
```

On post, each matched tender row gets:
- `td_settle_status = SETTLED`, or `PARTIAL` when `asl_amount_diff ≠ 0`;
- `td_settled_on` = the payout date;
- `td_settle_amount` = the line's gross;
- `td_mdr_amt` = the line's fee + tax;
- `td_settle_ref_no` = the payout UTR;
- `td_settle_voucher_id` = the TSet voucher.

Posting needs **no** approval. It records what the bank already did. **Void** = a reversal voucher; the
rows go back to PENDING.

## 6 · The two exception lists, and their approvals

### 6.1 Not received: our row, no money

The grid is over `ix_td_settle_pending`: PENDING / PARTIAL rows past `td_expected_settle_on +
tender.settle_grace_days` (default 2). It shows the bill, session, cashier, amount, reference and age.

| Outcome | How |
|---|---|
| it turns up in a later statement | matched normally; it leaves the list |
| it was a wrong tender | re-tender the bill (works on a closed session; the money moves in the current session, D7) |
| it never came | **`NONCASH_WRITE_OFF`** approval (STORE_MANAGER, shipped ALWAYS) with a NONCASH reason. Treatment: **RECOVER** (cashier's recovery ledger), **SUSPENSE** (still chasing), or **LOSS** (WRITE_OFF role). Posts a TVar: Dr that ledger / Cr the tender clearing ledger. `td_settle_status = FAILED`, `td_settle_voucher_id` = the TVar. A `NONCASH_WRITTEN_OFF` event carries the row's **original session**, so it shows on that session's History |

For a chain, HO sees the same list across stores and decides **through the store's API**. HO's comment
goes to `till_session_review` (outcome RECOVERY_ORDERED / WRITTEN_OFF).

### 6.2 Unexplained: money, no row

The grid is over `ix_asl_unexplained`: UNMATCHED SALE lines.

| Outcome | `asl_resolution` | Posting |
|---|---|---|
| the bill is found, its session still **OPEN** | **LINKED**: re-tender that bill to this tender (the drawer is live), then match MANUAL | journal: Dr Tender suspense / Cr clearing |
| the bill is found, its session **CLOSED** (the drawer came up short at close and that variance was decided) | **LINKED**: a **pairing**, not a re-tender. No money moves today, so nothing may touch today's drawer (D7). The line is paired with that session's variance; the variance is re-decided the 47 way (REVERSED row + new row RETENDERED) | none when the close variance went to SUSPENSE (both halves already sit in Tender suspense); else journal Dr Tender suspense / Cr where the variance went (Cash short & excess, or the cashier's recovery ledger, which pays the cashier back) |
| the customer paid twice / by mistake | **REFUNDED** | the refund's own payment |
| genuinely unbilled sale / unknown | **INCOME** (or a late bill) | journal: Dr Tender suspense / Cr the income ledger |
| still unknown | **SUSPENSE** | stays in suspense |

Each one goes through **`SETTLEMENT_RESOLVE`** (STORE_MANAGER) with a NONCASH reason (UNBILLED_PAYMENT …).

### 6.3 Tender suspense must reach zero

The `TENDER_SUSPENSE` ledger (role from 47) collects:
- close-time non-cash gaps (§4.2);
- unexplained money (§5.5);
- written-off amounts marked SUSPENSE (§6.1).

Its balance, broken down by tender row / statement line, is a report. A non-zero balance older than N days is
the one number the owner or HO watches.

## 7 · Server (NestJS)

| Service | Owns |
|---|---|
| `TenderRefGuard` (sales / receipts tender save) | §3: required ref, duplicate ref, PENDING + expected date |
| `TillLedgerService` (47) | expected per terminal (`tenderId`), already in the design |
| `TenderSettlementService` | import, match, confirm, unlink, ignore, resolve, post, void, write-off |
| `TillApprovalService` (47) | the two new events, same gate and authority grid |

Routes (`/api/v1/tender-settlement/*`, a new family; lists are grids, not routes):
- **Import and read:** `POST import`, `GET get?asiId`.
- **Matching:** `POST match`, `POST confirm` (line + td), `POST unlink`, `POST ignore`.
- **Decisions and posting:** `POST resolve` (line), `POST post`, `POST void`, `POST write-off` (td).

Grids: imports, lines of an import, not-received, unexplained, duplicate references, tender-suspense breakdown.

Error codes:
- **Billing:** `TENDER_REF_REQUIRED`, `TENDER_REF_DUPLICATE`.
- **Import:** `SETTLEMENT_FILE_DUPLICATE`, `SETTLEMENT_FORMAT_MISSING`, `SETTLEMENT_OTHER_STORE`.
- **Matching:** `SETTLEMENT_TD_ALREADY_MATCHED`, `SETTLEMENT_NOT_BALANCED`.
- **Write-off:** `SETTLEMENT_POSTED_LOCKED`, `NONCASH_WRITE_OFF_NEEDS_APPROVAL`.

## 8 · Screens (Qt), each its own `.ui` on NexDialog

| Screen | What |
|---|---|
| Tender master (existing) | per-terminal rows, `tnd_needs_ref`, settlement ledger + days, **statement format** (column map editor with a "test on a sample file" button) |
| Sale bill tender popup (existing) | the reference fields, mandatory per tender; duplicate refused inline |
| End Shift S4 (till) | one line per card terminal (§4.1) |
| Cockpit S5 (till) | **slip check** panel (§4.2) on a non-cash variance |
| **Settlement Reconciliation** (new menu) | import → match grid (statement lines ↔ tender rows, status chips) → confirm suggestions → Post (TSet). Tabs: Not received · Unexplained · Duplicate refs · Suspense |

Mockups (drawn 2026-10-07 by `till_settlement_ui_mockups.py`, not yet reviewed; once approved they are
the visual spec):
- `till_settlement_match_ui_mockup.png`: match grid, suggestion confirm, TSet preview.
- `till_settlement_exceptions_ui_mockup.png`: Not received + write-off, Unexplained + resolve.
- `till_settlement_import_ui_mockup.png`: import popup + the tender's statement column map.

They tell one story: an HDFC payout of 08-10 for T1 (C01) and T2 (C02), with gross 18,944.00, fee 474.56, GST 85.43 and net 18,384.01.
The 500.00 with no bill is SURESH B's −500 C02 cash shortage from the S8 / S11 mockups.

## 9 · Settings (49)

| Key | Default | Scope | Meaning |
|---|---|---|---|
| `tender.duplicate_ref` | BLOCK | company | same reference on two bills: BLOCK or WARN |
| `tender.close_by_terminal` | true | branch | card close = one batch total per terminal |
| `tender.match_window_minutes` | 30 | company | AMOUNT_TIME suggestion window |
| `tender.match_amount_tolerance` | 0 | company | accepted difference on a REF / AUTH match; over it, PARTIAL |
| `tender.settle_grace_days` | 2 | company | days past expected settlement before "not received" |
| `till.noncash_variance_default` | SUSPENSE | company | treatment of an approved non-cash close variance |

## 10 · Build order

1. **Billing (§3).** The sale bill / receipt tender refs and duplicate check, plus PENDING settlement status. This is the biggest win and needs no new screen.
2. **Close (§4).** Per-terminal tender masters, the S4 lines and the S5 slip check. This needs till phase 1 to be live.
3. **Statement (§5).** 49 applied, the import + format map for **one** provider first (the user's actual acquirer), match, and post.
4. **Exceptions (§6).** The two lists, two approvals and the suspense report.
5. **Integration (§3.4).** A separate plan per provider.

## 11 · Validation done (2026-10-07)

A throwaway PG 18 cluster was loaded with:
- `pg_dump --schema-only` of the live DB;
- the live rows of tender types / masters, voucher types / series, ledger roles, setting defs, companies, branches and devices.

Then 47 → 48 → 49 were applied. **49 applies, re-runs cleanly (idempotent), and 16 probes pass.** Each
probe that should fail is rejected by the intended constraint:

| Probe | Result |
|---|---|
| valid payout + matched line | accepted |
| payout net ≠ gross − fee − tax | ck_asi_net |
| line net ≠ gross − fee − tax | ck_asl_net |
| MATCHED with no tender row | ck_asl_linked |
| MATCHED with no who / when | ck_asl_matched |
| two sale lines on one tender row | ux_asl_td_matched |
| a refund line on the same row | accepted |
| same file twice | ux_asi_file |
| RESOLVED with no resolution | ck_asl_resolved |
| POSTED with no voucher | ck_asi_posted |
| VOIDED with no reason | ck_asi_void |
| NONCASH_WRITE_OFF rule, 48's CASH_PAYMENT rule | accepted |
| PAID_OUT rule | refused (stays retired by 48) |
| SLIP_CHECK + 48's EXPENSE_POSTED events, NONCASH reason | accepted |

**One fix found by the run:** a unique index on a partitioned table must include the partition key, so
`ux_asl_td_matched` carries `asl_acc_year`. The service covers a payout that crosses 31 March.

**Nothing was applied to 192.168.0.106.**

## 12 · Business choices still open (only the user can answer)

1. **Which acquirer / PSP does the shop use?** The import is built for one real file format first. A sample settlement file (card) and a UPI settlement report are needed.
2. **Static QR or dynamic QR for UPI today?** With a static QR, Layer 3 is the only proof for UPI, so the reference at billing (§3.1) matters most.
3. **Default treatment of a non-cash amount that never arrived:** RECOVER from the cashier, or LOSS? The plan ships SUSPENSE at close and decides at write-off.
