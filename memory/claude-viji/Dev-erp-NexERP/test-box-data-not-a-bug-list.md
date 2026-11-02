---
name: test-box-data-not-a-bug-list
description: "findings about ROWS on 192.168.0.106 are not work items — write asks against the schema and the API contract, never against the data"
metadata: 
  node_type: memory
  type: feedback
  originSessionId: 710f791a-1472-4ec1-9cd3-20c6a06e4ac6
  modified: 2026-09-17T14:34:00.405Z
---

**192.168.0.106 is a test box. Never turn something found in its DATA into a
task for Prathap.** Cite a row only as evidence that a hole is *reachable*,
say plainly that it is test data, and make the ask about the schema, the
constraint or the API contract that let it happen.

**Why:** the user said so on 2026-09-17, after I wrote note (24) leading with a
two-statement cleanup of a duplicate ledger pair and an argument built on "every
FK count is zero, so it is free right now". Both were beside the point — the
rows get reseeded, and the real hole was that `/customers/create` and
`/suppliers/create` each mint their own id, so the UI cannot produce a dual
party at all. That part of the note was the only part worth his time.
[[db-live-connection]] already said the schema is authoritative and the data is
not; I used the data anyway because it made a vivid argument.

**How to apply:**

- A row count is a fine *illustration* and a bad *premise*. "These four have no
  ledger" → the ask is the missing FK, not the four.
- Watch for arguments that only work on an empty box: "nothing references it
  yet, so the migration is free" is not a reason to do something now on a box
  whose data is disposable.
- Say which it is in the document itself. `ledger_group_changes.md` now opens
  with "every ask here is a SCHEMA or CONTRACT ask… not a cleanup list", and
  note (24) opens by saying the rows in it are test data and will be reseeded.
- The same care applies to *me*: don't propose a data repair the read-only
  `claude` role could not make anyway.

Exception: a row that proves a constraint is missing is still worth quoting —
just quote it as proof, and put the fix on the constraint.

**Happened AGAIN on 2026-09-18** — the user repeated "its a test db only". I
found that cancelling a receipt left its post-dated-cheque voucher posted, which
was a real behaviour finding and correctly raised as notes (26). But I then
counted the rows it had left behind (9 vouchers, 5,400) and **asked Prathap how
he wanted them unwound** — a cleanup task on disposable data, exactly what this
memory says not to do. Withdrawn in notes (27).

So the trap is not only "don't lead with rows". It is: **once the behaviour is
fixed, the rows it already produced are finished business.** Do not carry them
forward as an open item, a follow-up question, or a "still outstanding" line in
a summary. The fix closes the finding; the residue is not a second finding.

The same instinct made me spend several tool calls cancelling my own test
receipts one at a time. Tests SHOULD clean up after themselves — that keeps a
later assertion from passing for the wrong reason — but tidying the box by hand
afterwards is not work, and reporting the tidy-up is not progress.


2026-09-26, broken again: I raised "a cancelled voucher's bill stays OPEN" from rows on the box, then tabulated deleted bills box-wide. User: "its a test data only". Before writing any ask, check it against the CODE/contract and stop there; never count box rows to size an issue.
