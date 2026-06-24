# Configurable box order on the printed label — scan sequence vs. descending quantity

**Date:** 2026-06-24
**App:** PackCalc.html (standalone React-via-CDN, no build step)
**Status:** Approved; ready for implementation

## Context

The pieces-per-box bubbles on the printed label (`printBubble`, and the unused-but-parallel `printList`) currently always appear in scan order — box 1 scanned is `B01`, box 2 scanned is `B02`, etc. — because both templates simply `.map()` over `pallet.boxes` in array order.

Some users want the largest boxes to appear first on the printed label (e.g. to make the biggest box easy to spot when palletizing), without changing how boxes are tracked on screen during scanning.

## Decisions confirmed with user

- **Scope:** print-only. The on-screen box chips (live scan grid) and the segregation table keep showing scan order always, regardless of this setting.
- **Numbering:** when sorted descending, the label's `B01, B02…` numbers are reassigned to match the new order (the biggest box becomes `B01`). This is a purely visual/organizational renumbering — it does not preserve a way to trace back to the original scan order from the label.
- **Default:** off (scan sequence), preserving current behavior for existing users.

## Changes

### 1. New `cfg.sortLabelByQty` boolean, default `false`
Added to the `cfg` initial state alongside the other print-related flags (`qrOnLabel`, `lblDel`, etc.). No extra persistence work needed — `cfg` is already round-tripped through `localStorage` wholesale (see `Persistence` effects), so the new key is saved/restored automatically.

### 2. New checkbox in Advanced Settings → PRINT section
In the existing **PRINT** section of the Advanced Settings modal (currently holding just the "Include QR on label" checkbox), add a second checkbox:

> "Sort boxes by quantity (highest first) on label"

Same style/pattern as the existing `qrOnLabel`/`pnCheck` checkboxes (`<input type="checkbox" checked={cfg.sortLabelByQty} onChange={...}>`).

### 3. Sort applied at the two print call sites, not inside the templates
Add a small module-level helper near the print functions:

```js
const sortBoxesForLabel = (boxes, sortByQty) =>
  sortByQty ? [...boxes].sort((a, b) => b.qty - a.qty) : boxes;
```

(`Array.prototype.sort` is spec-stable, so boxes with equal `qty` keep their relative scan order.)

Use it at both call sites, building the boxes array passed into `doPrint`:
- `handlePrint`: when constructing the `pallet` object passed to `doPrint`, set `boxes: sortBoxesForLabel(boxes, cfg.sortLabelByQty)` instead of the raw `boxes`.
- `handlePrintPallet`: when constructing `ep` (which currently spreads `p`, inheriting `p.boxes`), override `boxes: sortBoxesForLabel(p.boxes, cfg.sortLabelByQty)`.

`printBubble` and `printList` are **not modified** — they keep mapping over `boxes` in array order via index `i`, which is already how renumbering happens; the sort just changes what order they receive. This respects the existing rule that print functions are module-level and only get explicit params, no `cfg`/state access.

### 4. QR payload order follows the same sort (intentional)
`printBubble`/`printList` call `encodePallet(pallet)` using the same `pallet` object they received — i.e. the already-sorted one when the toggle is on. `encodePallet`'s comment defines its `q1,q2,...,qn` list as "`B01..Bnn` order", so this is the *correct* behavior: the QR stays positionally consistent with whatever `B01, B02…` numbering is actually printed on that ticket. The TC520L app reads box N from the QR as "the box labeled B0N on this ticket" — that mapping must hold regardless of sort order, so QR and printed bubbles are sorted together, not independently.

### 5. No change to `printSummary`
The session summary only shows aggregated per-pallet totals (box count, piece count), not a per-box breakdown — nothing to reorder there.

## Out of scope
- Any change to on-screen ordering (live chips, segregation table) — confirmed print-only.
