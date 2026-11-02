---
name: commit-message-plain-text
description: "When the user asks for a commit message, give ONLY the message text — they paste it into the VS Code commit box, not a terminal"
metadata:
  node_type: memory
  type: feedback
  originSessionId: 4a0fbcf8-659a-4a8b-9dd6-0c14a6da3def
  modified: 2026-10-16T13:34:02.529Z
---

Give a commit message as plain text in one code block: subject, blank line, body. No `git commit` command, no heredoc, no `-F`/`-m`, and no splitting the staged set into several commits by path.

**Why:** 2026-10-16 the user pasted a `git commit -F - -- <path> <<'EOF' …` block into the VS Code Source Control message box. The whole command became the subject of cccceaf2, and the box committed everything staged (3,384 dist files plus the spec) in one commit, which was then pushed.

**How to apply:** one message describing everything staged. If something staged shouldn't go in (logs/error.log, an accidental dist deletion), say so in a line before the message, and tell them to unstage it in the Source Control panel. Check `git status` first: sibling sessions commit and stage between asks ([[concurrent-claude-sessions]]).
