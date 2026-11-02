---
name: e2e-suites-run-on-live-branch
description: "The e2e company/branch (Acme Foods / Coimbatore, helpers/payment-e2e) IS the user's live till data; a teardown may delete only rows it can prove it created — notes 115: till-money-paths deleted till_event by tev_day_id and wiped the live journal (2026-10-12 and 10-14, my own runs)"
metadata:
  node_type: memory
  type: feedback
  originSessionId: c78c5194-a12d-48dd-a125-6bfccf139357
  modified: 2026-10-14T13:09:08.976Z
---

The e2e suites run against the dev DB on the same company and branch the user works in from the Qt
client (`test/helpers/payment-e2e.ts` COMPANY / BRANCH = 019c8ea6…). A suite's session joins the
branch's real business day. So a teardown may delete only by ids it owns — its sessions, its
devices, its documents — never by a shared key (business day, branch, date, event code).

**Why:** notes 115 (2026-10-14). `till-money-paths-http` afterAll ran
`DELETE FROM accounts.till_event WHERE tev_day_id = <day>` unguarded. My till-suite runs for notes
113/114 (~17:53 on 10-14, and a run on 10-12) wiped every real session's audit journal of those
days (only `till_event`; counts, movements and vouchers were intact). Autovacuum ran at 17:53:55,
so the heap was gone at once. There is no backup and no WAL archive (archive_mode off). The only
copy was the 4 recycled WAL segments still in pg_wal (orig 5/A1–A4, 12-10 19:53 → 14-10 17:57),
which I copied out (pg_read_binary_file) to the session scratchpad `notes115/seg/`. Decoding them
(`pg_waldump --save-fullpage`) was blocked by the auto-mode classifier, so it waits for the user.

**How to apply:**
- Fixed 2026-10-14: `test/helpers/till-cleanup.ts` `dropTestDay()` drops a day only when no
  session and no event remain on it. All six till/expense/settlement suites now delete events
  only by session id and device id. money-paths snapshots the branch's days in beforeAll and
  never drops one it found.
- Before running any e2e suite that touches till/sales/accounts, diff the ids of the live rows
  it could reach (e.g. `select tev_id from accounts.till_event`) before and after. The fixed
  suites showed 0 missing / 0 left behind.
- A suite that must CLOSE a business day runs on a temporary branch of the e2e company
  (`test/till-day-close-http.e2e-spec.ts`): INSERT branch_master (copy br_state_code), its own
  devices / safe / counter; tenders and ledger maps are global so it just works. Its cleanup must
  also delete `accounts.acc_voucher_seq` rows of the branch (numbering creates them) or the branch
  delete fails, afterAll throws before `app.close()`, and jest hangs — run with `--forceExit` and
  output to a file, never `| head` (that hid a 10-minute hang).
- Recycled WAL in pg_wal keeps old content (renamed to future names) until overwritten. The
  first page header's xlp_pageaddr gives the original segment; rename the copy for pg_waldump.
- See also [[e2e-suites-steal-default-safe]], [[sales-e2e-specs-rights-fixture]].
