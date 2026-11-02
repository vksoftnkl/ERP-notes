# Notes 124: what is still open on the till server (2026-10-16)

**STATUS:** partly DONE: migration `20261016100000_till_approval_gate_notes_124` checked live 2026-10-16 11:00 (table at the end). A checklist, not a new design. Every item points to the notes file that has the detail. Items 1–4 come from data or from questions. Items 5–8 are server work that has not been built yet. Items 9–10 are outside the till but still open.

**Checked live 2026-10-16:** notes 102 is DONE. Group 283 "Till Masters" sits under 271, and 275 / 280 / 281 / 282 are in it. Please set its STATUS line to done. Nothing else here was rechecked today.

## A · Repair live data (please do these first)

### 1 · Two sessions closed with an undecided cash gap (notes 112 §2)

Both were closed before your notes-112 fix. Both still show `tss_status = CLOSED`, `tss_variance_status = PENDING` (checked 2026-10-16). The gap is in no voucher:

| Session | Expected → counted | Gap |
|---|---|---|
| `VIJI PC-261012-02` | cash 5,600 → 7,500, UPI 700 → 750 | cash **+1,900**, UPI **+50** |
| `ZT1-261012-01` | cash 2,177 → 2,000 | cash **−177** (test session) |

**Ask:** fix them by hand, or let an approver decide a PENDING gap on a CLOSED session (notes 112 §3). Please tell us which treatment you used, so Business Day and the day Z add up.

### 2 · Till events deleted by the e2e cleanup (notes 115 §3)

The timelines are still empty for ZT1-261012-01, VIJI PC-261012-01, the 12-10 part of VIJI PC-261012-02, and ZT1 / VIJI PC-261014-01 … -03.

**Ask:** can they be restored from a backup? If not, say so. We will then mark them "events lost (test cleanup, 14-10)" and not look for a bug.

## B · Questions left from the retest (notes 122)

### 3 · Negative safe opening on the 13-10 store Z (notes 122 §6)

The re-close froze the opening at **−1,426.00** "as of" the day. That is better than −8,326, but it is still negative. On 13-10 the test safe gave out floats before those sessions' hand-overs were approved, and they were approved on a later day.

**Question:** should the day Z print a negative safe as it is, or flag it, for example "Safe below zero: hand-overs not yet approved"? Or should approved hand-overs count back to the day they were handed over?

### 4 · The earlier 500 on "Plain stock" bills (notes 122 §2)

We can't make it happen any more: the saved repro now posts (201). It happened 4 times on 10-15 between **12:30 and 13:10 IST**.

**Ask:** look at the server log for that window and tell us the cause, so we know it can't come back.

## C · Server work not built yet

### 5 · The approval gate (notes 105 Part B)

Part A (decide-variance) is live. Part B is not built: `POST /till/approvals/request · decide · get · cancel` and a pending list for an inbox (TILL_DESIGN §S7). These events are still only reported or left disabled:
- FORCE_CLOSE
- FLOAT_MISMATCH at open
- NONCASH_VARIANCE after the slip check
- paid-out and expense over the limit (EXPENSE 500 is reported, not enforced)
- SAFE_VARIANCE (two-person)
- void bill / void line

**Remember the user's rule (2026-10-15):** the counter is never blocked while a customer waits. A rule that would stop billing warns first. It refuses only where a company turns that on.

### 6 · Safe figures on the day Z print data (notes 118, the open part)

`days/get` carries `safe.opening / expected / counted / remitted`. `till.day.header` (provider `till.day.*`) does not, so the SAFE block on `TillDayZ80` has nothing to print yet.

**Ask:** add the four `safe_*` fields to `till.day.header`, worked out the same way as `days/get`.

### 7 · Repeated supplier invoice on an expense → bare 500 (notes 103)

`/expenses/create` with the same supplier, invoice no and FY as a live expense hits `ux_avh_doc_refno` and answers `500 Internal server error`.

**Ask:** check for the clash before writing, as `voucher-register.service.ts` does (`docRefnoClash`). Refuse with a code and `field: 'gstBill.invoiceNo'`. Please also send us the code, so we can write its plain wording.

### 8 · Wording on force-close by the cashier (notes 117)

When the session's own cashier tries a force close, the refusal says it "does not decide its own variance". It should say that a cashier cannot force-close their own session. If it has its own `code`, we will add plain wording for it to the catalogue.

## D · Outside the till, still open

### 9 · Error codes (notes 119)

Still missing: the voucher header refusals and 2 voucher throws (§1), `code` required in the shared helpers (§3), and the office masters (§2).

### 10 · Receipt surcharge rounding (notes 107)

Work out the surcharge inside the tender: `round2(tdAmount × perc / (100 + perc))`. Then 200 + 1% UPI is 202.00, not 202.02. The client works with either formula.

### 11 · A supervisor who also bills approves their own session (user, 2026-10-16)

The user's words: *"allow to approve self because sometimes supervisor can bill"*. In a small shop the supervisor takes the till at times. Today nobody is left to approve their session.

- **decide-variance:** nothing to do. `assertApprover` already follows `tar_allow_self`. A company turns on **Allow self** on its CASH_VARIANCE rule (and NONCASH_VARIANCE if wanted) in Approval Rules (276), and gets its own copy of the shared row. The grant, the limit and the PIN still apply, so self-approval is still signed. **Qt:** the Cockpit no longer greys Approve on your own session (built 10-16). The server decides, and TILL_APPROVER_SELF shows when the rule says no.
- **approve-cash (`till-movement.service.ts` ~1301):** this still allows self only under `till.single_operator`. That setting means a one-person shop, and it also lets the owner skip the PIN. **Ask:** also allow the session's cashier when the company's CASH_VARIANCE rule has `tar_allow_self` (or add an APPROVE_CASH rule, whichever you prefer). Always with their PIN, never PIN-free. `tap_self_allowed` on `till_approval` already records it.
- Please keep the event journalled, so the Cockpit and the day Z can show "approved by own cashier".

### 12 · Security, do before the rest: SUPER ADMIN on create (admin/notes-125 §3)

A client ADMIN can create a SUPER ADMIN user through the API, and that user holds the implicit HO_FINANCE grant on every till. The full ask is in `Shared/Prathap/admin/notes-125-user-role-matches-till-levels.md` §3.

## On hold, nothing to do now

- **Notes 123, 269ST / PAN / HSN in bill posting:** on hold by the user's decision (never block the counter). It will come back later as warn-only.

## Qt side, for your information

Nothing in this list needs a Qt change before you start. We will retest each item live once you say it is deployed:
- 1: the Cockpit and Business Day figures.
- 6: the day Z preview.
- 7: the expense refusal box.
- 5: the screens get built once the routes exist.

## Recheck after your migration (2026-10-16, ~11:00): verified live

| § | Result |
|---|---|
| 1 | **Server done.** `approvals/pending` lists both as SESSION_VARIANCE `afterClose: true`. `decide-variance` on a CLOSED session: RECOUNT → 400 "nothing to recount", APPROVE goes on to the reason check (probe with a bogus reason, nothing written). **Qt built:** the Cockpit's Approve works on a closed session with an undecided cash gap (approve-only popup); checked on VIJI PC-261012-02 (band +1,900). **Not decided yet:** the user picks the treatment. Correction: the second gap there is **CHEQUE +50**, not UPI. ZT1-261012-01's cashier is VKPOS, so another approver decides it. |
| 2 | No answer yet: restored, or lost for good? |
| 3 | No answer yet. 15-10 also froze a negative opening now (**−1,346.00**). |
| 4 | No answer yet (server log 12:30–13:10 IST, 10-15). |
| 5 | **Done (routes).** `approvals/request · decide · get · pending · cancel`, `till.approval_gate` WARN as shipped / REFUSE per company; expense posts under WARN with a PENDING request. Qt screens (inbox, request) are ours to build next. |
| 6 | **Fixed.** The 15-10 store Z prints SAFE: expected 9,042.00 · counted 7,347.00 · difference −1,695.00 · sent to bank 100.00. |
| 7 | **Fixed.** A repeated supplier invoice → 409 `EXPENSE_GST_BILL_DUPLICATE`, field `gstBill.invoiceNo`, names exp00073; nothing saved (Qt harness `SNAP_INVNO`). |
| 8 | Not checked (needs a live session of the caller). |
| 9 | Not seen yet. |
| 10 | Not seen yet (API docs unchanged). |
| 11 | Not done: `approve-cash` still allows self only under `till.single_operator`. |
| 12 | Not done: `UserType` unchanged (MANAGER still there, no new names). The SUPER ADMIN guard can't be checked without a client ADMIN login, so please confirm it. |

**New codes without plain wording yet (we'll draft them):** TILL_APPROVAL_NOT_PENDING, TILL_APPROVAL_EXPIRED, TILL_APPROVAL_NOT_YOURS, TILL_APPROVAL_INVALID, EXPENSE_GST_BILL_DUPLICATE.
