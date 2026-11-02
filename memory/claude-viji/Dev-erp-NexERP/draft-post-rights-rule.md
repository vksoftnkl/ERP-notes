---
name: draft-post-rights-rule
description: "Create = save draft, Post = post incl. new doc; Post without Create = \"no drafts\" (single shop owner) — no new setting"
metadata:
  node_type: memory
  type: project
  originSessionId: 95fb9728-f266-4303-ad2e-7585b7adda08
  modified: 2026-10-07T04:51:46.901Z
---

**User restated 2026-10-12 (final):** CREATE = only a DRAFT; POST = can post, and a FAILED post must never leave a draft behind ("user thought it was saved"), whatever other rights the user has. Guaranteed only by server notes 94 B / 106.

Decided 2026-10-07 (user): no per-form or global "allow draft" setting. The existing user_menus ticks express it:
Create = save a DRAFT, Edit = change a draft, Delete = drop/cancel a draft, Post = post (a NEW doc too, no Create needed),
Cancel = reverse a POSTED doc, Amend = edit posted. New needs Create OR Post. Post-only = single-shop owner, no drafts.
Disable (not hide) a button the user lacks the right for.

The rule lives in ONE place: NexDialog::granted/mayStartNew/maySaveDraft/mayPost/mayCancelDocument (nex_dialog.h).
Wired on opening stock 44, physical 45, stock adjustment 264. TxnMainView::openEntryChild now calls
applyMenuGovernance() — before that, rights were NEVER applied to a screen opened from a txn list.

**Why:** the user's single-shop owner pressed Save draft instead of Save & Post. And a failed post must NOT leave
a draft behind (user, same day) — the owner doesn't know drafts exist.

**How to apply:** sales (bill/order/DC/quotation) and accounts (receipt/payment/register) NOT wired yet — they
wait on notes 94 (share admin/notes-94-draft-post-rights.md): A) new doc saves with Create OR Post (receipt/payment/
register refuse Post-only today), B) save+post atomic in one txn (physical, receipt, payment, bills, orders, DC are
two calls; opening + adjustment already atomic), C) stock routes check no rights. Sale bill already has
sales.auto_post (validate-first, then create+post) — reconcile with this rule when wiring it.

**2026-10-12 — user: "i want to stop to create DRAFT".** Client interim (until notes 94 B; escalated as Shared/Prathap/admin/notes-106-no-drafts-urgent.md):
- Failed Post never leaves a draft: sale bill (check→create→post, createThenPost/saveThenPost, adoptSavedDraft(createdNow)), DC (validate first + discardPostDraft), receipt / payment (no validate route: create→post, discardPostDraft on failure; their drafts hold no number), physical stock (no delete route → the throw-away draft is CANCELLED, number stays as CANCELLED). Sale order: single /create with soStatus — nothing to clean. A REOPENED draft is never deleted.
- Save draft gated on maySaveDraft(saved) (disabled + tooltip; F5/tender paths refuse too) on sale bill, DC, sale order, receipt, payment (physical already).
- Still open: Post-only cannot create on receipt/payment (server notes 94 A); numbers burned on bill/DC failed posts; bill/DC numbers issued at create.
- VIGNESH: 99/100 Post only (no View/Create) → told the user to tick View+Create.

**2026-10-12 later:** /receipts/delete and /payments/delete need the DELETE right, so a failed-post cleanup by a Create+Post user (VIGNESH) gets 403 and the draft stays (receipt 01a1303d left DRAFT). Client message now says so (403 branch). Server ask = notes-106 Part C (creator may delete own draft). Workaround: tick Delete on 99/100. /bills/delete has no rights check.

**2026-10-12 evening: notes 106 LIVE (server).** /bills, /delivery-challans, /receipts, /payments create take post:true (physical: header.status POSTED; sale order soStatus CONFIRMED) = save+post in one transaction, refusal leaves nothing, judged on Post right alone. Client switched on all six; interim delete/cancel-on-failure code removed; receipt/payment m_writeDenied now also needs !canPost (Post-only users can key). Sale bill + DC keep the /validate dry run + popup before the one call. Live-verified on bills only (tests/salebill run-full.sh postfail106 7/7 + postrefused 7/7).
