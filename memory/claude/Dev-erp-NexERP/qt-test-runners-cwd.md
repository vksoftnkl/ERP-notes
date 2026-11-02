---
name: qt-test-runners-cwd
description: tests/*/run.sh-style g++ links need cwd = repo root — some CMake include dirs are relative
metadata:
  node_type: memory
  type: reference
  originSessionId: d2a416c9-c420-4888-a853-d6940ff083ec
  modified: 2026-10-30T08:34:24.776Z
---

The `tests/<area>/run.sh` runners (and any probe built the same way) parse `target_include_directories(NexERP PRIVATE …)` out of CMakeLists.txt. Some entries are bare relative paths (e.g. `src/modules/accounts/expense`), not `${CMAKE_CURRENT_SOURCE_DIR}/…`, so `-I` only resolves when the shell is in `/home/vk/Dev/erp/NexERP`. Run from elsewhere → "fatal error: expense_voucher_entry.h: No such file".

`tests/printing/run.sh` does NOT just run unit tests: blocks OUT3 (document print), OUT4 (assignments: creates and deletes a row), OUT5, OUT7 and OUTRT (writes print_log and print_share rows) hit the dev server. Its first lines call `/home/vk/Qt/Tools/Ninja/ninja` (1.12) while the build dir is ninja 1.13 (`~/.local/bin`); on a fresh build dir that forces a full rebuild and failed, so use `cmake --build` instead. GUI tests piped through `| grep` hide a hang or non-zero exit; check that the final "ALL PASSED" line is printed.

Dev-server test login is VKPOS/VKPOS; company 019c8ea6-19e9-78a8-b15f-749e1cde7292, branch 019c8ea7-b0f5-72d5-96a5-1abfc80cc8ab, year 2026-2027. Grid run calls cap `limit` at 100 and reject `""` for uuid params.
