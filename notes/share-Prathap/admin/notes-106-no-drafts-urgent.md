# Notes 106: please build notes 94 Part B first, save and post in ONE call (2026-10-12)

**STATUS:** DONE 2026-10-12 (server). Client switched the same evening: sale bill, DC, receipt, payment, physical stock use the one call (`post: true` / `header.status POSTED`); sale order already sent CONFIRMED. **Verified live on bills:** a post refused after a clean dry run (device not registered) left NO sale_bill row, not even a deleted one (945 → 945, with auto_post off and on), and both refusals named bil01170, so the number was not consumed. Receipt / payment / DC / physical not yet run live.

**Small follow-up:** the refusal text still names a number ("Sale bill bil01170 cannot move stock…") although nothing was saved. On the one-call path please leave the number out (or say "this bill"), so the user does not read it as saved.

**Order:** before anything else in the queue. It is notes 94 Part B (written 2026-10-07), moved up.

## The rule (user, 2026-10-12, final)

> **CREATE** = create only a DRAFT.
> **POST** = can post. **When a post fails, do NOT save it as a DRAFT automatically**, because the user thinks it was saved.

So a Post that fails must leave **nothing**: no draft, no number, whatever rights the user has (Delete or not). Only the server can guarantee that (Part B). The client already deletes what it can, but on receipts and payments that delete needs the Delete right (Part C), and the number is already issued on bills and challans.

## What the user said

*"when i have refused to post means in sales bill it saved as DRAFT automatically its not fair im trying to post but saved as draft"*, and then: *"but i want to stop to create DRAFT"*.

## Why the client cannot do it alone

Every Save & Post is two calls today: `/create` (always a DRAFT, and **the bill number is issued here**), then `/post`. When `/post` refuses something that `/validate` did not catch, the draft already exists and its number is gone. The client now deletes that draft at once, but:

- the number is burned;
- for a moment there is a draft the user never asked for;
- the refusal text names the number ("Sale bill bil01141 cannot …"), so the user reads it as "saved".

**Today on .106**, sale bills that were created as DRAFT and deleted again after a failed post: bil01125 10:41, bil01126 10:42, bil01127 10:42, bil01132 11:53, bil01134 11:53, bil01135 12:35, bil01136 12:35, bil01137 12:52, bil01138 12:54, bil01141 12:58, bil01142 12:59.

`/bills/create` says so itself: *"sbStatus in the body is ignored — a saved bill is a DRAFT until /bills/post"*.

## Asked (as in notes 94 B)

For `/bills`, `/receipts`, `/payments`, `/stock/physical`, `/sale-orders`, `/delivery-challans`:

- A save that asks to post (`status: 'POSTED'` or `post: true`) runs **save + preflight + post in one DB transaction**.
- On any refusal, roll it all back: no header, no lines, **no number consumed**, nothing in the list.
- The refusal comes back exactly as `/post` reports it today (codes, line numbers, the WARN list and `overrides[]`).
- `/stock/opening` and `/stock/adjustment` already work this way; please copy that behaviour.
- The existing `/post` route stays, for a draft someone saved on purpose.

Also notes 94 **Part A** (small): a new document saved by that same posting request needs the **Post** right, not Create. Today `/receipts/create` and `/payments/create` refuse a Post-only user (`requireRight(... 'create')`). User VIGNESH is the live case: Post on 99 / 100, no Create, so the Receipt screen cannot make a receipt.

## Part C (found later on 2026-10-12): a Post-only cleanup is refused for want of Delete

When `/receipts/post` or `/payments/post` fails, the client deletes the draft it created a moment earlier. But `/receipts/delete` and `/payments/delete` require the **Delete** right (`receipt.controller.ts:467`, `payment.controller.ts:337`). A cashier with View + Create + Post and no Delete gets 403, so the draft stays.

**Live case:** user VIGNESH (menus 99/100: view, create, post; no edit, no delete). Receipt draft `01a1303d-96c7-7c75-a49d-6202a46b8caa`, created 18:41:45, is still DRAFT. The same failure for VKPOS at 18:37 (`01a13039-…`) was deleted correctly.

`/bills/delete` has no such check, so the sale bill is not affected.

**Asked, if Part B cannot come first:** let the creator delete their own DRAFT without the Delete right (`avh_created_by` = the caller, status DRAFT, no number). Part B makes this unnecessary.

## Client side

Until this is live, the Qt screens make sure no one is left with a draft: a failed post deletes the draft it made (sale bill done 2026-10-12; receipt, payment, physical stock, sale order and DC next). A user with Post but not Create gets no Save draft button. When B is live we switch every one of these screens to the single call, the way opening stock and stock adjustment already work.
