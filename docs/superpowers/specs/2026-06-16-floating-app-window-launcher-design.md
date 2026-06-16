# Floating app-window launcher for PackCalc

**Date:** 2026-06-16
**App:** PackCalc.html (standalone React-via-CDN, no build step)
**Status:** Approved; trivial scope, implemented directly (no separate plan doc)

## Context

PackCalc is opened today by double-clicking `PackCalc.html`, which loads as a normal tab in the default browser (Edge). The user wants it to open the way Windows Calculator does: in its own window, no browser tabs/address bar, starting small like Calculator's default window, but still resizable/movable by hand.

PackCalc's layout is desktop-only and fixed for 1920×1080 (two-panel, non-responsive — see project CLAUDE.md). Reflowing it to be usable at a small window size was explicitly ruled out as out of scope; the small starting size is cosmetic (matches Calculator's launch feel), and the user will resize/maximize the window before doing real work.

## Decision

No changes to `PackCalc.html` or any app code. Add a Windows shortcut on the Desktop that launches Edge in **app mode**:

```
Target:    C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe
Arguments: --app="file:///C:/Users/aspiezia/Downloads/scan%20calculator%200.1/PackCalc.html" --window-size=420,640
```

`--app=` strips Edge's tabs/toolbar/address bar so the page renders in its own chromeless, resizable window. `--window-size` sets the initial size only (420×640, Calculator-like); the user can resize/move/maximize it afterward like any window.

## Out of scope

- Always-on-top behavior (not achievable from a web page; would require an Electron/Tauri wrapper — rejected to avoid adding a build step to this project).
- Responsive/compact layout for small window sizes.
- Taskbar pinning or custom icon (left as a manual follow-up if wanted; default Edge icon is used).
