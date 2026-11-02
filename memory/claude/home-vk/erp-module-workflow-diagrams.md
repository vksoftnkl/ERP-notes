---
name: erp-module-workflow-diagrams
description: "\"Workflow image / workflows functionalities\" for a server module = two PNGs in <module>/diagrams/; how they are drawn and rendered"
metadata:
  node_type: memory
  type: reference
  originSessionId: f985be56-5956-450c-95d8-e4630788c940
  modified: 2026-10-12T07:40:58.631Z
---

A request like "create the workflow image and workflows functionalities" for an ERP server module means the pair of PNGs in `<module>/diagrams/`:
`<module>-workflow.png` ("— functionality-wise workflow": request pipeline, scope checks + refusals, the SQL engine, each route's steps, checks, payload) and `<module>-tables.png` ("— table-wise data model": table cards with key columns, FK lines, a route × table matrix). Examples: reports/party-outstanding, settings/employeeMaster, reports/day-summary (2026-10-12).

Only the PNGs go in the repo. The party-outstanding / employee-master HTML sources were not kept. The day-summary sources are in `~/.cache/erp-diagrams/day-summary/`: common.css, wires.js (orthogonal connectors drawn after the fonts load), workflow.html, tables.html, render.sh. Reuse them as the template for the next module.

Rendering: `render.sh <out-dir> workflow tables` serves the folder over a local http.server, runs snap Chromium headless (2000 CSS px wide × 2.4 scale = 4800 px, Inter + JetBrains Mono from Google Fonts), then trims the bottom with PIL. Snap Chromium cannot open file:// in hidden dirs or write to /tmp, so the script shoots into ~/snap/chromium/common and moves the file.

Related: [[erp-party-outstanding-backend]]
