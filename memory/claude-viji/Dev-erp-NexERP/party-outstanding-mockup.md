---
name: party-outstanding-mockup
description: Party-wise Outstanding report (2026-10-09) — mockup PNG + backend plan (9 routes) + React plan in share/report/; nothing built, mockup unreviewed
metadata:
  type: project
---
`/home/vk/Shared/Prathap/report/`: `party_outstanding_ui_mockup.py/.png`, `plan-backend-party-outstanding.md`, `plan-react-party-outstanding.md`, README section. No menu, no routes, no Qt code. Mockup not yet reviewed (once approved → [[mockup-fidelity]]).

Routes planned: `/api/v1/reports/party-outstanding/` options, parties, party, bills, bill-wise, bill-history, summary, due-calendar, export ([[new-module-own-urls]]). Menu proposed under 137 at pos 0.60 (next free id 279 on 10-09). 3.0 reference: pos/Reports/customeroutstandings.* (menus 97/112, grids 578/579/580).

Core rule (backend §4): pending(b,D) = bill − Σ abj rows with adj_date ≤ D (reversals are negative + carry the ORIGINAL date; PDC rows carry the cheque date, so the date filter gives the PDC rule) − derived counter tender. Chain rule: skip a bill that has a live OPENING child (carry-forward) in a year ≤ year(D).

Findings (backend §3), for Prathap:
1. Counter tender sets abl_alloc_amount with no abj row (settleInvoiceAllocation) — 35 bills on the box.
2. SUSPECTED: recomputeBills rebuilds alloc from rows only → would drop the tender part. Test given, not confirmed.
3. carryBills leaves the parent pending; getCreditSummary / open-items / recompute read all years → double count after first carry-forward (none run yet).
4. abl_salesman_id never stamped.
5. A bounce re-opens the original bills and adds a JOURNAL `BNC/<chq>` charge bill. There is no bill for the cheque amount; the mockup was corrected to show this.

Decisions O1–O7 open.
