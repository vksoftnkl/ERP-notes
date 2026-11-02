---
name: sale-bill-live-test
description: "tests/salebill/run-full.sh — the live end-to-end sale bill suite (123/8/5 on 2026-09-24), its ZT- data, the VKPOS rights grant, the 83 burned bill numbers, and the 8 open defects"
metadata:
  node_type: memory
  type: project
  originSessionId: 2a57b2fc-d57a-4c6a-8daf-436af25c2f7f
  modified: 2026-09-24T07:53:57.345Z
---

Built 2026-09-24. `tests/salebill/run-full.sh` + `test_full.cpp` drive the bill
screen through its own slots (`#define private public`) and check every effect
in the DB with read-only psql. **It POSTS FOR REAL on .106** (user's decision).
It uses its own data, find-or-create and never deleted (master names stay
unique after a delete): ZT-CUST-CREDIT, ZT-CUST-CASH, ZT-ITEM-A (18%),
ZT-ITEM-B (5%). A start-of-run cleanup cancels or deletes earlier runs' ZT bills
and draft orders, and resets the credit customer's terms (5,000 limit, NO
bill-count limit). Each run uses a fresh temp-credit mobile. `run-full.sh probe
<file>` dumps one bill's payload instead.

Final 2026-09-24: **123 pass / 8 fail / 5 skip.** Every fail is a real defect,
written for Prathap in `share/sales/NOTES_bill_live_test_2026-09-24.md`:
GST 0% from the item-price lookup (it reads item_tax_master), no HSN in the
lookup, a set-off bill can't cancel (allocation guard), the credit-note
sbAdvanceAmt contract (NEEDS USER DECISION), amend 500, second re-tender 500,
sale-order post 500 / advance-tender 500, overridable WARN also sent as a
refusal. Client fixes made the same day: sbiBucket null (every bill had been
refused since 0137e5e), adjustments[] sent on /post, allowCredit on loaded bills.

**Side effects on .106:**
- VKPOS was granted post/cancel/amend/override on menus 11, 13, 182 via
  /user-administration/create. It is a full-replace save, so the whole menu set
  was re-sent; the before/after diff showed only those 3 rows changed.
- My payload bisect created 83 empty drafts. They were deleted, but **bill
  numbers ~bil00157–bil00239 are burned**. Next time bisect against
  /bills/validate, or with a key set that fails, never with /create.

**After Prathap's fixes (same day, evening): 145 pass / 1 fail / 4 skip.** The
server now amends IN PLACE (revision-1 legs soft-deleted), links re-tender contras
by avh_doc_refno '<bill>/RT<n>', and writes NO set-off leg (the advance receipt
already credits the party). sbNoteAdjAmt carries the credit-note set-offs. The one
open defect: cancelling a bill that set off a CREDIT NOTE → 500 (advance-only
cancels fine). The client now sends sbiSrcDocLineNo (it was always null, so billed
orders stayed CONFIRMED). His deploy briefly preceded his migration — a blanket
500 across every post means check _prisma_migrations before debugging.

**CLOSED, 2026-09-24 night: 149 pass / 0 fail / 3 skip.** Every backend item is
fixed, including order advance tenders and the order header's advance figures.
Save-and-post (sales.auto_post) now VALIDATES BEFORE /create (postNewBill →
createThenPost), so a refused post leaves no draft and burns no number. A post
that fails after create adopts the draft (adoptSavedDraft sets isNewEntry=false,
so the retry sends sbId); the duplicate bills came from the missing sbId.

Related: [[sale-bill-party-rules]], [[test-box-data-not-a-bug-list]], [[user-menu-full-replace]]

**Device trap (2026-09-24):** the suite hard-codes a REGISTERED device id ("raman",
019e7286-…), so it passes while the real app fails. The app sends its hardware
fingerprint (AppConfig::deviceId) at login, the server's login device match is
commented out (auth.service.ts:52) and returns device_id null, and
AppSession::deviceId() then falls back to the fingerprint. /bills/post refuses it
(SALES_DEVICE_UNREGISTERED). The fix is server-side (share/sales NOTES, last
section). A "works in the suite, fails in the app" report means: check the device first.

**Re-run 2026-09-25 evening: 156 pass / 0 fail / 3 skip** (loyalty off, no promo
for ZT items, no GSP sandbox). Extra modes all green: chequecheck,
chequebounce, advamend (4 cases), partial, holdcheck, tendershot, amendmatrix,
itemclear, custlimit, chargeform.
**Harness traps fixed that day:** (1) the askText remark prompt logs TWO
messages, so waits written as `g_msgs.size() > 1` ended early and the next
freshScreen() destroyed the screen with its prompt still open (amend never
sent). Waits now look for "updated" / "cancelled" / an error. (2) Extra argv
to modes that take a filter (advamend, amendmatrix) silently narrows them. (3)
An edit that replaces a block up to "the next `if (argc > 1`" swallowed the
`cleanupEarlierRuns(); ensureStock();` call; check for it after every harness
edit.
Hold/autosave snapshots now also carry brand / section / category / loyalty PV
(the promotion binder keys on them).

- 2026-10-12: **Post now checks BEFORE it writes** (user: "refused to post but saved as DRAFT"). postBill → /bills/validate on the payload → popup → new bill: createThenPost (draft deleted if /post fails — verified bil01139 sb_is_deleted) / saved draft: saveThenPost. adoptSavedDraft(id, refno, createdNow): only a draft created by this post is deleted on failure — the auto-post Save of a REOPENED draft used to delete the operator's draft. `run-full.sh postrefused` 7/7 (both auto_post modes: refused Post → 0 new bills). Suite device kDevice 019e7286 ("raman") was DEACTIVATED 10-12 12:24 → every real post refuses "not a registered device and the session has no counter" — the full suite will fail until kDevice changes.
