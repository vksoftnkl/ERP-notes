---
name: counter-never-blocked-269st
description: "User decision 10-15 — legal cash checks (269ST, PAN/Form 60) must never block billing at the counter; 269ST work on hold, later WARN-only"
metadata:
  node_type: memory
  type: project
  originSessionId: 3ef83d0b-5600-4825-9b13-0befb9e93814
  modified: 2026-10-15T13:55:12.270Z
---

269ST cash limit / PAN-Form 60 / HSN-digits checks are commented out in the server's bill posting (bill-lifecycle.service.ts step 7). User's call 2026-10-15: "we can work on backend later — we can't make the customer wait at the counter." Notes 123 (Shared/Prathap/sales/notes-123-269st-cash-limit.md) is ON HOLD.

**Why:** supermarket clients bill everything to the walk-in "cash customer"; many don't file GST; a refusal at the counter stops the queue. The user knows the law (269ST is Income Tax, not GST).

**How to apply:** don't push 269ST/PAN refusals. When it is picked up: WARN only first, walk-in = per-bill cash, named party = day total; REFUSE only by a later per-company decision. The sale bill suite's 3 area-D 269ST checks are expected to fail until then — not a regression. Related: [[law-params-per-company]], [[simple-error-messages]].
