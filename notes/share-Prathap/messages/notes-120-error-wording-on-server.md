# Notes 120: The plain error wording lives on the server (2026-10-15)

**STATUS:** DONE, verified live 2026-10-15: GET /error-messages (version 2026-10-15.1, 110 rows), `user` blocks on HOLD_NOT_FOUND, VALIDATION_*, TILL_REMIT_OVER_BALANCE. Qt prefers the `user` block.

## Why

Notes 119 gave every refusal a `code`. Today the plain words for those codes ("Cannot take payment · Your till is waiting for supervisor OK. · Do: Call supervisor.") sit in a table **inside the Qt app** (`src/core/api/api_error_text.cpp`).

That works for Qt, but React and mobile read the same errors. Each would need its own copy, every wording change would be made two or three times, and the copies would drift apart. The user's call (10-15): **the server owns the wording; each app only shows it.**

The wording itself does not change. The user approves it in the share files (`till-error-wording.md`, `counter-error-wording.md`) and you load what they approve. Your `message` stays as it is, for developers: the apps show it small and grey under the plain words.

## 1 · A `user` block on every error

Beside `field`, `code` and `message`, each error carries the plain words, already chosen for this case:

```json
{ "success": false, "message": "Session VIJI PC-261015-01 is PENDING_APPROVAL: it takes no money",
  "errors": [ {
    "field": "sbSessionId", "code": "TILL_SESSION_NOT_OPEN",
    "message": "Session VIJI PC-261015-01 is PENDING_APPROVAL: it takes no money",
    "status": "PENDING_APPROVAL",
    "user": { "title": "Cannot take payment",
              "line": "Your till is waiting for supervisor OK.",
              "todo": "Call supervisor." } } ] }
```

- **One place builds it:** the exception filters (`all-exceptions.filter.ts`, `TillExceptionFilter` and the module filters), from the `code`, so no throw site changes.
- **No row for the code → no `user` block.** Never put the developer sentence into `user`. The app then shows your `message` under a plain title, as Qt does today.
- **The same goes for the `/validate` answers** (the sale bill's warning strip: `warnings[]` / `refusals[]`). Each note with a code gets its `user` block too.

## 2 · Variants: the same code, different words

A few codes need different words by an extra field on the error. The seed below marks them:

| Code | Variant | Why |
|---|---|---|
| `TILL_SESSION_NOT_OPEN` | `status=PENDING_APPROVAL / COUNTING / SUSPENDED / CLOSED`, `status=*` | a cashier at the counter: what to do depends on the till's state |
| `TILL_SESSION_NOT_OPEN` | `field=tssId` | the same code on the Cockpit (a supervisor): "Not now · Refresh" |

Variant order: an exact `field=` match first, then `status=`, then `status=*`, then the plain row.

**Prefix rows:** `HOLD_*`, `TENDER_LINE_*`, `TENDER_REF_*`, `VALIDATION_*` (the code means the app sent something wrong) apply only when the exact code has no row of its own.

## 3 · The catalogue: one place, and a route for the apps

- **Storage:** a small table (your schema call), e.g. `error_message(code, variant, lang, title, line, todo, is_active, modified_on)`, and a catalogue version.
- **`GET /error-messages?lang=en[&since=<version>]`:** every row, plus the version.
  - Rights: login only (no menu): every screen of every app needs it.
  - The apps cache it, so a counter working offline still shows the words for an error it raises locally, and an app can word an error from an older server.
- **Language:** `lang` from the query or `Accept-Language`; `en` only for now. Tamil later is new rows, no app change.

## 4 · Seed

`Shared/Prathap/messages/error-messages-seed.json`: 109 rows, version `2026-10-15.1`, made from the two approved wording files:
- every till code (notes 117 / 118 included);
- held bills, payment lines and form checks (notes 119);
- `INTERNAL_ERROR`;
- the variants and prefix rows above.

Each row is `{ code, variant, lang, title, line, todo }`.

**Wording changes later** come the same way: the user approves them in the share file, I send you the rows, and you load them. A new code you add gets a row only once the user has worded it; until then it simply has no `user` block.

## 5 · What stays in each app

Only what depends on the screen:
- the box layout and the grey line (status · code · field · your `message`);
- the screen's own "Bill not lost." line;
- which box the cursor goes back to (a wrong PIN clears the PIN box).

Qt switches to your `user` block when it is present and keeps its own table only as the fallback until this is live everywhere. Then that table goes.

## Not in this note

- Codes for the sale bill and voucher refusals already exist. Their wording (about 101 `SALES_…` codes, and the receipt / payment codes) is still to be written and approved; those rows are added to the catalogue as they are approved.
- Notes 119's open items (voucher header codes, `code` required in the helpers, office masters) stay in notes 119.
