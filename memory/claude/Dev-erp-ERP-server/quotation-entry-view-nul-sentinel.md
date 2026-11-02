---
name: quotation-entry-view-nul-sentinel
description: "ERP client quotation-entry-view.tsx holds a literal NUL byte, so git calls it binary and text-tool edits can silently drop it"
metadata: 
  node_type: memory
  type: project
  originSessionId: b5bd0dbd-6a83-4250-ae95-74215812ebf8
  modified: 2026-08-08T06:31:23.821Z
---

`ERP client/features/sales/quotation/components/quotation-entry-view.tsx` contains one literal NUL byte: `const NEW_DOCUMENT = "\0new"` — a deliberate sentinel for "form showing a quotation with no id yet", chosen because it cannot collide with a real `sqId`. `file` reports the source as `data` and `git diff` shows it as `Bin 29081 -> 29095 bytes` instead of a patch.

**Why:** the NUL is load-bearing, but grep skips the file by default (needs `-a`), diffs are unreadable, and a whole-file rewrite through a text-oriented editor can drop or mangle the byte — turning the sentinel into the string `"new"`, which *can* collide.

**How to apply:** edit it byte-wise (e.g. a small `python3` script doing an asserted single-occurrence `bytes.replace` and writing back in `"wb"`), then confirm `count(b"\x00") == 1` afterwards. Verified 2026-08-08 while adding `itemSize` to `TEXT_LINE_FIELDS` — see [[ui-table-columns-drive-grids]].
