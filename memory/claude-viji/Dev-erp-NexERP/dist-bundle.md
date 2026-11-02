---
name: dist-bundle
description: "dist/NexERP is built from the DEBUG build (user's call 2026-10-02) by scripts/deploy-linux.sh; script now keeps appconfig.ini + certs and bundles the CUPS print plugin"
metadata:
  type: project
---

`dist/NexERP` is a portable Linux folder made by `scripts/deploy-linux.sh`, with the Qt libs, plugins and run.sh.

**User's instruction (2026-10-02): "no use debug folder".** Build it from the Debug build:
`BUILD_DIR=$PWD/build/Desktop_Qt_6_12_0_Debug bash scripts/deploy-linux.sh`.
The script's default `build/Release` doesn't exist; the Qt Creator release dir is `build/Desktop_Qt_6_10_2-Release`.

**Script fixes made 2026-10-02:**
- It used to `rm -rf` the folder, which wiped the site's `appconfig.ini` (with `CaCertFile=certs/rootCA.pem`) and `certs/`. Both are now kept across a rebuild.
- The `printsupport/libcupsprintersupport.so` plugin was added. Without it QPrinter sees no printers. Qt6PrintSupport, Pdf, PdfWidgets and libcups come in through the ldd walk.

**Verification:** run `ldd` on the binary and all plugins with LD_LIBRARY_PATH=lib and look for "not found". The bundle ships no offscreen plugin, so to start it headless point QT_QPA_PLATFORM_PLUGIN_PATH at the SDK's plugins/platforms.

`NexERP.zip` and `NexERP-x86_64.AppImage` in dist/ are from 2026-07-03 and were NOT rebuilt. The AppImage tools are in scripts/appimage-tools.
