# Memory Index

- [ERP Restructure Progress](project_restructure_progress.md) — Phases 1–3 complete; what was split, what imports changed, what's next (Phase 4)
- [Pasted design notes need verification](erp-pasted-design-notes-need-verification.md) — the SQL notes handed over for migrations are drafted against an idealised schema; check every reference against the live DB first
- [Partition helper correction](erp-partition-helper-correction.md) — TWO helpers exist; ensure_acc_year_partitions is the one the app calls, and new partitioned tables must be added to its explicit list
- [Sales endpoints implemented 2026-09-22](erp-sales-endpoints-implemented.md) — what the HANDOVER build left as stubs (GST gateway, day-close), assumed menu ids, shadow stock voucher, new error code
- [Workspace layout on Prathap's PC](erp-workspace-layout-prathap-pc.md) — 3 repos + DB + share on .106 after the 10-28 move; what a reboot kills (/tmp, :3011, :3001) and how to resume
- [Consolidated notes+memory copy](erp-notes-memory-consolidated-copy.md) — snapshot of all ERP memory + share md/txt at Dev/erp/notes-and-memory, refreshed 2026-11-02; not the master, untracked, holds 2 credentials
- [ERP agents location](erp-agents-location.md) — 11 custom subagents: project copy in .claude/agents + original in ~/.claude-viji; keep permanently, never delete
- [Mobile app = React Native](erp-mobile-app-react-native-decision.md) — 2026-10-31 decision over Qt-QML; reuse facts, open questions, pilot-first order
- [User: Qt dev new to React](user-qt-developer-new-to-react.md) — explain React/RN in Qt terms
- [Qt Party Outstanding like Next.js (v50)](qt-party-outstanding-react-port.md) — 2026-10-31 port; in NexERP main since 11-02 (now b093677, unpushed); v50 NOT published
- [Qt user menu like Next.js (v49)](qt-user-menu-react-look-v49.md) — 2026-10-31 main-window user menu; v49 LIVE on dev; in NexERP main since 11-02 (now b093677, unpushed)
- [Finished work must land in the checkout](erp-finished-work-must-land-in-checkout.md) — user runs NexERP main / ERP client dev, never worktrees; ff their branch at job end
- [Qt parity screens in main (11-02)](qt-parity-screens-in-main.md) — Audit Logs/Day Book/Templates + fix batch A1–C2 in NexERP main b093677 (unpushed); theme blocks unpublished
- [PO entry landing cost (11-02)](po-entry-landing-cost.md) — 6 PO points + charges spread on lines with landing cost; server DEV-v1 bbef3f41 + NexERP main bcd844a unpushed; Item col cramped
