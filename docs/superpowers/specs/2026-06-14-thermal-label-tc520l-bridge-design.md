# Thermal label + TC520L bridge — PackCalc design

**Date:** 2026-06-14
**App:** PackCalc.html (standalone React-via-CDN, no build step)
**Status:** Approved (design); ready for implementation plan

## Context

Incoming pallets are received and counted box-by-box. Pallets arrive **mixed**: some boxes hold a full pack (SPQ), others hold fewer pieces (partials). The WMS only stores the pallet's total pieces + SPQ, so it loses the per-box detail. Later, when an order needs a quantity that isn't a multiple of SPQ, a VNA combi-truck operator must find the optimal combination of boxes across one or more pallets (exact or just under, never over).

PackCalc is the **receiving station**: it scans/counts pieces per box and prints a pallet label. A **separate app on the Zebra TC520L** handheld does the picking/combination later and reprints updated labels on a **Zebra QLn420** mobile printer.

This design covers only PackCalc's part: a thermal-printable label + a machine-readable bridge for the TC520L app.

## Goals

1. Print the existing "bubble" / "list" label correctly on warehouse thermal printers (Zebra ZT620, QLn420) — **monochrome** (black on white).
2. Let the worker choose whether **Delivery, Part Number, COO** print on each pallet label (default ON; order DEL → PN → COO).
3. Emit a compact **QR** on the label encoding the **box quantities** so the TC520L app can ingest them.

## Non-goals (out of scope)

- Combination optimization — TC520L app.
- Scanning / reloading / updating labels inside PackCalc — done by TC520L + QLn420.
- SPQ capture and explicit partial-box marking — separate, parked feature.
- A4 output — stays in the current color (amber) layout, unchanged.

## Design

### 1. Monochrome thermal rendering

When the target printer is thermal (`ZT` / `QLn`, i.e. `!isA4`), render the label **black on white**. A4 keeps its current amber/color palette.

Thermal palette mapping:
- **Bubbles:** `2px solid #000` border, white fill, qty `#000`, `B##` `#000` (today: amber border + `#fffbe6` fill).
- **RECEIVING tag:** white bg, `#000` text, `#000` border (outlined) — today amber fill.
- **VERIFIED:** solid `#000` bg, white text. **MISMATCH:** white bg, `#000` text, `2px #000` border. Distinguished by the ✓ / ⚠ icon + text, not color.
- **Totals:** bordered boxes (`#000`), no gray fill.
- Drop light grays (`#ddd/#eee/#f7f7f7/#888/#999/#555/#aaa`) → `#000` for rules/labels so they print crisp.

Implementation: branch colors on `!isA4` in `labelCSS()` and in the inline bubble/row styles of `printBubble()` / `printList()`.

### 2. Worker label-field toggles (R1)

- New persisted config flags: `cfg.lblDel`, `cfg.lblPn`, `cfg.lblCoo`, default **`true`**.
- UI: checkboxes in the print/printer settings — "Show on label: Delivery / Part Number / COO".
- Render order on the label: **DEL → PN → COO**. Each field prints only if its toggle is ON **and** the field has data.
- **PO** stays always-shown when present (not part of the toggle set).
- Independent from the VERIFY scan toggles (`cfg.po/del/pn/coo`) — what you scan ≠ what you print.

### 3. QR bridge (R2)

Replace the current JSON `encodePallet` payload with a compact, versioned string:

```
PC1|<palletId>|<PN>|<PO>|<DEL>|<COO>|<q1,q2,...,qn>
```

- `q1…qn` = pieces per box, in B01…Bnn order.
- Pallet-level PN/PO/DEL/COO are **always** included in the QR (machine data, independent of the print toggles). Empty fields are left blank between pipes.
- **No per-box COO/PN** — quantities only (confirmed). Smaller QR, easier to scan on a small thermal label.
- Delimiter `|` (warehouse codes don't contain it). Version prefix `PC1` so the TC520L app can parse and evolve the format.
- QR stays on both thermal and A4 (it's black → prints on thermal).

The TC520L app: scans QR → reads quantities → computes combination → removes picked boxes → reprints updated label on the QLn420. (All outside PackCalc.)

## Touch points (code)

- `labelCSS(labelW, isA4)` — color branching by `!isA4`.
- `printBubble()` — bubble colors; meta fields order + toggles.
- `printList()` — row colors; meta fields order + toggles.
- `encodePallet()` — compact `PC1|…` format.
- `cfg` defaults + restore + print settings UI — add `lblDel` / `lblPn` / `lblCoo`.

## Verification (manual — no test suite/build)

1. Open PackCalc, scan a few boxes, open print preview:
   - A4 → unchanged (amber, all fields).
   - ZT / QLn → black/white, legible at 128×63 mm / 113×63 mm.
2. Turn DEL/PN/COO toggles off → those rows disappear; remaining order stays DEL → PN → COO; PO still shows.
3. Scan the printed QR → returns `PC1|…|q1,…,qn`; quantities match the on-screen boxes.

## Open questions

None.
