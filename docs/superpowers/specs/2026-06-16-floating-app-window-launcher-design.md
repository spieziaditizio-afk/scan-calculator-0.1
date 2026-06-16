# Floating app-window launcher for PackCalc

**Date:** 2026-06-16
**App:** PackCalc.html (standalone React-via-CDN, no build step)
**Status:** Approved; trivial scope, implemented directly (no separate plan doc)

## Context

PackCalc is opened today by double-clicking `PackCalc.html`, which loads as a normal tab in the default browser (Edge). The user wants it to open the way Windows Calculator does: in its own window, no browser tabs/address bar — but, since PackCalc's layout is desktop-only and fixed for 1920×1080 (two-panel, non-responsive — see project CLAUDE.md), the window should start large/maximized rather than small, to avoid the fixed layout looking broken/cut off at launch. The window remains a normal, resizable/movable window after that.

## Decision

No changes to `PackCalc.html` or any app code. Add a Windows shortcut on the Desktop that launches Edge in **app mode**, maximized:

```
Target:    C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe
Arguments: --app="file:///C:/Users/aspiezia/Downloads/scan%20calculator%200.1/PackCalc.html" --start-maximized
```

`--app=` strips Edge's tabs/toolbar/address bar so the page renders in its own chromeless, resizable window. `--start-maximized` makes it open full-screen-like on launch; the user can un-maximize, move, or resize it afterward like any window.

## Out of scope

- Always-on-top behavior (not achievable from a web page; would require an Electron/Tauri wrapper — rejected to avoid adding a build step to this project).
- Responsive/compact layout for small window sizes.
- Taskbar pinning or custom icon (left as a manual follow-up if wanted; default Edge icon is used).
