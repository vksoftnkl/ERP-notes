---
name: erp-full-height-page-shell
description: "Full-height feature pages must use height:100% (not 100vh/100dvh) — the .erp-app-content shell slot already sizes them, and viewport units clip the footer"
metadata: 
  node_type: memory
  type: project
  originSessionId: 4a39015f-7a53-4678-8e34-b39f5f337f69
---

Feature pages render inside `.erp-app-content` (app/globals.css ~line 104), which is `flex:1; min-height:0; overflow:hidden` and already gives every direct child `height:100%` via `.erp-app-content > *`. The slot starts *below* the app header (~81px tall), so it is shorter than the viewport (e.g. 819px in a 900px window).

A page root that sets `height:100dvh; min-height:100vh` therefore overflows its own slot and its footer/pagination gets clipped out of view — the content still "exists" and Playwright reports `isVisible: true`, so this only shows up by comparing the footer's `y + height` against `innerHeight`.

**Why:** the module class and the global `.erp-app-content > *` rule have equal specificity (0,1,0), and module CSS loads later, so the page's viewport-unit height silently wins over the shell's correct `height:100%`.

**How to apply:** for a full-height page use `height: 100%; min-height: 0; overflow: hidden;` on the root. Do not copy `height:100dvh; min-height:100vh` from `features/masters/sales/state/page.module.scss` — the State Master skin carries this latent bug. Verified fixed on the audit-logs page (footer moved y=944 → y=863 in a 900px viewport).

Related: [[erp-client-ui-browser-verification]], [[erp-master-shell-modal-table-styles]]
