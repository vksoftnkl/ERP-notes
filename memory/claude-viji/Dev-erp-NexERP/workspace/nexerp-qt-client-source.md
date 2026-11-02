---
name: nexerp-qt-client-source
description: Qt/C++ desktop client source lives at /home/vk/VKNexERP/NexERP (GitHub vksoftnkl/NexERP, branch main)
metadata:
  type: reference
---

Qt 6 / CMake desktop client for the ERP server. Cloned 2026-10-02 to /home/vk/VKNexERP/NexERP,
origin https://github.com/vksoftnkl/NexERP.git, branch `main`; HEAD then was 09f8896 "1-10-2026"
(commits are date-named by the user, Author "Viji").
Layout: src/{app,core,modules,shared}, tests/<module> (one dir per form), docs/ (sale bill gap notes),
plan.md (item_entry.ui payload/label plan), original.sql. `react` at the root is a 0-byte FILE, not the
React client (that's `/home/vk/Dev/erp/ERP client`). ~/NexERP is only a compiled build.
API routes the client calls: src/core/api/api_endpoints.h.
Related: [[erp-app-theme-state]], [[erp-sale-bill-react-port]].
