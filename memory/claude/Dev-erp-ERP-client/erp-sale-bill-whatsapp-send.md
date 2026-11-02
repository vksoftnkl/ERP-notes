---
name: erp-sale-bill-whatsapp-send
description: Sale bill WhatsApp send (2026-10-28) — click-to-chat + a public expiring PDF LINK in the message (server /print-share, table print_share_link); user rejected the Business API twice; fallback = download & attach
metadata:
  type: project
---

Sale bill strip icon + bill-list toolbar button open
`features/sales/salebill/components/whatsapp-bill-dialog.tsx`.

**User decisions (2026-10-28):** click-to-chat, NOT the WhatsApp Cloud API / a BSP.
First version only typed the message (operator attached the downloaded PDF by
hand); the user then wanted the PDF "in the chat" and chose **a link to the PDF
in the message** over Cloud API / share sheet / paste-as-image. A wa.me /
web.whatsapp.com link can carry TEXT only — no page can attach a file through it.

**Why:** no Meta account or per-message cost. Truly automatic attachment would be
a new Cloud API integration (template with DOCUMENT header).

**How to apply:**
- Server: `POST /print-share/create` (auth) freezes {version, doc, company,
  branch, acc year, device, user} under a 22-char random token in
  `public.print_share_link` (migration 20261028170000, sync column + guarded sync
  triggers, no FKs); public `GET /print-share/<token>[/<file>.pdf]` re-renders a
  PDF on every open (404/410 HTML pages for unknown/expired). Env
  `PRINT_SHARE_TTL_DAYS` (default 30), `PRINT_SHARE_PUBLIC_BASE_URL` (set it on an
  OFFLINE site so links point at the cloud, where the synced row is served).
  `/print-share/create` is in the cache-invalidation READ_ONLY list (the dialog
  creates a link on every open).
- Why a DB table, not Redis: production (CloudJiffy) runs `REDIS_ENABLED=false`,
  and `RedisCacheService.set` silently no-ops when disabled.
- Client: `useDocumentPdf` renders (WHATSAPP → PDF → PRINT ladder) and exposes
  `source`; the dialog then calls `features/printing/api/share.ts`; link URL =
  server `url` else API_BASE + path. If create fails → message says "attached",
  Send downloads the PDF (old flow).
- In local dev the link is `https://192.168.0.106:3011/...` — a LAN address a
  customer's phone cannot open; only a public deployment makes links work.
- Dev data: no customer has `cus_whatsapp_no`; only the phone fallback runs.
- Related: [[erp-rtk-pending-query-dedupe-abort]], [[erp-throwaway-server-instance]],
  [[erp-prisma-schema-hand-edit-trap]].
