# NexERP memory — the whole product (PostgreSQL + NestJS + Qt + React), merged 2026-10-30

**Read first, every session:** [START HERE](session-resume.md) · [After the move — facts that win](after-move-facts.md) · [Project map](project-map.md) · [How the user works](user-working-style.md)
**Before working in an area, read its full index:** Qt + DB → [qt-db-index.md](qt-db-index.md) (188 notes) · NestJS → [server/INDEX.md](server/INDEX.md) (112) · React → [client/INDEX.md](client/INDEX.md) (134) · cross-repo / deploy / history → [workspace/INDEX.md](workspace/INDEX.md) (35)

## House rules (all layers)
- [Qt first, React after the user tests Qt](user-working-style.md) — record "React owes" in session-resume; one client/erp-qt-<sha>-port.md per ported range
- [Commit straight to main](feedback-commit-to-main.md) · [Commit messages in plain text](server/commit-message-plain-text.md) · [Prathap server access](prathap-server-access.md) — commit only when asked; by path; migrations after the user's OK
- [Run commands with `!`](user-run-commands-with-bang.md) — the claude DB role is read-only; migrations, secrets, settings = a `!` line for the user, then verify
- [SQL vs NestJS rule](sql-vs-nestjs-rule.md) — NO DB functions/triggers; CHECKs/indexes stay · [Stock engine is TypeScript](server/stock-engine-ddl-lives-outside-repo.md)
- [API house conventions](api-house-conventions.md) · [New module = own URLs](new-module-own-urls.md) — one partyId (= led_id), no /list routes, never reuse old route families
- [Verify against the artefact](verify-against-artefact.md) · [Mockup fidelity](mockup-fidelity.md) · [Plans: separate file](plans-separate-file.md) — read the live response; the approved PNG is the spec; never revise a share plan in place
- [Grid runner binding rules](grid-runner-binding-rules.md) · [Grid SQL: no comments](grid-sql-no-comments.md) · [grid_param convention](server/configured-grid-param-convention.md) — send every token; no `--` or `/*` in grid_sql
- [Config rows are dev-DB only](client/erp-live-config-drift.md) · [Configured grid registry](client/erp-configured-grid-registry.md) — grids/dropdowns/ui_tables by NAME; designer-made rows need INSERTs for live
- [DTO own-undefined trap](dto-own-undefined-trap.md) · [DTO hasOwnProperty always true](server/dto-hasownproperty-always-true.md) — use isPresentField; `?? null` wipes columns
- [Draft/Post rights rule](draft-post-rights-rule.md) · [Save & post in one call](server/save-and-post-one-call-notes-106.md) — Create=draft, Post=post; post:true
- [Simple error messages](simple-error-messages.md) · [Error codes on every refusal](server/error-codes-notes-119.md) — plain English + grey server line; words by errors[].code
- [Times are AM/PM](time-format-ampm.md) · [Dates dd/MM/yyyy](client/erp-date-format-house-rule.md) · [DB session is IST](server/db-session-timezone-ist.md)
- [Blank company = shared](company-scope-nullable.md) · [Never read item_tax_master](server/avoid-item-tax-master.md) · [Money stays Decimal](server/money-decimal-rule-sales.md)
- [Several sessions edit at once](server/concurrent-claude-sessions.md) · [Workspace layout: never git stash](client/erp-workspace-layout.md) — check mtimes; unstage only your own files
- [Don't re-run every suite per change](dont-test-every-change.md) · [Test-box data is not a bug list](test-box-data-not-a-bug-list.md) · [Sale bill layout REVOKED — ask WHY first](sale-bill-new-layout.md) · [Stock designs are mine](stock-section-ownership.md)

## Qt desktop + DB — key notes (full list: qt-db-index.md)
- [Menu Master = only what the code opens](menu-master-code-only.md) — 137 unbuilt menus removed 10-30 (migration 20261030200000); re-add file in share admin/menu-cleanup-20261030; menu_separator is NOT a divider row
- [Txn main view architecture](txn-main-view-architecture.md) · [Txn keys standard](txn-keys-standard.md) · [Accounts screens own .ui](accounts-screens-own-ui-rule.md) · [Txn entry screens on .ui (purchase to convert)](txn-screens-on-ui-rule.md) · [Prev/Next walk order](txn-adjacent-walk-order.md)
- [Widget signal wiring](widget-signal-wiring-rule.md) · [Grid entry = delegate](grid-entry-delegate-preference.md) · [Popup Enter rule](popup-enter-navigation-rule.md) · [NexTable grid traps](nextable-grid-traps.md) · [NexDialog stack crash](nexdialog-stack-crash.md)
- [Theme template decision](theme-template-decision.md) · [Theme shipping: API on dev, then a guarded migration](theme-shipping-workflow.md) · [Grid look = theme only](grid-look-theme-only.md) · [Theme master screen](theme-master-screen.md) · [Bool pills + widths %](api-grid-bool-chips-stretch.md) — v48 published 10-30 + migration 20261030190000
- [Qt offscreen test harness](qt-offscreen-test-harness.md) · [Qt test runners: repo root, run.sh writes](qt-test-runners-cwd.md) · [Widget config on .ui screens](widget-config-ui-screens.md)
- [Printing parity 10-30](printing-parity-2026-10-30.md) · [Printing UI screens](printing-ui-screens.md) · [Document print](document-print-built.md) · [Print designer parity](print-designer-parity.md)
- [Purchase module](purchase-module-analysis.md) · [Purchase build](purchase-module-build.md) · [Party master rules](party-master-rules.md) · [Till management](till-management-design.md) · [Stock small masters](stock-small-masters.md)
- [Receipt design](receipt-design.md) · [Payment screen](payment-screen-built.md) · [Voucher register](voucher-register-design.md) · [Sale bill upgrade](sale-bill-upgrade.md) · [Expense voucher 277](expense-voucher-screen.md)
- Schema designs: [sales](sales-backend-schema-design.md) · [stock](stock-schema-design.md) · [printing](printing-schema-design.md) · [posting ledgers](posting-ledger-schema-files.md) · [share location](db-schema-share-location.md)

## NestJS server — key notes (full list: server/INDEX.md)
- [Prisma schema is generated](server/prisma-schema-is-generated.md) · [Prisma migration gotchas](server/prisma-migration-gotchas.md) · [VSCode stale Prisma types](server/vscode-stale-prisma-types.md) — never hand-edit schema.prisma; tsc --noEmit
- [Dev box runs the API twice](server/dev-box-runs-two-apis.md) · [VPS deploy topology](server/vps-deploy-topology.md) · [Live VPS P3009](live-vps-sync-migration-p3009.md) · [Cloud sync is local-only](server/cloud-sync-is-local-only.md)
- [Pre-existing test failures](server/pre-existing-test-failures.md) · [E2E suites run on the live branch](server/e2e-suites-run-on-live-branch.md) · [HTTP-test guarded routes](server/erp-server-http-testing-without-credentials.md) · [Unit jest transpile-only](server/unit-jest-is-transpile-only.md)
- [Token scope = home company + device branch](server/token-scope-is-home-company-and-device-branch.md) · [Receipt/payment menu rights](server/receipt-payment-menu-rights.md) · [Transaction rights grantable](server/user-menu-transaction-rights.md)
- [Bill tables partitioned by acc_year](server/sale-bill-acc-year-partitions.md) · [Stock → accounts posting](server/stock-to-accounts-posting.md) · [Error wording on the server](server/error-wording-notes-120.md) · [Table registry](server/db-table-master-registry.md)

## React web — key notes (full list: client/INDEX.md; Qt→React ports = client/erp-qt-*-port.md)
- [CrudMasterPage extensions](client/erp-crud-master-extensions.md) · [CrudMasterPage column heuristics](client/erp-crud-master-grid-column-heuristics.md) · [Grid columns config](client/erp-grid-columns-config.md) · [List grid fit-to-width](client/erp-list-grid-fit-to-width.md)
- [Menu href mapping](client/erp-menu-href-mapping.md) · [Menu permission gating](client/erp-menu-permission-gating.md) · [Configured dropdowns](client/erp-configured-dropdowns.md) · [NexDropdown](client/erp-nex-dropdown-component.md)
- [Message popup, no toasts](client/erp-message-popup.md) · [Global UI scale](client/erp-global-ui-scale.md) · [Full-height page shell](client/erp-full-height-page-shell.md) · [Screen memory](client/erp-screen-memory.md)
- [Client UI browser verification](client/erp-client-ui-browser-verification.md) · [Lint baseline](client/erp-lint-baseline.md) · [Stale dist 404](client/erp-stale-dist-404.md) · [Server 500 debugging](client/erp-server-500-debugging.md)
- [Printing module](client/erp-printing-module.md) · [Print designer UI](client/erp-print-designer-ui.md) · [QtRPT parity in the designer](client/erp-print-qtrpt-parity.md) · [Template save clears header](client/erp-print-template-save-clears-header.md)
- Latest Qt ports: [e0c56ef purchase](client/erp-qt-e0c56ef-purchase-port.md) · [737e671 purchase](client/erp-qt-737e671-purchase-port.md) · [75c682b transfers](client/erp-qt-75c682b-port.md) · [a204dc8 TxnKeys](client/erp-qt-a204dc8-port.md) · [131b042 printing](client/erp-qt-131b042-port.md) · [e782e59 user Copy](client/erp-qt-e782e59-user-copy-port.md)

## Workspace, agents, infra
- [Workspace layout on Prathap's PC](workspace/erp-workspace-layout-prathap-pc.md) · [Move to Prathap PC](move-to-prathap-pc.md) · [Live VPS deploy](workspace/erp-live-vps-deploy.md) · [API endpoint docs](api-endpoint-docs.md)
- [Agent team](agent-team.md) · [Agent Office monitor](agent-office-monitor.md) · [Prathap profile: agents + hooks](prathap-profile-agents-hooks.md) — start sessions with `claude-viji`
- [React port plans](react-port-plans.md) · [Module workflow diagrams](workspace/erp-module-workflow-diagrams.md) · [Pasted design notes need verification](workspace/erp-pasted-design-notes-need-verification.md)
