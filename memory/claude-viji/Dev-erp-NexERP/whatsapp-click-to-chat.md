---
name: whatsapp-click-to-chat
description: Sale Bill WhatsApp button BUILT 10-27 — wa.me chat + bill PDF on clipboard (Ctrl+V); Cloud API is the later automatic option
metadata:
  node_type: memory
  type: project
  originSessionId: c376796f-ad85-4411-a8bc-3642f66289b0
  modified: 2026-10-27T13:54:10.337Z
---

Built 2026-10-27 on the user's go, uncommitted. Sale Bill btnQuickWhatsapp (enabled once m_docRefno is set):
number = ledger ledWhatsappNo → ledPhone1/2 → bill phone (walk-in: bill phone) → ask; normalised to 91XXXXXXXXXX.
Renders the SALE_INVOICE PDF with no dialog (`PrintingModule::fetchPdf`, same ladder as Preview), saves to
temp/NexERP-whatsapp, puts the FILE on the clipboard, tells the operator "Ctrl+V in the chat", opens https://wa.me/<no>?text=.
Helper: src/shared/utils/nex_whatsapp.* (normalisePhone, openChat, copyFileForPaste). Test: tests/whatsapp/run.sh (12/12 live).

**Why:** wa.me cannot carry a file; truly automatic sending needs the WhatsApp Business Cloud API (Meta account,
templates, per-message charge) on the NestJS server — Prathap's side, not started. Download-link option blocked
because the server is LAN-only (192.168.0.106).

**How to apply:** Sale Order / Quotation / Receipt can reuse fetchPdf + NexWhatsApp the same way (Sale Order already has
the button). Clipboard paste is reliable in WhatsApp Desktop on Windows; on Linux/Web the drag-from-path fallback is shown.
