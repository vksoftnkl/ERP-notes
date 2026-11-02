---
name: stale-transfer-http-suites-are-destructive
description: "Five old /stock/transfer HTTP e2e suites expect the pre-2026-09-28 engine gaps; given rights they really post, then raw-SQL teardowns delete docs/transit rows and leave orphan ledger rows — they 403 at menu 41 on purpose"
metadata:
  node_type: memory
  type: project
  originSessionId: 46d6391f-c740-4636-9dfe-826333b87402
  modified: 2026-10-26T04:55:40.780Z
---

`test/stock-transfer-http`, `transfer-sql-trace`, `transfer-in-preflight`, `transfer-all-endpoints-tables`
and `transfer-receive-real-data` were written when despatch/receive could not complete. Since the TS
engine (2026-09-28) they really post, so they fail their assertions AND their teardowns (DELETE the
receipt document, DELETE stock_transit, `UPDATE svh_status='DRAFT'`) strand the ledger rows.

Seen 2026-10-26 when notes 153 briefly granted them menu 41: lot 8568e4ab (Texttile Item, Viji's
live-check lot) went Coimbatore 100 → 96 (real branch despatch TRF/…/44c01ccc…/4, RECEIVED) and karur
(branch Counter2) 0 → 10 from two TRANSFER_IN ledger rows whose documents were deleted (src ids
01a1768c-6785-… +6, 01a1768c-8022-… +4). Repair was left to the user. Those two orphan rows (6 @ ₹20 from a SIMULATED transit row + 4 @ ₹500) are ALL of karur's opening holding: its ₹212 average (notes 154 #5) is theirs, and Viji's later Coimbatore ↔ karur round trips moved stock at it, so a repair now touches Coimbatore's average too.

**Why:** a test that fails is harmless; one whose teardown assumes nothing moved corrupts dev data.
**How to apply:** do NOT add menu-41 grants to these five (they 403 since notes 153 E3 and must stay
that way) and do not run them, until they are rewritten or retired. New transfer e2e work goes in a
suite whose teardown cancels through the API, like `test/transfer-loading-notes-153.e2e-spec.ts`.
See [[e2e-suites-run-on-live-branch]], [[notes-153-godown-transfer-loading]].
