---
name: cheque-status-doc-type-by-tra-type
description: "acc_pdc_register holds BOTH directions (apd_tra_type R/P); txn_status_log steps file as CHEQUE_RECEIVED or CHEQUE_ISSUED via chequeDocTypeOf(); a relabel keyed on 'id is in the register' mislabels issued cheques"
metadata:
  type: project
---

Fixed 2026-09-28. `logChequeStatus` (cheques.utils.ts) is shared by the received-cheques
module AND issuedCheques; it now files by `chequeDocTypeOf(apdTraType)` (P → CHEQUE_ISSUED,
else CHEQUE_RECEIVED), and each module's `history` reads the same type. Migration
20260928100000 §6 had relabelled every register-backed OTHER row to CHEQUE_RECEIVED;
20260928173000 moved the P rows to CHEQUE_ISSUED (applied on localhost).

**Why:** the issued-cheques history route read OTHER and returned nothing after the
Phase-1 relabel — the only http e2e case that caught it was "present / return / cancel"
in test/vouchers-register-http.e2e-spec.ts.

**How to apply:** any query that says "this doc id is a cheque" must also say which
direction; a writer and its reader must share the doc-type function, never a literal.
