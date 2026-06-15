# Pallet identity + verification options — PackCalc design

**Date:** 2026-06-15
**App:** PackCalc.html (standalone React-via-CDN, no build step)
**Status:** Approved (design); ready for implementation plan

## Context

Rule: every pallet's boxes are ordered and share a **single PO, a single Delivery, a single PN, and a single COO**. During receiving the verifier needs to:
- Capture the pallet's identity for the printed label. The **PN** is a long alphanumeric code that is error-prone to type or compare by eye, so it should be **scanned** (from the first box's label barcode). **PO / Delivery / COO** are short and easy to compare visually, so they are **typed**.
- Count pieces per box.
- Optionally scan every box's PN to confirm they all match (enforce the single-PN rule).
- Packing-list reconciliation is done **visually against the paper document**, not in the app.

Today `meta` (per-pallet identity: `pn/po/ref/coo`) is initialized empty and never edited by any UI — the label derives identity from scanned box fields via `meta.x || distinct(...)`. So populating `meta` from a UI makes the label use it automatically (no print changes needed).

## Goals

1. Add a **Pallet Identity** card at the top of the scan area (right panel) where the verifier sets, **once per pallet**: PN (scanner-filled), PO, Delivery, COO (typed). These populate the active pallet's `meta` and therefore the printed label.
2. Make per-box scanning **count pieces (QTY) only** by default (faster; identity already captured once).
3. Keep an **optional "Verify PN per box"** mode: scan each box's PN, and flag any box whose PN differs from the pallet PN.
4. Keep the packing-list form and cross-check unchanged (visual reconciliation is the worker's job; the in-app cross-check stays available).

## Non-goals

- Scanning PO/Delivery/COO per box (they are typed once / compared visually).
- Automatic packing-list match enforcement (done visually).
- SPQ / partial-box marking (separate parked feature).
- Print/label layout changes (label already reads `meta`).

## Design

### 1. Pallet Identity card (new)

A compact card rendered at the **top of the scan area** (the `panel-left` CSS class = visually right), above the step chips / scan zone. Bound to the **active pallet's `meta`** (so each PLT tab has its own identity):

- **PART NUMBER** — text input the **scanner fills** (focus it, scan the first box's PN barcode → value). Editable. Writes `meta.pn`. Placeholder hints "scan box 1 barcode". Pressing Enter blurs/returns focus to the box-count input.
- **PO** — typed text input → `meta.po`.
- **DELIVERY** — typed text input → `meta.ref`  (note: delivery is stored in `meta.ref`).
- **COO** — typed text input → `meta.coo`.

Binding: each input updates the active pallet via `updActive(p => ({...p, meta:{...p.meta, <field>:value}}))`. Values are uppercased on write (consistent with `commit`). Per-pallet `meta` is already persisted inside `pallets` in `localStorage`.

Because the label/print already uses `meta.pn || distinctPn[0]` (and the same for po/coo/ref), no print-template change is required.

### 2. Per-box scanning = QTY only by default

Change the `cfg` defaults so the optional scan fields start **off**:
`pn:false, coo:false, po:false, del:false` (was all `true`). With all optional fields off, `steps = FIELDS.filter(f=>!f.optional||cfg[f.key])` resolves to **`[qty]`** — the verifier only counts pieces per box.

(The existing VERIFY toggles remain available for anyone who wants per-box scanning — this is a default change, not a removal.)

### 3. "Verify PN per box" (optional)

Surfaced as a clearly labeled control (reuses the existing `cfg.pn` VERIFY toggle, relabeled "Verify PN on every box"). When on:
- PN is added to the per-box steps → the verifier scans each box's PN.
- Mismatch detection flags any box whose PN does not match. Enhance the existing `mismatchIds` logic: **when `meta.pn` is set, compare each box's `pn` against `meta.pn`** (a box ≠ pallet PN is flagged); when `meta.pn` is empty, fall back to the current mode-based "match each other" logic.

PO/Delivery/COO are compared visually by the worker and are not part of per-box scanning.

### 4. Packing list — unchanged

The packing-list form (left panel) and `plCrossCheck` stay as-is. In QTY-only mode boxes carry no po/del/coo, so the cross-check is largely inert; in PN-verify mode it still cross-checks PN. This is acceptable — reconciliation is primarily visual now.

## Data model / touch points

- `initPallet()` — `meta` shape unchanged (`{pn,coo,po,ref,worker}`); now also written by the Identity card.
- `cfg` default (`useState`) — set `pn/coo/po/del` to `false`.
- New Identity card JSX in the scan area (top of `panel-left`), bound to `meta` via `updActive`.
- `mismatchIds` `useMemo` — add the `meta.pn` comparison branch for PN.
- No change to `handlePrint` / `handlePrintPallet` / print templates (they already read `meta`).

## Verification (manual — no test suite)

1. Open PackCalc, hard refresh. The scan area shows a **PALLET IDENTITY** card on top.
2. Focus PART NUMBER, scan (or type) a PN → it appears; PRINT → the label shows that PN.
3. Type PO / Delivery / COO → PRINT → they appear on the label (respecting the print toggles).
4. Default per-box prompt is **QTY only** (no PO/DEL/PN/COO steps). Add 3 boxes by entering quantities.
5. Switch to a second PLT tab → its Identity card is independent (empty until set).
6. Turn on **Verify PN on every box** → each box now prompts a PN scan; enter a PN that differs from the pallet PN → that box chip is flagged red.
7. Reload → identity values persist per pallet.

## Open questions

None.

---

## Revision — 2026-06-15 (supersedes the design above)

After review the Pallet Identity card and the inline VERIFY toggles were dropped. Final shipped design:

- **Removed:** the Pallet Identity card and the inline VERIFY toggle row (scan area is now clean).
- **Per-box scanning = QTY only.** No inline identity scanning.
- **PN check (Option C):** opt-in tool. Print settings → **VERIFICATION** → "PN check" toggle (`cfg.pnCheck`, default off). When on, a transient **PN CHECK** panel appears at the top of the scan area: scan each box's PN barcode → live verdict **✓ MATCH (PN)** / **✗ MISMATCH (a ≠ b)**; RESET clears. No per-box record. Usable before or after counting (no fixed order).
- On **✓ MATCH**, a **"use this PN on label"** button stamps the confirmed PN into `meta.pn` so the printed label shows it (the only PN→label path now).
- New state: `pnScans` (array of scanned PNs in the current check), `pnInput` (string). The earlier per-box mismatch / `meta.pn` comparison logic remains but is **inert** (no per-box identity scanning happens).
