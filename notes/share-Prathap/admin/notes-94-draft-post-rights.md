# Notes 94 — Post without Create = "no drafts"; save & post in ONE transaction

**From:** Qt client (Viji / Claude) · **Date:** 2026-10-07 · **For:** Prathap
**Diagram:** `notes-94-permission-tables.png` (this folder) — the three tables and who reads them

## Why

A single shop where the owner does every job keeps pressing **Save draft** instead of **Save & Post**,
and drafts pile up. The user does not want a new setting for this: the rights we already have
should express it.

On a transaction menu the ticks in User Administration now mean:

| Right  | Allows                                   |
|--------|------------------------------------------|
| Create | save a **draft**                         |
| Edit   | change a saved draft                     |
| Delete | delete / cancel a **draft**              |
| Post   | post — **a brand-new document too**      |
| Cancel | reverse a **posted** document            |
| Amend  | edit a posted document                   |

So **Post ticked, Create unticked** = the owner: New → Save & Post, no Save draft.
**Create ticked, Post unticked** = an entry operator who leaves drafts for a supervisor.

The Qt client applies this on opening stock (44), physical stock (45) and stock adjustment (264)
today. The rest of the transaction screens follow once A and B below are live.

## A. A new document may be saved with Create OR Post

Today these refuse a Post-only user on the first call of Save & Post:

- `receipt.controller.ts:343` — `requireRight(avhVoucherId ? 'edit' : 'create', …)`
- `payment.controller.ts:244` — same
- `voucher-register.service.ts:246` and `:391` — `!rights.create`

Rule wanted: **saving a new document needs Create, unless the same request posts it, then Post.**
Saving an existing draft stays Edit. With B below, "the same request posts it" is a flag on the save.

## B. Save & Post must be ONE transaction (the important one)

When a post fails, the user must **not** be left with a draft they never asked for. A Post-only
user does not even know drafts exist.

| Route | Today |
|---|---|
| `/stock/opening` | one call: `header.status = 'POSTED'` saves and posts atomically ✔ |
| `/stock/adjustment` | one call: `header.status = 'POSTED'` ✔ |
| `/stock/physical` | two calls: `/create` (always DRAFT) then `/post` ✘ |
| `/receipts`, `/payments` | two calls: `/create` then `/post` (needs `avhVoucherId`) ✘ |
| `/bills`, `/sale-orders`, `/delivery-challans` | two calls: `/create` then `/post` ✘ |
| `/vouchers` (register) | please confirm |

Please give each ✘ route the opening-stock behaviour: a save that asks to post (`status: 'POSTED'`
or `post: true`) runs save + preflight + post in **one DB transaction**, and on any refusal rolls
the whole thing back — no header, no lines, no number consumed, nothing in the list. The refusal
comes back exactly as `/post` reports it today (codes, line numbers, WARN list for overrides).

The existing `/post` route stays for posting a draft that was saved on purpose.

## C. Stock routes check no rights at all

`stocks/opening-stock-voucher`, `stocks/physical-stock-voucher` and the adjustment route never read
`user_menus`. Please apply the same table on menus 44 / 45 / 264 via `common/posting/rights.ts`
(`loadRights`): Create/Edit on save, Post on post (or save-with-post), Delete on cancelling a
draft, Cancel on cancelling a posted one. Today the client is the only gate.

## Live note

On the box, VKPOS / ravi / raman / VIGNESH hold Create+Edit+Delete but **no** Post/Cancel/Amend on
44 and 45; VIJAY and VKPOS1 hold all. Nothing to change — just so a test with VKPOS failing to post
is expected.
