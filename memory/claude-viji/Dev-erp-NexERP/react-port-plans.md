---
name: react-port-plans
description: "Where the React port plans live, what each covers, and the house format they follow"
metadata:
  type: reference
---

`/home/vk/Shared/Prathap/react-plans/` (NOT `reactplans/`, which holds one stale
copy of plan-react.md — ignore it). See [[db-schema-share-location]]: the share
is outside the git repo.

    plan-react.md                   Quotation Entry — and it OWNS the pricing
                                    engine every other sales plan imports
    plan-react-sale-order.md        Sale Order Entry
    plan-react-sale-bill.md         Sale Bill / "sales entry" (2026-09-12)
    plan-react-sale-tender.md       the settlement dialog, shared bill + order
    plan-react-customer-template.md one feature: save-as-default-template
    plan-react-printing.md          the print pipeline that does not exist yet
    plan-react-dropdown.md          the searchable picker component
    plan-react-app-settings.md      settings + overrides
    plan-react-tax-rate-master.md   GST Rate master, menu 40 (2026-09-12)
    plan-react-opening-balance.md   Opening Balances, menu 55 (2026-09-24) —
                                    DTOs read off live Swagger; §13 lists the
                                    open points (frozen-bill grace days untested)
    plan-react-receipt.md           Receipt Entry, menu 99 (2026-09-24) — domain/
                                    (identity, allocate, netAllocations, merges)
                                    first; §17 lists 9 open points incl. the Qt
                                    AFTER THIS RECEIPT formula being wrong for
                                    credits/TDS and /update-header never wired
    plan-react-user-admin.md        User Administration (user_entry.h + list, grid 62)
                                    (2026-09-26) — full-replace menus[] guarded
                                    (load gate + revoke confirm), menuVerbs cell
                                    gating, all 11 flags now accepted, §10 gaps
    plan-react-received-cheques.md  Received Cheques, menu 51 (2026-09-24) — shared
                                    domain/actions for menu 52; ActionSpec per verb;
                                    §14: Qt Replace-from-HELD sends no reason, drill-
                                    through uses session scope, 2nd-bounce 500 unretested

Outside that folder: `../report/plan-react-ledger-statement.md` — Ledger
Statement (2026-09-25), not a port (no Qt screen), kept beside its backend plan.

**The house format** (follow it — the user reads these as engineering
documents, not summaries): title + scope + Qt source paths + companion plans
that are NOT repeated + a one-line "the whole screen in one line" callout.
Then numbered sections, always including **Module layout** (a file tree),
**Backend gaps this screen inherits**, **Tests**, **Phases**, and **Do not
port**. Prose over bullets where a rule needs a reason; tables for field lists.

**What makes them worth writing:** each one records the traps that are invisible
in the code — why a rule lives on the server, which Qt workarounds are artefacts
of NexTable/QDialog rather than design, and which client behaviours are actually
defects that must not be carried over. State those explicitly in "Do not port".

Verify against the source before asserting; several earlier claims in the gap
docs turned out to be schema gaps rather than client bugs.
