# CLAUDE.md

Behavioral guidelines to reduce common LLM coding mistakes. Merge with project-specific instructions as needed.

**Tradeoff:** These guidelines bias toward caution over speed. For trivial tasks, use judgment.

## 1. Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

Before implementing:
- State your assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them - don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

## 2. Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

Ask yourself: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

## 3. Surgical Changes

**Touch only what you must. Clean up only your own mess.**

When editing existing code:
- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it - don't delete it.

When your changes create orphans:
- Remove imports/variables/functions that YOUR changes made unused.
- Don't remove pre-existing dead code unless asked.

The test: Every changed line should trace directly to the user's request.

## 4. Goal-Driven Execution

**Define success criteria. Loop until verified.**

Transform tasks into verifiable goals:
- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"
- "Refactor X" → "Ensure tests pass before and after"

For multi-step tasks, state a brief plan:
```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
3. [Step] → verify: [check]
```

Strong success criteria let you loop independently. Weak criteria ("make it work") require constant clarification.

---

**These guidelines are working if:** fewer unnecessary changes in diffs, fewer rewrites due to overcomplication, and clarifying questions come before implementation rather than after mistakes.

---

## Project: PackCalc

**Architecture**: `PackCalc.html` is the app — standalone React 18 via CDN + Babel standalone. No npm, no build step. All JS, CSS, and JSX in one file. `ReceivingBento.html` is a separate static design mockup (light-WMS bento grid) — not part of the app; same for `ZT620LabelMockup.html` / `ThermalBubbleMockup.html` (label previews). Specs & plans live in `docs/superpowers/`.

**Verify**: Open in a browser. No automated tests or build step exists.

**Domain / scope**: PackCalc = receiving station (count boxes → print pallet label + QR on Zebra ZT620). Picking, optimal box combination, and label re-printing are a SEPARATE app on a Zebra TC520L handheld (+ QLn420 mobile printer) — NOT PackCalc. SPQ-aware partial-box marking is a parked future feature.

**Component pattern**: Defined at module level as `const X = ({props}) => {...}`. Color object `const C = {...}` now maps each key to a CSS var (`bg:'var(--bg)'`, etc.), so the CSS `--var` tokens in `:root` are the single source of truth. To change colors, edit `:root` (light) / `:root[data-theme="dark"]` (dark) only — NOT `C`.

**Theming / dark mode**: Default is the light WMS palette (slate primary `#334155` + green accent `#059669`, cool-gray bg, white cards); dark mode lives in `:root[data-theme="dark"]`. The header DARK/LIGHT toggle sets `data-theme` on `document.documentElement` and persists to `localStorage` `packcalc_theme`. The navy header gradient is hardcoded (identical in both themes). The `Svg` component applies `stroke` via `style={{stroke:color}}` (NOT the `stroke=` attribute) so `var()` colors resolve under theming — don't revert it.

**Font stack**: Fira Sans (all UI) + JetBrains Mono (codes/data only), via Google Fonts; print templates use `Inter`. (Was Chakra Petch + Bebas/Barlow — all removed.)

**Print templates** (`printBubble`/`printList`/`printSummary`): A4 (`isA4`) = color — amber `#ffb700`, green/red badges. Thermal (ZT only, `!isA4`) = monochrome via the `k` palette object in `labelCSS` (branch on `!isA4`). PackCalc prints to **Zebra ZT + A4 only** (QLn was removed; the external TC520L app handles QLn420 reprints). Respect this A4-vs-thermal split on any print-color edit; never apply app UI colors here.

**Layout**: Desktop-only (1920×1080). Two-panel: sidebar (left, `panel-right` CSS class, `order:1`) + scan area (right, `panel-left` CSS class, `order:2`). CSS `order` is swapped from class names — `panel-right` is visually LEFT. `view` state was removed; do not re-add tab switching.

**Pallet naming**: Display as `PLT ${idx+1}` (index-based). Reuses indices when pallets are deleted — intentional.

**localStorage keys**: `packcalc_history` (rolling 20 sessions, saved on NEW SESSION click); `packcalc` (config/state restore); `packcalc_theme` (`light`/`dark`).

**Edit safety**: Before `replace_all` on large blocks, read the current section first — prior edits may have changed `old_string`.

**Duplicate strings**: Some strings appear twice in the file (e.g. `statusBadge`). Always read surrounding context to pick the right instance — `replace_all:false` will error if there are two matches.

**Wrapper divs in JSX**: When wrapping existing JSX in a new `<div>`, add the closing tag in the same edit to avoid JSX parse errors that break the entire app.

**Data flow — print**: `handlePrint` reads from current pallet `meta` + `packingList`. `handlePrintPallet` reads from stored pallet `p.meta`. Any new field (e.g. `worker`) must be added to: `initPallet()`, session restore, `packingList` state, `handlePrint`, AND `handlePrintPallet`.

**Print functions are module-level**: `printBubble/printList/printSummary/labelCSS/encodePallet/doPrint` have NO access to `cfg`/React state — pass what they need as params (e.g. the `lbl={del,pn,coo}` object threaded `handlePrint`→`doPrint`→`printBubble/printList`). Vars scoped inside `labelCSS` (e.g. `k`) can't be referenced from the body template strings.

**Print-time box transforms**: Sort or reshape `boxes` for print only by building a modified pallet before `doPrint`: `{...pallet, boxes: transformed}` in `handlePrint`, or `{...ep, boxes: transformed}` in `handlePrintPallet`. `printBubble`, `printList`, and `encodePallet` (QR) all read `pallet.boxes` from that same object — one transform keeps them in sync. Use a module-level pure helper placed before `doPrint`. Example: `cfg.sortLabelByQty` (bool, default `false`) — Advanced Settings → PRINT — sorts printed bubbles descending by qty; B0x renumbers to match; QR follows automatically.

**QR bridge**: `encodePallet` returns compact versioned `PC1|palletId|PN|PO|DEL|COO|q1,q2,…` (pieces/box) for the external TC520L app; rendered on every label (top-right). Generated via **`qrcode-generator`** (global `qrcode`: `qrcode(0,'M').addData(s).make().createDataURL(3)`) — NOT node-qrcode's `QRCode.toDataURL`, which has no browser build (that nonexistent CDN path was the original latent bug: QR silently never rendered). Keep it versioned/parseable; QR ink is black so it prints on thermal.

**Verification / scanning**: Per-box scanning is QTY-only — the inline VERIFY toggles and the Pallet Identity card were both removed. PN matching is opt-in: Advanced Settings → VERIFICATION → `cfg.pnCheck` reveals a transient **PN CHECK** panel (scan box PN barcodes → ✓ match / ✗ mismatch; state `pnScans`/`pnInput`; no per-box record). Its "use this PN on label" button sets `meta.pn`. The old per-box mismatch `useMemo` (`cfg.po/del/pn/coo`, `meta.pn`) remains but is inert.

**Bubble label (`printBubble`, the live layout)**: shows ONLY the PIECES PER BOX bubble cards (B01, B02, …) — no PO/DELIVERY/PN/COO/WORKER text, since that's already on the WMS-printed label. Those fields stay in the QR (`encodePallet`) for the TC520L bridge — only the visible printed text was stripped. `bubbleSize(count,isA4)` tiers were enlarged to use the freed space; re-tune both isA4 and thermal tiers together if adjusting.

**Label-field toggles — removed**: `cfg.lblDel/lblPn/lblCoo` (default true) still exist in `cfg` and are still threaded through `handlePrint`/`handlePrintPallet` → `doPrint` → `printBubble`/`printList` as the `lbl` param, but the **Advanced Settings "Show on label" checkboxes were deleted** (no UI sets them — they're permanently `true`). `printBubble` no longer reads `lbl` at all (dead param, kept for signature compatibility). `printList` (an unused alternate layout) still gates its meta block on `lbl.del/lbl.pn/lbl.coo`. Don't re-add the checkboxes unless asked.

**Advanced Settings modal** (`printerOpen` state, unchanged name): printer choice + PRINT subsection (`cfg.qrOnLabel` — QR on label; `cfg.sortLabelByQty` — sort boxes by qty descending on label) + VERIFICATION (`cfg.pnCheck`). Originally titled "Print Settings" with its gear-icon trigger in the scan-area config row — both were moved: modal retitled "ADVANCED SETTINGS", trigger button now lives in the **app header** (next to DARK/GUIDE/HISTORY/NEW SESSION). `printerOpen`/`historyOpen`/`guideOpen` booleans are unchanged.

**Header button styling**: `.app-header .new-btn` CSS overrides color to `#cbd5e1` (light gray) — distinct from `C.amber` used elsewhere in the scan area. Any button moved into the header (e.g. the gear/Advanced-Settings trigger) must use `#cbd5e1`/inherit, not `C.amber`, to match DARK/GUIDE/HISTORY/NEW SESSION.

**Barcode scanner input (Zebra DS etc.)**: keyboard-wedge scanners may send `Tab` as the terminator instead of `Enter`. The main scan `<input>`'s `onKeyDown` must treat `'Tab'` identically to `'Enter'` (call `handleEnter` + `preventDefault`) or digits land in the field but never commit, and default Tab focus-traversal steals focus to the next element. A `focusout` safety-net `useEffect` (deps: `printerOpen,historyOpen,guideOpen`) reclaims focus on the scan input whenever it lands on a non-editable element, unless a modal is open.

**packingList state**: `{ po:'', del:[], pn:'', coo:[], qty:'', worker:'' }` — `del` and `coo` are arrays (multi-value tag fields), the rest are strings. UI form only shows `qty` and `worker` — the other fields remain in state for per-box validation. Do not re-add them to the packing list form.

**Worker field**: Rendered above the packing list card, outside it.

**Segregation table**: Below the box chips in the left panel. Groups current pallet's boxes by `b.qty` (pieces/box). Uses IIFE pattern in JSX: `{boxes.length>0&&(()=>{ ... })()}`.

**summaryLine**: Still defined as `useMemo` but no longer rendered anywhere — orphaned dead code. Do not re-add its display.

**Flex button clipping**: Buttons inside flex rows that get cut off when the panel narrows need `flexShrink:0, whiteSpace:'nowrap'`.

**Print template branding**: All three templates (`printBubble`, `printList`, `printSummary`) now use `PACKCALC · SEVENUM`. Update all three together if rebranding again.
