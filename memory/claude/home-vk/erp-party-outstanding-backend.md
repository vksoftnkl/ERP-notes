---
name: erp-party-outstanding-backend
description: "Party-wise Outstanding report backend (reports/party-outstanding, menu 279) built 2026-10-09 uncommitted; plan lives on the SMB share; open questions it left"
metadata:
  node_type: memory
  type: project
  originSessionId: 89d13473-da82-44de-a1f8-222ea8f7c3fb
  modified: 2026-10-09T08:14:25.612Z
---

Built 2026-10-09 in `~/Dev/erp/ERP server/src/modules/reports/party-outstanding` from `Prathap/report/plan-backend-party-outstanding.md`. The share is mounted at `/run/user/1000/gvfs/smb-share:server=192.168.0.103,share=shared/Prathap/report`; the React plan sits beside it.

**State:** all uncommitted; the till changes staged by an earlier session are untouched. Migration `20261009100000_party_outstanding_menu` (menu 279 under 137) is applied on the dev DB. dist is rebuilt; the :3000/:3011 servers were not restarted. Unit spec: 27 tests. `test/party-outstanding-http.e2e-spec.ts`: 26 tests, passing. A brute-force SQL check also agreed with the report.

**Open for the user:**
- O1: sale-bill counter tenders still write no adjustment row; the report derives them.
- The recompute suspicion in plan §3.1 is unverified.
- O7: the other bill readers will double-count after carry-forward.
- `arm_collection_days` is assumed ISO, Mon=1..Sun=7. Stored values run 1–7. The Qt cust-group/supplier widget uses Mon=0.

**Gotcha:** in HTTP e2e, refusal codes arrive under `body.message.errors` (AllExceptionsFilter wraps the response), not `body.errors`.

Related: [[erp-server-http-testing-without-credentials]], [[erp-readonly-service-harness]].
