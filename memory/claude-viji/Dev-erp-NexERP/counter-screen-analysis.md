---
name: counter-screen-analysis
description: 2026-10-07 measured analysis of a one-screen counter (Bill/Return/DC/Order/Quote) — inside NexERP vs a separate app (EPOSGo); the facts and the conclusion. Pending user answers on hardware and offline.
metadata:
  node_type: memory
  type: project
  originSessionId: 437b9134-899f-4f46-94f2-a058376f179c
  modified: 2026-10-07T07:15:00.373Z
---

**The question:** the user wants an EPOSGo-style single counter screen covering Bill, Return, DC, Order and Quote, with conversion. It must be fast. Office screens are not to be touched. Should it be built inside NexERP or as a separate app? The user pushed back on an unsupported recommendation ("analyze don say blindly"), so this was measured.

**Write-up:** `share/till/analysis-counter-screen.md`.

**EPOSGo (/home/vk/Projects/NextLvl/EPOSGo):**
- 38k lines in total; `mainwindow.cpp` alone is 4,739.
- It covers Bill/Order/DC plus three conversions: order→bill, order→DC, DC→bill.
- It has NO quotation and NO return.
- It calls DB functions directly, so it cannot be reused under the [[sql-vs-nestjs-rule]].
- Its copied files have drifted from 3.0: tillsessiondetail by 114 lines.
- It holds two copies of its own widgets, 150+ lines apart.
- The "Office" button runs a shell script to start the other app.

**NexERP:**
- core (7.5k lines) and shared (25k) are reusable. Login has no module dependency.
- `main.cpp` builds `MenuWindow` after login; that is the Till-mode hook.
- The single CMake target plus `form_manager` (which includes every module) is the only obstacle to a second executable.
- The server already has every route and conversion: bill open-sources, DC open-for-bill, return bill-lines, hold convert.
- Line DTO overlap with the bill: order 83%, quote 84%, DC 60%, return 59%.
- The four office entry screens (20.7k lines) share 42 same-named functions that have ALREADY DRIFTED: GST/cess code is 54–56% alike, qty recalc 18–27%.
- No Qt sale-return screen exists yet.
- Speed is the same in every option: 2 GETs per scan, 2–4 POSTs per save.

**Conclusion given:**
- Not a separate app: it would rewrite ~32.7k lines and give no speed gain.
- Build a Counter module inside NexERP (Till mode). A second executable from the same repo stays possible later.
- Build the counter on an extracted, screen-free `SalesLineEngine`, never on copied functions.
- Speed comes from the screen flow, plus a server create-and-post call.
- **Open questions to the user:** what hardware the counters run on, and whether any counter runs without the store server.

**Mockups drawn 2026-10-07** (user: "just mockup"): `till/counter_entry_ui_mockups.py` produces two images.
- `counter_entry_ui_mockup.png`: the Till-mode full screen in Bill mode. It has a top bar, Save-as tabs (Alt+B/R/D/O/Q), a "loaded from" chip, a customer line, the scan box, one grid, a big NET panel, offers, quick items, and a key bar (F10 Pay, F2, F8/F9 hold, F12 load/convert, F7, F4, Ctrl+F6, F11, Esc).
- `counter_entry_modes_ui_mockup.png`: the Pay, Return (scan bill), Order (pickup/doorstep/e-com plus advance), DC, Quote and F12 load/convert panels.
- Figures: NET 1,232; gross 1,310; offers −78; customer saved 123 on MRP.
- None of it is reviewed yet.
