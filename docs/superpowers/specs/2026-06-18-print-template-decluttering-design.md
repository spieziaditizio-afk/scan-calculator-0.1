# Print template decluttering — remove brand/RECEIVING, optional QR, bigger bubbles

**Date:** 2026-06-18
**App:** PackCalc.html (standalone React-via-CDN, no build step)
**Status:** Approved; ready for implementation

## Context

The printed pallet label (`printBubble`, the live layout) currently spends header space on a "PACKCALC · SEVENUM" brand line, a "RECEIVING" tag chip, and a QR code — leaving less room for the per-box piece-count cards (the actual operationally useful content). The QR is also always-on today, even for users who don't use the external TC520L picking app it's meant for.

Goal: reclaim header space for the piece-count cards, and make the QR opt-in.

## Changes

### 1. Remove "PACKCALC · SEVENUM" brand line — all three templates
Remove the `.brand` div from `printBubble`, `printList`, and `printSummary`. This orphans:
- The shared `.brand{...}` CSS rule in `labelCSS` (used by bubble/list) — remove it.
- The `p.brand` size key in both `labelCSS` size configs (isA4 / thermal) — remove it.
- The separate `.brand{...}` CSS rule inside `printSummary`'s own `<style>` block — remove it.

`printSummary`'s header keeps `RECEIVING SUMMARY` + date; `printBubble`/`printList`'s `.hdr-left` keeps just the pallet id (`.pid`).

### 2. Remove "RECEIVING" tag — bubble/list templates
Remove the `<div class="tag">RECEIVING</div>` from `printBubble` and `printList` (its only two usages). This orphans, in `labelCSS`:
- `.tag{...}` CSS rule
- `p.tag` / `p.tagSz` size keys (both variants)
- `k.tagBg` / `k.tagFg` / `k.tagBd` color keys (both variants)

All removed since nothing else references them.

### 3. QR becomes opt-in via Advanced Settings
- New `cfg.qrOnLabel` boolean, default **`false`**.
- New **PRINT** section in the Advanced Settings modal (between printer choice and the existing VERIFICATION section), with one checkbox: "Include QR on label".
- Threaded through the existing `lbl` object (already passed `handlePrint`/`handlePrintPallet` → `doPrint` → `printBubble`/`printList`): add `qr:cfg.qrOnLabel` alongside the existing `del/pn/coo` keys at both call sites.
- In `printBubble`/`printList`, gate QR generation: `const qrUrl = lbl.qr ? await genQR(encodePallet(pallet)) : null;` (currently unconditional).
- `printSummary` has no QR; unaffected.

### 4. Enlarge `bubbleSize` tiers to use the freed space
Bump both isA4 and thermal tiers ~15% (width/height/digit font size); per-box label font (`fs`) nudged up where it doesn't crowd the card:

| isA4 tier (count) | before w×h / fn / fs | after |
|---|---|---|
| ≤6  | 60×76 / 24 / 9 | 70×88 / 28 / 10 |
| ≤12 | 48×60 / 20 / 8 | 56×70 / 23 / 9 |
| ≤20 | 38×48 / 16 / 7 | 44×56 / 19 / 8 |
| else | 30×38 / 13 / 6 | 35×44 / 15 / 7 |

| thermal tier (count) | before w×h / fn / fs | after |
|---|---|---|
| ≤4  | 58×68 / 24 / 8 | 66×78 / 27 / 9 |
| ≤10 | 46×56 / 20 / 8 | 54×64 / 23 / 9 |
| ≤20 | 36×44 / 16 / 7 | 42×52 / 19 / 8 |
| ≤35 | 28×34 / 13 / 6 | 32×40 / 15 / 7 |
| else | 22×28 / 11 / 6 | 26×32 / 13 / 6 |

`bubbleFontSize`'s auto-shrink-to-fit logic is unaffected (still derives from the new `sz.fn`/`sz.w`).

## Out of scope
- Any change to the QR payload format (`encodePallet`) — still versioned `PC1|...` for the TC520L bridge, just gated on/off.
- `printList`'s general layout (it remains the unused alternate layout; only brand/tag/QR removal applied for consistency, per the existing "update all three together" convention).
