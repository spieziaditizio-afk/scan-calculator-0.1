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

**Architecture**: `PackCalc.html` is the only file — standalone React 18 app via CDN + Babel standalone. No npm, no build step. All JS, CSS, and JSX in one file.

**Verify**: Open in a browser. No automated tests or build step exists.

**Component pattern**: Defined at module level as `const X = ({props}) => {...}`. Color object `const C = {...}` is at module level (~line 125), alongside CSS `--var` tokens — keep both in sync when changing colors.

**Font stack**: Chakra Petch (all UI) + JetBrains Mono (codes/data only). Bebas Neue and Barlow Condensed are removed — do not re-add.

**Print templates**: `printSummary`, `printList`, `printBubble` use `#ffb700` intentionally (paper docs). Don't apply app UI colors there.

**Layout**: Desktop-only (1920×1080). Two-panel: sidebar (left, `panel-right` CSS class, `order:1`) + scan area (right, `panel-left` CSS class, `order:2`). CSS `order` is swapped from class names — `panel-right` is visually LEFT. `view` state was removed; do not re-add tab switching.

**Pallet naming**: Display as `PLT ${idx+1}` (index-based). Reuses indices when pallets are deleted — intentional.

**localStorage key**: `packcalc_history` — rolling 20 sessions, saved on NEW SESSION click.

**Edit safety**: Before `replace_all` on large blocks, read the current section first — prior edits may have changed `old_string`.

**Duplicate strings**: Some strings appear twice in the file (e.g. `statusBadge`). Always read surrounding context to pick the right instance — `replace_all:false` will error if there are two matches.

**Wrapper divs in JSX**: When wrapping existing JSX in a new `<div>`, add the closing tag in the same edit to avoid JSX parse errors that break the entire app.

**Data flow — print**: `handlePrint` reads from current pallet `meta` + `packingList`. `handlePrintPallet` reads from stored pallet `p.meta`. Any new field (e.g. `worker`) must be added to: `initPallet()`, session restore, `packingList` state, `handlePrint`, AND `handlePrintPallet`.

**packingList state**: `{ po:'', del:[], pn:'', coo:[], qty:'', worker:'' }` — `del` and `coo` are arrays (multi-value tag fields), the rest are strings. UI form only shows `qty` and `worker` — the other fields remain in state for per-box validation. Do not re-add them to the packing list form.

**Worker field**: Rendered above the packing list card, outside it.

**Segregation table**: Below the box chips in the left panel. Groups current pallet's boxes by `b.qty` (pieces/box). Uses IIFE pattern in JSX: `{boxes.length>0&&(()=>{ ... })()}`.

**summaryLine**: Still defined as `useMemo` but no longer rendered anywhere — orphaned dead code. Do not re-add its display.

**Flex button clipping**: Buttons inside flex rows that get cut off when the panel narrows need `flexShrink:0, whiteSpace:'nowrap'`.

**Print template branding**: All three templates (`printBubble`, `printList`, `printSummary`) now use `PACKCALC · SEVENUM`. Update all three together if rebranding again.
