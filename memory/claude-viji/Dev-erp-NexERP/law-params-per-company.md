---
name: law-params-per-company
description: "Every GST / income-tax threshold, date and rate is resolved PER COMPANY on a date through statutory_limits (+ company override) and the companys applicability flags — never a code constant, never a global setting"
metadata: 
  node_type: memory
  type: feedback
  originSessionId: 0b00a13b-6c14-4cbf-9fa7-d864e9a434dd
  modified: 2026-09-20T06:16:32.919Z
---

User, 2026-09-20, after the compliance audit of the sales plan: **"all the gst and it laws
should be based on companys only."**

**Why:** a chain runs several companies with different turnover classes, registration
types and states; e-invoicing starts on a company-specific date; laws change by Finance
Act (206C(1H) was removed from 2025-04-01). A constant in code or one global setting is
wrong for at least one company the day it ships and needs a release to fix.

**How to apply:** any guard that needs a law figure reads it ONLY through
`fn_statutory_limit(company_id, code, on_date, applies_to)` — company override row first,
shipped pack (`stl_company_id IS NULL`) second, `applies_to` matched to the company's
state — and the refusal message names the `stl_code`, value and date. Applicability
(does it apply at all) comes from the `companys` row (`comp_gst_reg_type`,
`comp_einvoice_*`, `comp_eway_*`, `comp_aato_class`). Enforcement (WARN/REFUSE) is per
company too. The full layer model is `share/accounts/statutory_masters.md`; the sales
application of it is `share/sales/analysis_2026-09-20.md` §8b.5. See
[[statutory-masters-model]], [[sales-txn-3-0-reference-scope]].
