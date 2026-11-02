---
name: till-printouts-notes-135
description: notes 135 LIVE 10-17; TillMovement80 PUBLISHED + Qt buttons built; TillRemitA5 DRAFT (no REMIT on the box to render)
metadata:
  type: project
---
2026-10-17: Shared/Prathap/till/notes-135-till-movement-slip-remit-memo.md asks Prathap for purposes TILL_MOVEMENT (docId tcm_id; providers till.movement.header with gives/receives signature names + till.movement.lines) and TILL_REMIT (till.remit.header + lines), print_count bumped per render like tss. Z/X/day Z are live (notes 117 §7, 121).

Facts found: designer has `numToWords` (no amount_words field needed); company.profile / branch.profile providers exist; NO ledger holds bank account no/IFSC (asked: new ledger fields vs blank line). Open question: how EXCHANGE stores its IN/OUT sides.

**When live:** layouts TillMovement80 + TillRemitA5 (script beside till_z_x_templates.py), buttons: S3 Post & print, Cockpit Money tab Print slip, Cash Office Print memo. Related: [[till-cash-office-287]], [[till-session-cockpit-screen]].

**10-17 later — Prathap ran 135:** purposes TILL_MOVEMENT (113) / TILL_REMIT (114, 2 copies "Bank / Agency Copy,Store Copy" → `{{ ctx.copyLabel }}`), providers till.movement.* / till.remit.*, fields in nexerp-api-ro src/modules/till/README.md "Printing". EXCHANGE = two till_count rows (tcm_count_id = IN). Bank a/c fields null (option b).
**Layouts:** Shared/Prathap/print/till_slip_memo_templates.py (`--post [--draft]`, `ONLY=<code>`, `--render <dir> <purpose> <tcmId>`; ids in till_slip_memo_ids.json; imports till_z_x_templates helpers; VOID = watermark). TillMovement80 PUBLISHED + rendered OK on tdr00003/tfl00005/tdr00004/tfl00001. TillRemitA5 DRAFT only: the box has 0 REMIT rows (safe ZTS at −8,000, a remit would be refused) — rendering it needs a safe count + a real remit. **User 10-17: "dont publish now"** — leave TillRemitA5 as DRAFT, no safe count / remit test until the user brings it back.
**Qt (uncommitted):** DocumentPrint::Purpose::TillMovement/TillRemit; Cockpit btnPrintSlip (any till movement on Money tab, currentSlip()); Cash Movement dialog asks "Print the slip now?" after post; Cash Office btnPrintMemo (Remittances tab, any state) + asks after a remit.
Change top-up prints "Given by" blank (server gives = null there; the giver is the other till's cashier) — minor, not raised.
