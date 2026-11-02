---
name: ledger-group-tests
description: "tests/ledger/run.sh — the 61-check live harness for AccountGroupEntry + LedgerEntry, and what it seeds/cleans"
metadata: 
  node_type: memory
  type: project
  originSessionId: 710f791a-1472-4ec1-9cd3-20c6a06e4ac6
  modified: 2026-09-17T12:09:51.141Z
---

`tests/ledger/run.sh` + `tests/ledger/test_ledger_master.cpp`, added
2026-09-17. Same arrangement as `tests/cheques/run.sh` (link ninja's objects
minus `main.cpp.o`, run under `QT_QPA_PLATFORM=offscreen`). **61 checks, all
passing**; needs the live 192.168.0.106 box.

It creates its own group (`ZZ TEST GRP (ledger run)` under Sales Accounts), a
child, and a ledger, and **deletes all three at the end** — it also purges those
names at the start, so a run that dies mid-way cannot make the next
duplicate-name assertion pass for the wrong reason. Deletes are soft, so each
run leaves a `led_is_deleted` row behind; that is expected.

Covers: the account-group contract ([[acc-group-master-contract]]);
AccountGroupEntry filling + round-tripping an edit; the ledger form's `ledTaxId`
and the absence of the three dropped keys ([[ledger-tax-id-change]]); group →
ledger type for **all nine profile/nature pairs on the box**; section visibility
and combo locking per profile ([[ledger-type-derivation]]); a real create driven
from the form's own `collectJson()`; and an edit of the live ROUNDOFF ledger
(`01a08f3d-44bb-7ecd-87b5-95c1e0a18b2c`).

The harness helper worth reusing: `Answer`/`sync()` keeps the **HTTP status and
`errors[].field`**, which the existing `syncGet`/`syncPost` helpers throw away —
most of the assertions here are about which 400 comes back.

Unrelated: `tests/receipt/run.sh` shows **7 failures against the working tree
and 1 against HEAD** — that is the user's own in-progress receipt rework, not a
regression from the ledger work.
