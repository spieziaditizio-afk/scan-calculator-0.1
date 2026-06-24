# Configurable box order on the printed label — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add an Advanced Settings toggle (`cfg.sortLabelByQty`) that, when on, prints the pieces-per-box bubbles in descending-quantity order (and renumbers `B01, B02…` to match) instead of scan order — print-only, on-screen chips unaffected.

**Architecture:** Single-file React-via-CDN app (`PackCalc.html`), no build step, no test runner — verification is **manual in the browser**. A new boolean lives in the existing `cfg` object (already persisted to `localStorage` wholesale, so no extra plumbing). A pure helper `sortBoxesForLabel` builds the (possibly reordered) boxes array; it's applied only at the two print call sites (`handlePrint`, `handlePrintPallet`), never inside `printBubble`/`printList`, which keep mapping over whatever boxes array they're handed.

**Tech Stack:** HTML/CSS/JSX (Babel standalone), React 18 UMD.

**Spec:** `docs/superpowers/specs/2026-06-24-label-box-sort-order-design.md`

## Global Constraints

- Default for the new setting is `false` (scan sequence) — must not change current printed output for existing users who don't touch the new checkbox.
- Scope is print-only: on-screen box chips and the segregation table must keep showing scan order regardless of this setting (per spec "Decisions confirmed with user").
- `printBubble` / `printList` are not modified — they only ever receive an already-ordered `boxes` array from their caller.
- QR payload order (`encodePallet`) is expected to follow the same sort when the toggle is on, since it reuses the same `pallet` object passed into `printBubble`/`printList` — this is intentional per spec section 4, not a bug to fix.
- Match existing Advanced Settings checkbox style exactly (same inline style object shape as `qrOnLabel`/`pnCheck`).

---

## File Structure

Only `PackCalc.html` changes:
- `cfg` state default (~L742) — add `sortLabelByQty:false`.
- New helper near `doPrint` (~L373) — `sortBoxesForLabel`.
- `handlePrint` (~L970-977) and `handlePrintPallet` (~L979-990) — sort boxes before building the object passed to `doPrint`.
- Advanced Settings modal, PRINT section (~L1526-1533) — new checkbox.

Line numbers are approximate — search for the quoted code before editing.

---

### Task 1: `cfg.sortLabelByQty` default + Advanced Settings checkbox

**Files:**
- Modify: `PackCalc.html` — `cfg` default (~L742), Advanced Settings PRINT section (~L1526-1533)

**Interfaces:**
- Produces: `cfg.sortLabelByQty` (boolean), read by Task 2.

- [ ] **Step 1: Add the new cfg field, default false**

Find:
```js
  const [cfg,          setCfg]         = useState({ pn:false, coo:false, po:false, del:false, printer:'a4', layout:'bubble', lblDel:true, lblPn:true, lblCoo:true, pnCheck:false, qrOnLabel:false });
```
Replace with:
```js
  const [cfg,          setCfg]         = useState({ pn:false, coo:false, po:false, del:false, printer:'a4', layout:'bubble', lblDel:true, lblPn:true, lblCoo:true, pnCheck:false, qrOnLabel:false, sortLabelByQty:false });
```

- [ ] **Step 2: Add the checkbox to the Advanced Settings PRINT section**

Find:
```jsx
            <div style={{marginTop:16,paddingTop:14,borderTop:`1px solid ${C.border}`}}>
              <div style={{fontSize:10,fontWeight:800,color:C.muted,letterSpacing:'.1em',fontFamily:"'Fira Sans',sans-serif",marginBottom:8}}>PRINT</div>
              <label style={{display:'flex',alignItems:'center',gap:10,padding:'7px 2px',cursor:'pointer'}}>
                <input type="checkbox" checked={cfg.qrOnLabel} onChange={e=>setCfg(c=>({...c,qrOnLabel:e.target.checked}))}
                  style={{width:16,height:16,accentColor:C.amber,cursor:'pointer',flexShrink:0}}/>
                <span style={{fontSize:13,fontWeight:600,color:C.text,fontFamily:"'Fira Sans',sans-serif"}}>Include QR on label (for TC520L scanning app)</span>
              </label>
            </div>
```
Replace with:
```jsx
            <div style={{marginTop:16,paddingTop:14,borderTop:`1px solid ${C.border}`}}>
              <div style={{fontSize:10,fontWeight:800,color:C.muted,letterSpacing:'.1em',fontFamily:"'Fira Sans',sans-serif",marginBottom:8}}>PRINT</div>
              <label style={{display:'flex',alignItems:'center',gap:10,padding:'7px 2px',cursor:'pointer'}}>
                <input type="checkbox" checked={cfg.qrOnLabel} onChange={e=>setCfg(c=>({...c,qrOnLabel:e.target.checked}))}
                  style={{width:16,height:16,accentColor:C.amber,cursor:'pointer',flexShrink:0}}/>
                <span style={{fontSize:13,fontWeight:600,color:C.text,fontFamily:"'Fira Sans',sans-serif"}}>Include QR on label (for TC520L scanning app)</span>
              </label>
              <label style={{display:'flex',alignItems:'center',gap:10,padding:'7px 2px',cursor:'pointer'}}>
                <input type="checkbox" checked={cfg.sortLabelByQty} onChange={e=>setCfg(c=>({...c,sortLabelByQty:e.target.checked}))}
                  style={{width:16,height:16,accentColor:C.amber,cursor:'pointer',flexShrink:0}}/>
                <span style={{fontSize:13,fontWeight:600,color:C.text,fontFamily:"'Fira Sans',sans-serif"}}>Sort boxes by quantity (highest first) on label</span>
              </label>
            </div>
```

- [ ] **Step 3: Verify in browser**

1. Open `PackCalc.html`, click the gear icon (Advanced Settings, app header).
2. Under **PRINT**, confirm a second checkbox reads "Sort boxes by quantity (highest first) on label", unchecked by default.
3. Check it, close the modal, reopen it → still checked (state holds).
4. Hard refresh the page → reopen Advanced Settings → still checked (persisted via the existing `cfg` localStorage round-trip).
5. Uncheck it for a clean baseline before Task 2.

- [ ] **Step 4: Commit**

```bash
git add PackCalc.html
git commit -m "feat(print): add sortLabelByQty setting and Advanced Settings checkbox"
```

---

### Task 2: Apply the sort at print time

**Files:**
- Modify: `PackCalc.html` — new helper near `doPrint` (~L373), `handlePrint` (~L975-976), `handlePrintPallet` (~L982-989)

**Interfaces:**
- Consumes: `cfg.sortLabelByQty` (from Task 1).
- Produces: `sortBoxesForLabel(boxes, sortByQty)` — pure function, returns the original array unchanged when `sortByQty` is false, otherwise a new array sorted descending by `qty` (stable, ties keep original order).

- [ ] **Step 1: Add the `sortBoxesForLabel` helper**

Find:
```js
const doPrint = (pallet, printerId, layout, hasMismatch, lbl) =>
  layout === 'bubble'
    ? printBubble(pallet, printerId, hasMismatch, lbl)
    : printList(pallet, printerId, hasMismatch, lbl);
```
Replace with:
```js
const sortBoxesForLabel = (boxes, sortByQty) =>
  sortByQty ? [...boxes].sort((a,b)=>b.qty-a.qty) : boxes;

const doPrint = (pallet, printerId, layout, hasMismatch, lbl) =>
  layout === 'bubble'
    ? printBubble(pallet, printerId, hasMismatch, lbl)
    : printList(pallet, printerId, hasMismatch, lbl);
```

- [ ] **Step 2: Use it in `handlePrint` (prints the active pallet)**

Find:
```js
    const pallet = {id:palletName,receivedAt:dateStr(),meta:{pn:effPn,coo:effCoo,po:effPo,ref:effRef,worker:packingList.worker||meta.worker||''},boxes};
    await doPrint(pallet,cfg.printer,cfg.layout,mismatchIds.size>0,{del:cfg.lblDel,pn:cfg.lblPn,coo:cfg.lblCoo,qr:cfg.qrOnLabel});
```
Replace with:
```js
    const pallet = {id:palletName,receivedAt:dateStr(),meta:{pn:effPn,coo:effCoo,po:effPo,ref:effRef,worker:packingList.worker||meta.worker||''},boxes:sortBoxesForLabel(boxes,cfg.sortLabelByQty)};
    await doPrint(pallet,cfg.printer,cfg.layout,mismatchIds.size>0,{del:cfg.lblDel,pn:cfg.lblPn,coo:cfg.lblCoo,qr:cfg.qrOnLabel});
```

- [ ] **Step 3: Use it in `handlePrintPallet` (prints a stored pallet from the pallet list)**

Find:
```js
  const handlePrintPallet = async (p, i) => {
    const bs=p.boxes;
    const dp=k=>[...new Set(bs.filter(b=>b[k]).map(b=>b[k]))];
    const ep={...p,id:`PLT ${i+1}`,meta:{
      pn:  p.meta.pn  ||(dp('pn').length===1?dp('pn')[0]:''),
      coo: p.meta.coo ||(dp('coo').length===1?dp('coo')[0]:''),
      po:  p.meta.po  ||(dp('po').length===1?dp('po')[0]:''),
      ref: p.meta.ref ||(dp('del').length===1?dp('del')[0]:''),
      worker: packingList.worker||p.meta.worker||'',
    }};
    await doPrint(ep,cfg.printer,cfg.layout,false,{del:cfg.lblDel,pn:cfg.lblPn,coo:cfg.lblCoo,qr:cfg.qrOnLabel});
  };
```
Replace with:
```js
  const handlePrintPallet = async (p, i) => {
    const bs=p.boxes;
    const dp=k=>[...new Set(bs.filter(b=>b[k]).map(b=>b[k]))];
    const ep={...p,id:`PLT ${i+1}`,boxes:sortBoxesForLabel(bs,cfg.sortLabelByQty),meta:{
      pn:  p.meta.pn  ||(dp('pn').length===1?dp('pn')[0]:''),
      coo: p.meta.coo ||(dp('coo').length===1?dp('coo')[0]:''),
      po:  p.meta.po  ||(dp('po').length===1?dp('po')[0]:''),
      ref: p.meta.ref ||(dp('del').length===1?dp('del')[0]:''),
      worker: packingList.worker||p.meta.worker||'',
    }};
    await doPrint(ep,cfg.printer,cfg.layout,false,{del:cfg.lblDel,pn:cfg.lblPn,coo:cfg.lblCoo,qr:cfg.qrOnLabel});
  };
```

- [ ] **Step 4: Verify in browser — active pallet print (`handlePrint`)**

1. Hard refresh, NEW SESSION (confirm if prompted) for a clean pallet.
2. Scan/type 4 boxes with quantities **5, 20, 3, 12** in that order (Enter after each).
3. Confirm the on-screen box chips show, left to right: `B01=5, B02=20, B03=3, B04=12`.
4. Open Advanced Settings → printer **A4 Sheet** → confirm "Sort boxes by quantity" is **unchecked** → close.
5. Click **PRINT** → the new tab's label shows bubbles in scan order: `B01=5, B02=20, B03=3, B04=12`.
6. Back in the app, open Advanced Settings → check "Sort boxes by quantity (highest first) on label" → close.
7. Click **PRINT** again → the new tab's label now shows bubbles reordered descending: `B01=20, B02=12, B03=5, B04=3`.
8. Confirm the on-screen box chips in the main app are still `B01=5, B02=20, B03=3, B04=12` (unchanged) — print-only scope holds.

- [ ] **Step 5: Verify in browser — stored pallet print (`handlePrintPallet`)**

1. With the checkbox still **checked** from Step 4.6, click **+ PALLET** to create a second pallet and switch to it.
2. Scan/type 3 boxes with quantities **8, 1, 15**.
3. Switch back to the pallet list view (or wherever the per-pallet print button lives) and click the print button for this pallet (not the main PRINT button — the one that calls `handlePrintPallet`).
4. Confirm the printed label shows bubbles reordered descending: `B01=15, B02=8, B03=1`.
5. Uncheck the setting in Advanced Settings, print the same pallet again → confirm it now prints `B01=8, B02=1, B03=15` (original scan order).

- [ ] **Step 6: Commit**

```bash
git add PackCalc.html
git commit -m "feat(print): sort label boxes by quantity when sortLabelByQty is enabled"
```

---

## Self-Review

**Spec coverage:**
- New `cfg.sortLabelByQty`, default `false` → Task 1, Step 1 ✓
- Checkbox in Advanced Settings PRINT section → Task 1, Step 2 ✓
- `sortBoxesForLabel` helper, stable sort, used at both print call sites, `printBubble`/`printList` untouched → Task 2, Steps 1-3 ✓
- On-screen chips/segregation table unaffected (print-only) → Task 2, Step 4.8 explicitly verifies this ✓
- QR payload follows the same sort (intentional, no separate task needed since it reads from the same already-sorted `pallet` object) → covered by Global Constraints note; no code change required ✓
- `printSummary` unaffected → not touched, no task needed ✓

**Placeholder scan:** none — every step has concrete find/replace code or a manual verification with an exact expected result.

**Type/name consistency:** `cfg.sortLabelByQty` (Task 1) is the exact name read in `sortBoxesForLabel(boxes, cfg.sortLabelByQty)` calls (Task 2). `sortBoxesForLabel` signature `(boxes, sortByQty)` matches both call sites (`boxes`/`bs` as first arg, `cfg.sortLabelByQty` as second). `handlePrintPallet`'s `ep` object gets `boxes:` set explicitly before the `...p` spread's `boxes` would otherwise win — confirmed object literal order: `{...p, id:..., boxes:..., meta:...}` means the explicit `boxes:` key overrides the one inherited from `...p` regardless of where it's written in the literal (later key wins on duplicate, and `boxes:` is written after `...p`).
