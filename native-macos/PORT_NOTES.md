# macOS native bridge port

This directory is a from-scratch macOS port of the Windows-only native palette
bridge in `native/` (`VwxBridgePalette.cpp` + `ModuleMain.cpp`). It is a
**separate, macOS-only build** — the ported `.cpp` fully replaces the Win32
APIs with Core Foundation / POSIX equivalents, so it no longer compiles on
Windows. The original `native/` Windows project is untouched.

Result: a working native `.vwlibrary` plugin that gives Vectorworks 2026 on
macOS the same background-pump architecture the Windows build has — both
reads and writes confirmed working end-to-end against a live document,
including while Vectorworks is in the background (unfocused).

## Build requirements

- Xcode (built and tested against Xcode 26.6, macOS 26.x)
- The official Vectorworks Mac SDK matching your VW version (this port was
  built against SDK build 832364 / VW 2026)
- The `VwxBridge.xcodeproj` here expects the SDK's `SDKLib` folder to be
  reachable via a relative path from **three directories above wherever you
  clone this repo** (`../../../SDKLib` from `native-macos/VwxBridge.xcodeproj`).
  Easiest fix: symlink it as a sibling of your clone, e.g. if you clone to
  `~/projects/vwx-mcp`, run:
  ```
  ln -s /path/to/VectorworksSDK/SDKLib ~/projects/SDKLib
  ```
- Build the **Release 64** configuration, not Debug — Debug mode symlinks the
  `.vwr` resource folder instead of copying it (fine for local iteration on
  the build machine, but the symlink breaks the moment you move the built
  bundle anywhere else, e.g. to another Mac or into `/Applications`).
- When moving the built `.vwlibrary` between machines, use `ditto -c -k
  --sequesterRsrc --keepParent` (not `cp`/`scp` directly) to preserve the
  ad-hoc code signature and resource forks.

## Install

1. Build `VwxBridge.xcodeproj`, target `VwxBridge`, config `Release 64`.
2. Copy the resulting `VwxBridge.vwlibrary` into
   `/Applications/Vectorworks 2026/Plug-Ins/`.
3. Launch Vectorworks. macOS will show an "Unknown Developer Plug-ins"
   warning (ad-hoc signed, not a Vectorworks Partnership build) — enable it
   in Plug-in Manager → Developers, or via the popup dialog, then relaunch.
4. Open any document. Under **Tools → Workspaces → Edit Current Workspace...**,
   find **"VWX Bridge Palette anzeigen"** and **"VWX Pump (intern)"** under the
   "Extras" category in the Commands list, and drag "VWX Bridge Palette
   anzeigen" into a menu (these commands do **not** auto-appear in any menu —
   this is a one-time Vectorworks requirement, same as on Windows).
5. **One additional one-time step, Python menu command** (this replaces the
   Windows keystroke-based mutation trigger — see "Design change" below):
   Plug-in Manager → Custom Plug-ins → New... → **Command** → name it exactly
   `VWX Bridge Start` → Script Editor → set Language to **Python** → paste:

   ```python
   import os, sys, importlib

   _base = os.path.join(os.path.expanduser('~'), 'Library', 'Application Support',
                        'Vectorworks', '2026', 'Plug-ins')
   for _name in ('VW-MCP', 'VWX-MCP'):
       _dir = os.path.join(_base, _name)
       if os.path.isdir(_dir):
           if _dir not in sys.path:
               sys.path.insert(0, _dir)
           break

   import vwx_pump
   importlib.reload(vwx_pump)
   vwx_pump.pump_all()
   ```
6. Click the "VWX Bridge Palette anzeigen" menu item to open the palette —
   Status should read "active".

## Bugs found and fixed during the port

1. **Resource-folder name must match `DefaultPluginVWRIdentifier()`.**
   `VWFC::PluginSupport::GetStandardURL()` builds its resource lookup path as
   `DefaultPluginVWRIdentifier() + "/" + htmlFolderName` and enumerates VW
   resources under that name. If the product/bundle name (and hence the
   built `.vwr` folder name) doesn't match the string
   `DefaultPluginVWRIdentifier()` returns (`"VwxBridge"`), the lookup finds
   nothing and the SDK dereferences an uninitialized file identifier — an
   instant `EXC_BAD_ACCESS` on palette open. Keep the Xcode target/product
   name equal to `DefaultPluginVWRIdentifier()`'s return value.
2. **`.vwstrings` files must be UTF-16LE with BOM**, not UTF-8. This SDK's
   `BuildVWR` tool rejects UTF-8 (even with a BOM) with a parse error
   ("Found unexpected character when expecting open quote for string key").
3. **`CountJobs()`'s `opendir()` call had a leftover Windows glob pattern**
   (`opendir(".../ipc/jobs/*.json")`) — `opendir()` doesn't do glob
   expansion, so it silently returned 0 jobs always, even with files
   present. Fixed to open the plain directory; the `.json` suffix filter
   already existed correctly in the `readdir()` loop below it.
4. **The debounce timer overflowed a `double`→`uint32_t` conversion.**
   `CFAbsoluteTimeGetCurrent() * 1000.0` in the mid-2020s is on the order of
   7.9×10^11 (milliseconds since the 2001 epoch), which vastly exceeds
   `UINT32_MAX` (~4.3×10^9). Casting an out-of-range double to an unsigned
   integer type is undefined behavior in C++; in practice it produced a
   ~constant saturated value on this hardware, permanently jamming the
   120ms debounce check after the very first successful pump (every job
   after the first silently never fired). Fixed by keeping the timer in
   `double` throughout — no cast, no overflow, plenty of precision.

## Design change: mutation trigger

The Windows build fires writes by simulating a `Ctrl+Shift+B` keystroke
(`keybd_event`/`PostMessage`) to trigger the `'VWX Bridge Start'` menu
command, with separate foreground/background code paths since a synthetic
global keystroke only reaches the frontmost app.

macOS has no direct equivalent, so instead of simulating input at all, the
native timer calls the SDK's own `gSDK->DoMenuName("VWX Bridge Start", 0)`
directly. This is an in-process call into Vectorworks' own command
dispatcher rather than a synthetic OS-level event, so — usefully — **it
works regardless of whether Vectorworks is the focused/frontmost
application.** Confirmed live: triggered a document write via MCP while
Finder was frontmost and Vectorworks was fully backgrounded; it fired
correctly. This means the "background-wake-while-unfocused" problem the
Windows keystroke dance solves doesn't need solving separately on macOS —
`DoMenuName` sidesteps it entirely.

## Known limitation / TODO

**Auto-dismiss-error-dialogs is stubbed, not implemented**, on macOS. On
Windows this walks other windows (`EnumWindows`) looking for a dialog-class
window and clicks its OK button, to keep unattended background operations
from getting stuck behind a VW error/message dialog. There's no direct
macOS equivalent without the Accessibility API (`AXUIElement`), which
requires granting Vectorworks Accessibility permission in System Settings —
a real, broad, persistent permission grant, not something to add casually.

Deliberately left as a stub for now: `DismissErrorDialogs()` logs
`"auto-dismiss-error-dialogs / menu-command-ID lookup not implemented on
macOS (v1)"` and returns. Failure mode if this is ever needed is benign — a
stuck job just times out client-side after ~55s; nothing corrupts or hangs
Vectorworks itself. Revisit if it turns out to matter in practice.
