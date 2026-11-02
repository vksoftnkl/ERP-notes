---
name: simple-error-messages
description: User rule — every error box shows simple English (what happened + what to do) plus a small grey server-detail line; wording reviewed in share before coding
metadata:
  node_type: memory
  type: feedback
  originSessionId: 3ef83d0b-5600-4825-9b13-0befb9e93814
  modified: 2026-10-15T04:17:01.842Z
---

Every error message a user sees must be SHORT and simple (many users do not know formal English): title 2-4 words, one short "what happened" line, one "Do:" line, everyday words (till not session, "cash short or extra" not variance, "supervisor OK" not approval). A plain title (the action, never "409"/"Bad Request"), what happened, what to do. Below it a small grey line keeps the server detail (status · code · field + the server's own text) so a user's screenshot is still readable by us/Prathap.

**Why:** 2026-10-15 a cashier hit "409 · Session … is PENDING_APPROVAL: it takes no money" on tender and couldn't know what to do; the user wants this for ALL errors, not just till. They also said: confirm before doing anything on this.

**How to apply:** wording goes in a share file FIRST for the user to correct (till: Shared/Prathap/messages/till-error-wording.md, REV 2 short version 2026-10-15). Till wording CODED 10-15: src/core/api/api_error_text.cpp (table) + ApiClient::handleError + AppMessage::showWithDetail (QLabel#msgDetail, template v27 unpublished) + till popups use ApiErrorText::inlineHtml; screens set property "nexKeptLine" ("Bill not lost."). Test SNAP_SRC=err_text tests/till/snap.sh 11/11. CORRECTION 10-15: sales posting (bill/return/quotation/order/DC) and vouchers (receipt/payment) ALREADY send `code` (throwSales*, throwState etc.); uncoded = txn-hold, tenderDetail, voucher header, settings/Inventory/master masters, generic module-shared helpers, ValidationPipe 400s → asked in Shared/Prathap/messages/notes-119-error-codes-everywhere.md. Next on our side: wording files for sales (~101 codes) + vouchers. Related: [[till-management-design]].

**10-15 later:** counter-error-wording.md (HOLD_/TENDER_LINE_/VALIDATION_/INTERNAL_ERROR, notes 119) + till §8 safe codes approved and coded; [APP] row = any HOLD_/TENDER_LINE_/TENDER_REF_/VALIDATION_ code without its own row. A code with NO wording (SALES_*, voucher codes) shows the server sentence under a plain title + code in grey — the "Something went wrong" fallback hid real refusals, removed. Next: wording for SALES_ (~101) + voucher codes. Test SNAP_SRC=err_text 19/19.

**10-15 decision (user):** wording belongs on the SERVER (hybrid: Qt + React + mobile). Asked in Shared/Prathap/messages/notes-120-error-wording-on-server.md: `user {title,line,todo}` on every error + `GET /error-messages` catalogue (cached for offline), seeded from error-messages-seed.json (109 rows from api_error_text.cpp). Qt table stays only as fallback until notes-120 is live; then Qt shows the server's `user` block. Approval flow unchanged: user approves share-file wording → Prathap loads rows.
**10-15 notes-120 built by Prathap** (migration 20261015130000; fixed.error_message erm_* columns, 110 rows / 104 codes; GET /error-messages; ErrorWordingInterceptor words /validate notes) — NOT deployed at 10-15 pm (404). Qt: ApiErrorText::plain() now prefers errors[0].user; Qt table = fallback. err_text 20/20.
**10-15 late:** sales wording drafted — Shared/Prathap/messages/sales-error-wording.md REV 1 (all 90 SALES_/sales GST_ codes; only SALES_DAY_CLOSED was in the catalogue) + sales-error-messages-seed.json (90 rows / 89 codes). notes-123 §5 LOADED 10-15 7:30 PM (catalogue 2026-10-15.3, 202 rows), verified 90/90; 269ST still commented out (on hold).
