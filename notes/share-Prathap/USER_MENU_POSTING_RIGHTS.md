# The posting rights cannot be granted to anyone. Small backend fix.

**2026-09-22.** Found while wiring the sale bill's Post / Cancel / Amend verbs.
Everything below was checked against the live box and the shipped code.

---

## The state today

```sql
SELECT count(*) rows,
       count(*) FILTER (WHERE um_can_post)     can_post,
       count(*) FILTER (WHERE um_can_cancel)   can_cancel,
       count(*) FILTER (WHERE um_can_amend)    can_amend,
       count(*) FILTER (WHERE um_can_override) can_override,
       count(*) FILTER (WHERE um_can_retender) can_retender
  FROM public.user_menus;

 rows | can_post | can_cancel | can_amend | can_override | can_retender
 1111 |        0 |          0 |         0 |            0 |            0
```

**Nobody in the system can post, cancel, amend, override or re-tender
anything** — and there is no screen on which to grant it.

## Why

The columns exist. `30` added `um_can_post`, `um_can_cancel`, `um_can_amend`,
`um_can_override`, `um_can_retender` — all `NOT NULL DEFAULT false`.

**The API does not know about them.** Checked in the read-only snapshot:

| file | has |
|---|---|
| `dto/save-user-administration.dto.ts` · `SaveUserMenuDto` | umCanView, umCanCreate, umCanEdit, umCanDelete, umCanPrint, umCanExport |
| `dto/user-administration-response.dto.ts` · `UserMenuPayloadDto` | the same six |

So the five are never read and never written, and every row sits at the
column default.

**And saving a user's menus is a FULL REPLACE**, so this is not merely a gap:
the moment anyone grants these by hand in SQL, the next save from the user
admin screen silently takes them away again.

---

## The fix — 5 fields, 2 DTOs, 1 mapping

1. **`SaveUserMenuDto`** — add the five as `@IsOptional() @IsBoolean()`,
   matching the six already there.
2. **`UserMenuPayloadDto`** — add the same five so `/get` returns them.
3. **`user-administration.service.ts`** — carry them through the same
   read and the same replace as the existing six.

Nothing else. No migration: the columns are already there.

---

## Why the client cannot do it alone, and what it is doing meanwhile

`main.ts:190` sets **`forbidNonWhitelisted: true`**. So a client that sends
`umCanPost` today does not get it quietly ignored — it gets

```
400  property umCanPost should not exist
```

on the WHOLE save. **That would break user administration outright**, which is
worse than the gap it fixes.

So the Qt side is built and waiting behind one flag —
`kServerHasPostingRights` in `user_menu_widget.cpp`. The five columns exist in
the tree with their tooltips, are read back in `applyPermissions()` and are
sent in `fieldValue()`; while the flag is false they are **hidden** (not
greyed — a greyed checkbox on a permissions screen reads as "denied", and an
administrator would believe they had looked at this and decided).

**Tell me when the three files above are in and I flip the flag.** One line.

---

## What it blocks

The sale bill's new verbs read `/bills/get`'s `rights` block, which is these
same `user_menus` flags. With every one false, the screen correctly greys Post,
Cancel and Amend for every user — so **the bill screen cannot be tested end to
end until this lands**, however well the routes work.

Suggested once the DTOs are in: grant the five on menu 12 to the test user, so
the bill screen can be driven for real.

---

## FOLLOW-UP 2026-09-22 evening — `rights.retender` is missing from `/bills/get`

The five DTO fields landed and are live; thank you. One gap left, found by
reading a real `/bills/get` response rather than the spec:

```jsonc
"rights": { "post": true, "cancel": true, "amend": true, "override": true }
```

**Four keys. `retender` is not among them** — while `um_can_retender` exists on
`user_menus`, is granted to `tester1` on menu 12, and `menu_verbs` carries
RETENDER on that menu alone.

`sales-doc-blocks.service.ts` builds the `rights` block; it needs the fifth
key, read from the same `user_menus` row as the other four.

**What it blocks:** §1.7b, the re-tender dialog — *"the counter's most common
correction"*. The plan is explicit that Ctrl+F6 must be enabled by
`um_can_retender` and **not** by `amend`, so a till operator can fix their own
counter in thirty seconds without being able to edit sales. With the key
absent the client cannot tell the two apart and would have to either grey the
feature for everyone or light it for anyone who can amend — and the second is
exactly the conflation the separate right exists to prevent.

Everything else in the block checked out against the live response: `locks`
carries returns / allocations / dayClosed / irnLive / ewbLive /
irnCancelWindowUntil / ewbValidUpto / editable{document,transportBand}, and the
header comes back FLAT (sbStatus, sbRevisionNo, sbBillRefno at the top level,
no `header` sub-object).

