---
name: react-sale-bill-plan
description: The React plan for the whole sale bill (sales entry) — share/sales/plan-react-sale-bill.md REV 2 (2026-09-25), supersedes react-plans/plan-react-sale-bill.md + sale-tender for the bill; its 12 open decisions and the new defects it found
metadata:
  type: project
---
`/home/vk/Shared/Prathap/sales/plan-react-sale-bill.md` (32 sections, ~2,850 lines, 2026-09-25). The user asked for it in the SALES folder, not react-plans/. It was built from 7 parallel extractions of the Qt screen + the NestJS source (the notes were in the session scratchpad and are gone now). Based on the uncommitted Qt tree of 2026-09-25.

It supersedes `react-plans/plan-react-sale-bill.md` (predates A3) and, for the bill, `plan-react-sale-tender.md`. The order still uses the shared TenderDialog. The pricing engine stays owned by `react-plans/plan-react.md` §10. One change is asked there: split CGST/SGST from one rounded figure.

Open decisions Q1-Q12 (§28): credit-anyway, server holds, quotation double-billing, AllowNegative, source rate, walk-in GSTIN, loyalty points, list grid, freight weight, unloading/Qtl, re-tender button, TEMP_CR term.

New suspected defects to verify live before reporting (§27):
- CHG-TAX: /charges/get has no numeric tax rate since ledGstRate was dropped.
- T-VOID: /bills/get neither filters nor flags voided tenders.
- E-WAY-DB: the e-way check reads the stored band.
- The no-customer refusal uses the code SALES_PAN_REQUIRED.

Client defects D1-D9 are in §15. The web client cannot post until login returns a registered device (DEV). Related: [[sale-bill-upgrade]], [[react-port-plans]], [[sale-bill-live-test]].

**Imports split out (2026-09-26):** `share/sales/plan-react-sale-bill-imports.md` (10 sections, ~490 lines) OVERRIDES this plan's §13.1–13.4, Q4, Q5 and the AllowNegative lines of §19/§32. It covers: fill from the source line, never re-price; the stock-only refresh with godown_id; the sqi/soi/sdi key maps; the DC per-qty discount rebuild. The user said "separate plan file, do not touch original", so the REV 2 file stays byte-identical. Put future import changes in the imports file.
