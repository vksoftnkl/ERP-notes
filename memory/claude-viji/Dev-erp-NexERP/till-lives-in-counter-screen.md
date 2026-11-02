---
name: till-lives-in-counter-screen
description: The till's cashier UI (strip, till actions) belongs in the one-screen Counter (Till mode), NOT bolted onto office sale bill / receipt / payment
metadata:
  type: feedback
---
User 2026-10-09, while I was building S2 (till strip + Ctrl+F11 Till menu attached to the office sale bill, receipt, payment): "whats the need of till menu i ve showed you a one screen mode".

**Why:** the Counter screen mockups (share/till/counter_entry_ui_mockup.png + counter_entry_modes_ui_mockup.png, [[counter-screen-analysis]]) already carry the till: top bar = Counter · Shift · Cashier · Drawer gauge (that IS the S2 strip), key bar holds the till keys. The analysis also said office screens stay untouched. TILL_DESIGN §8 S2 predates the counter screen.

**How to apply:** before building any till cashier screen, check it against the Counter (Till mode) mockups first; don't add till UI to office screens. Reverted 10-09: TillStrip::attachTo on sale bill/receipt/payment. Kept: sbSessionId/soSessionId send the S1 till session id when present (server-side correctness). src/modules/sales/till/strip/ (TillStrip, TillMenu) still in the tree, unused — pending the user's call.
