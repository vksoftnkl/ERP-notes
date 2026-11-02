# Status: party master rules, 2026-10-29

Plan: `plan-party-master-rules.md` (not edited). Answers: `decisions-party-master-rules-2026-10-29.md` ("go with all recommendations"; Q9 = computed on read, Q7 = a). Server notes: `/home/vk/Shared/Prathap/purchase/notes-163-server-changes-2026-10-29.md`.

## Built and applied (phase 1, step 1)
1. Customer module bug fixes, COMMITTED by the user 10-29 13:04 to 13:05 (three commits, subject "feat(party): party master rules P1 + customer save fixes; ledger names reusable after delete"): null numbers become 0; price level / branch existence checks; blank name on update refused; actor always from the request; create name-clash code; DTO min/max + GSTIN (all GSTN types; state prefix for NORMAL only) / PAN / email; lookup TCS + unknown company. Customer-group and master-lookup too. Customer DTO and service specs extended (customer.service.spec +308 lines).
2. Migrations APPLIED 17:32: `110000_ledger_name_unique_skip_deleted` (also covers the Tally name), `120000_party_master_rules_p1`. The ck_ GST-type constraints are phase 2.
3. Repo-wide DTO own-undefined fix (`isPresentField`): staged in the server index, NOT committed. Details in notes-163 section 4 and memory dto-own-undefined-trap.

## Running now
Server build of the rest of the plan (delete rules, dual-party sync, GST type rules, read-time "last billed"): running when this was written. No result yet; nothing to report. I could not confirm which steps are done.

## Open / waiting
1. Error-message seed not run (DB catalogue 2026-10-29.1; file .4). The party wording rows from migration 120000 are in the DB; later ones are not.
2. Existing UNREGISTERED / OVERSEAS rows with a GST No are left as they are until their next save (decision Q5).
3. Phase 2: drop the `cus_billed_*` / `sup_billed_date` columns; supplier "last billed" from the last posted purchase invoice (Q10); the GST-type CHECKs.
4. Test counts for the party build: not available to me.

## Print template (related)
PARTY_OUTSTANDING_BILLWISE_A4 rev 2 published. Rev 1 died in the print-template status-only wipe. Script `print/party_outstanding_billwise_template.py`. VKPOS was granted 279, 297 and Override on 196.
