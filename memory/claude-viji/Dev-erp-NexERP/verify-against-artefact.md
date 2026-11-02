---
name: verify-against-artefact
description: Read the live response / the real file before writing code that consumes it — the method that would have prevented four bugs on 2026-09-22
metadata:
  node_type: feedback
  type: feedback
---

The user, after the third fix in a row on the same block of code:
*"why dont you all in one"*

Fair, and the cause was one habit: **I wrote code against the written spec and
against guessed names, instead of against the artefact.** Four bugs in one
session, each found by the user, each costing a round trip:

1. `ctx.canPost` — does not exist. `NexDialog`'s ctx has create/edit/delete/
   export/print; post/cancel/amend/override are SERVER rights.
2. `enterEditMode()` unlocked a POSTED bill without setting `m_amending`, so
   Save hit `/bills/create` and the server answered "use /bills/amend". The
   operator had done nothing wrong — the screen let them edit and then refused
   to save.
3. Gated Amend on `locks.editable.document`. The server computes it as
   `status === 'DRAFT' && amendable !== false`, so it is **false on every
   posted bill by design — that IS lock 1.** Amend is the verb that reopens
   it, so the gate asked "may I unlock this?" and answered "no, it is locked".
4. Audited the screen by grepping for widget names I invented (`frameParty`,
   `grpBillTo`) and reported real features as missing. The `.ui` had them under
   other names.

**Why:** I had a well-written plan and treated it as the source of truth. It
describes intent; the shipped code and the live response are what the client
actually meets.

**How to apply — before writing code that reads something, go and look at it:**

* **An API response:** call it. Token —
  `POST /auth/login {"usrLoginName":"VKPOS","usrPassword":"VKPOS"}` →
  `access_token`; then `curl -sk -H "Authorization: Bearer $T" ...`. Checking
  every field of `/bills/get` at once took two minutes and found both a dead
  `header` fallback I would never have noticed AND the missing
  `rights.retender`.
* **A widget:** list the real names —
  `grep -oE 'name="[a-z]+[A-Za-z0-9_]*"' x.ui | sort -u` — never grep for a
  name you expect.
* **A DTO / a column:** read it (`information_schema`, the `.dto.ts`), do not
  infer it from the field's purpose.

**Also: never claim a build passed from file timestamps.** I reported "built
clean" off an object and binary that predated the failing compile. Read the
build OUTPUT for `Linking CXX executable` or the error.

**And in scripted edits, always `assert old in s` before `.replace()`.** One
patch without it silently did nothing (the member was `QMap`, not `QHash`) and
I moved on believing it had applied.
