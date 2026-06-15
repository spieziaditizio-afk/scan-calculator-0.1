# Thermal label + TC520L bridge — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make PackCalc's pallet label print monochrome on warehouse thermal printers, let the worker toggle DEL/PN/COO on the label, and emit a compact QR that the Zebra TC520L app can read.

**Architecture:** Single-file React-via-CDN app (`PackCalc.html`), no build step, no test runner. Print "templates" are module-level functions that build a standalone HTML string opened in a new window. Verification is **manual in the browser** (CLAUDE.md: "Open in a browser"). A4 output stays in its current color palette; only thermal (`ZT`/`QLn`) output and the QR change. Label-field toggles live in `cfg` and are threaded into the module-level print functions as a parameter.

**Tech Stack:** HTML/CSS/JSX (Babel standalone), React 18 UMD, qrcode.js (already loaded).

**Spec:** `docs/superpowers/specs/2026-06-14-thermal-label-tc520l-bridge-design.md`

---

## File Structure

Only one file changes: `PackCalc.html`. The edits are localized to:
- `encodePallet()` (~L228) — QR payload format.
- `labelCSS()` (~L196) — shared print CSS; add a monochrome palette branch.
- `printBubble()` (~L318) — bubble colors + meta fields/order/toggles.
- `printList()` (~L239) — row colors + meta fields/order/toggles.
- `doPrint()` (~L376) — pass label-toggle options through.
- `App` `cfg` state (~L736) — add `lblDel/lblPn/lblCoo` defaults.
- Print Settings modal (~L1512) — "Show on label" checkboxes.
- `handlePrint` (~L959) / `handlePrintPallet` (~L972) — pass toggles to `doPrint`.

Line numbers are approximate — search for the quoted code before editing.

---

### Task 1: Compact QR payload

**Files:**
- Modify: `PackCalc.html` — `encodePallet()` (~L228-232)

- [ ] **Step 1: Replace the JSON payload with the compact format**

Find:
```js
const encodePallet = (pallet) => JSON.stringify({
  v:1, id:pallet.id,
  m:{ pn:pallet.meta.pn||'', coo:pallet.meta.coo||'', po:pallet.meta.po||'', ref:pallet.meta.ref||'' },
  b: pallet.boxes.map(b=>({ i:b.id, pn:b.pn||'', coo:b.coo||'', po:b.po||'', del:b.del||'', q:b.qty })),
});
```

Replace with:
```js
// Compact, versioned QR payload for the TC520L picking app.
// Format: PC1|<palletId>|<PN>|<PO>|<DEL>|<COO>|<q1,q2,...,qn>   (q = pieces per box, B01..Bnn order)
const encodePallet = (pallet) => {
  const m = pallet.meta || {};
  const qtys = pallet.boxes.map(b=>b.qty).join(',');
  return `PC1|${pallet.id}|${m.pn||''}|${m.po||''}|${m.ref||''}|${m.coo||''}|${qtys}`;
};
```

- [ ] **Step 2: Verify in browser**

1. Open `PackCalc.html` (hard refresh, Ctrl+F5).
2. Scan/add 3 boxes (type a code + Enter through the steps; at QTY type e.g. `20`, `10`, `25`).
3. Click **PRINT** (any printer). In the print-preview window, the QR is shown.
4. Scan that QR with a phone QR reader.

Expected: the decoded text is `PC1|PLT 1|...|...|...|...|20,10,25` (the trailing list matches the box quantities in order).

- [ ] **Step 3: Commit**

```bash
git add PackCalc.html
git commit -m "feat(label): compact versioned QR payload (PC1) for TC520L bridge"
```

---

### Task 2: Monochrome palette for thermal printers

**Files:**
- Modify: `PackCalc.html` — `labelCSS()` (~L196-225), `printBubble()` bubbles (~L324-330), `printList()` thermal rows (~L264-268)

- [ ] **Step 1: Add a monochrome color branch to `labelCSS`**

Replace the entire `labelCSS` function (from `const labelCSS = (labelW, isA4=false) => {` through its closing `};`) with:

```js
const labelCSS = (labelW, isA4=false) => {
  const p = isA4
    ? { gap:'8px', pad:'16px 20px', hdrPb:'8px', brand:11, pid:11, tag:'3px 9px', tagSz:11, pn:32, pnMix:16, metaGap:'20px', metaPt:'6px', mflbl:9, mfval:15, totGap:'8px', totPt:'6px', totPad:'6px 14px', totLbl:9, totVal:24, ftrPt:'6px', ft:9.5, ok:10, okPad:'3px 10px' }
    : { gap:'5px', pad:'9px 12px',  hdrPb:'5px', brand:9,  pid:8.5,tag:'2px 7px', tagSz:9,  pn:22, pnMix:13, metaGap:'12px', metaPt:'4px', mflbl:7, mfval:11, totGap:'4px', totPt:'4px', totPad:'3px 7px',  totLbl:7, totVal:16, ftrPt:'4px', ft:7.5, ok:8,  okPad:'2px 7px' };
  // Palette: A4 = color (paper); thermal = monochrome (black on white only)
  const k = isA4
    ? { sub:'#888', subAlt:'#555', pnMixFg:'#aaa', ftFg:'#999', line:'#ddd', tagBg:'#fffbe6', tagFg:'#92600a', tagBd:'1px solid #f0c040', totBg:'#f7f7f7', totBd:'none', okBg:'#14532d', okFg:'#22c55e', warnBg:'#7c1c1c', warnFg:'#f87171', warnBd:'none' }
    : { sub:'#000', subAlt:'#000', pnMixFg:'#000', ftFg:'#000', line:'#000', tagBg:'#fff',    tagFg:'#000',    tagBd:'1.5px solid #000', totBg:'#fff',    totBd:'1.5px solid #000', okBg:'#000',    okFg:'#fff',    warnBg:'#fff',    warnFg:'#000',    warnBd:'2px solid #000' };
  return `
  *{margin:0;padding:0;box-sizing:border-box;}
  body{background:#f0f0f0;display:flex;flex-direction:column;align-items:center;justify-content:${isA4?'flex-start':'center'};min-height:100vh;font-family:'Inter',sans-serif;gap:14px;padding:${isA4?'30px 20px':'20px'};}
  .label{width:${labelW};background:#fff;border:1.5px solid #111;padding:${p.pad};display:flex;flex-direction:column;gap:${p.gap};}
  .hdr{display:flex;justify-content:space-between;align-items:center;border-bottom:2px solid #111;padding-bottom:${p.hdrPb};gap:6px;}
  .hdr-left{flex:1;min-width:0;}
  .brand{font-size:${p.brand}px;font-weight:900;letter-spacing:.06em;white-space:nowrap;}
  .pid{font-family:'JetBrains Mono',monospace;font-size:${p.pid}px;font-weight:800;color:${k.subAlt};letter-spacing:.04em;}
  .tag{font-family:'JetBrains Mono',monospace;font-size:${p.tagSz}px;font-weight:900;padding:${p.tag};border-radius:3px;background:${k.tagBg};color:${k.tagFg};border:${k.tagBd};white-space:nowrap;flex-shrink:0;}
  .pn{font-family:'JetBrains Mono',monospace;font-size:${p.pn}px;font-weight:900;color:#111;letter-spacing:.02em;}
  .pn-mixed{font-family:'JetBrains Mono',monospace;font-size:${p.pnMix}px;font-weight:700;color:${k.pnMixFg};}
  .meta{display:flex;flex-wrap:wrap;gap:${p.metaGap};border-top:1px solid ${k.line};padding-top:${p.metaPt};}
  .mf .lbl{font-size:${p.mflbl}px;font-weight:700;letter-spacing:.1em;color:${k.sub};}
  .mf .val{font-family:'JetBrains Mono',monospace;font-size:${p.mfval}px;font-weight:800;color:#111;}
  .totals{display:grid;grid-template-columns:1fr 1fr;gap:${p.totGap};border-top:1px solid ${k.line};padding-top:${p.totPt};}
  .tot-block{background:${k.totBg};border:${k.totBd};border-radius:3px;padding:${p.totPad};}
  .tot-lbl{font-size:${p.totLbl}px;font-weight:700;letter-spacing:.1em;color:${k.sub};}
  .tot-val{font-family:'JetBrains Mono',monospace;font-size:${p.totVal}px;font-weight:900;color:#111;}
  .ftr{display:flex;justify-content:space-between;align-items:center;border-top:1px solid ${k.line};padding-top:${p.ftrPt};}
  .ft{font-size:${p.ft}px;color:${k.ftFg};}
  .ok{padding:${p.okPad};border-radius:3px;font-size:${p.ok}px;font-weight:800;background:${k.okBg};color:${k.okFg};white-space:nowrap;}
  .warn-badge{padding:${p.okPad};border-radius:3px;font-size:${p.ok}px;font-weight:800;background:${k.warnBg};color:${k.warnFg};border:${k.warnBd};white-space:nowrap;}
  .btn{padding:9px 24px;border:none;border-radius:6px;background:#ffb700;color:#07080b;font-weight:800;font-size:13px;cursor:pointer;}
  @media print{body{background:#fff;}button{display:none!important;}}
  `;
};
```

(`.btn` stays amber — it's the on-screen Print button, hidden by `@media print`.)

- [ ] **Step 2: Make the bubbles monochrome on thermal in `printBubble`**

In `printBubble`, find the line that defines `sz` and the bubbles, near:
```js
  const sz       = bubbleSize(boxes.length, isA4);
  const qrSz     = isA4 ? 64 : 46;
  const bubbles  = boxes.map((b,i)=>`
    <div style="display:flex;flex-direction:column;align-items:center;justify-content:center;width:${sz.w}px;height:${sz.h}px;border:1.5px solid #ffb700;border-radius:6px;background:#fffbe6;">
      <div style="font-family:'JetBrains Mono',monospace;font-size:${bubbleFontSize(b.qty,sz.fn,sz.w)}px;font-weight:900;color:#111;line-height:1;">${b.qty}</div>
      <div style="font-size:${sz.fs}px;font-weight:700;color:#888;letter-spacing:.04em;margin-top:2px;">B${String(i+1).padStart(2,'0')}</div>
    </div>`).join('');
```

Replace with:
```js
  const sz       = bubbleSize(boxes.length, isA4);
  const qrSz     = isA4 ? 64 : 46;
  const bubBd    = isA4 ? '1.5px solid #ffb700' : '2px solid #000';
  const bubBg    = isA4 ? '#fffbe6' : '#fff';
  const bubSub   = isA4 ? '#888' : '#000';
  const bubbles  = boxes.map((b,i)=>`
    <div style="display:flex;flex-direction:column;align-items:center;justify-content:center;width:${sz.w}px;height:${sz.h}px;border:${bubBd};border-radius:6px;background:${bubBg};">
      <div style="font-family:'JetBrains Mono',monospace;font-size:${bubbleFontSize(b.qty,sz.fn,sz.w)}px;font-weight:900;color:#111;line-height:1;">${b.qty}</div>
      <div style="font-size:${sz.fs}px;font-weight:700;color:${bubSub};letter-spacing:.04em;margin-top:2px;">B${String(i+1).padStart(2,'0')}</div>
    </div>`).join('');
```

- [ ] **Step 3: Darken the thermal list rows in `printList`**

In `printList`, in the `else` (thermal) branch, find:
```js
      <div style="display:flex;justify-content:space-between;align-items:center;padding:1px 3px;break-inside:avoid;border-bottom:1px solid #f0f0f0;">
        <span style="font-size:6.5px;color:#888;font-weight:700;white-space:nowrap;">#${String(i+1).padStart(2,'0')}</span>
```

Replace the `color:#888` in that line with `color:#000`:
```js
      <div style="display:flex;justify-content:space-between;align-items:center;padding:1px 3px;break-inside:avoid;border-bottom:1px solid #f0f0f0;">
        <span style="font-size:6.5px;color:#000;font-weight:700;white-space:nowrap;">#${String(i+1).padStart(2,'0')}</span>
```

- [ ] **Step 4: Verify in browser**

1. Hard refresh `PackCalc.html`. Add 3 boxes.
2. Gear (PRINT SETTINGS) → choose **A4 Sheet** → PRINT. Expected: label looks **unchanged** (amber bubbles, amber RECEIVING tag, green VERIFIED).
3. Gear → choose **Zebra ZT** → PRINT. Expected: **black/white only** — black-bordered white bubbles, outlined RECEIVING tag, solid black VERIFIED box with white text, bordered total boxes, no faint grays.
4. Repeat with **Zebra QLn420** → same monochrome look, slightly narrower.

- [ ] **Step 5: Commit**

```bash
git add PackCalc.html
git commit -m "feat(label): monochrome rendering for thermal printers (A4 stays color)"
```

---

### Task 3: Label-field toggle config + Print Settings UI

**Files:**
- Modify: `PackCalc.html` — `cfg` default (~L736), Print Settings modal (~L1512)

- [ ] **Step 1: Add toggle defaults to `cfg`**

Find:
```js
  const [cfg,          setCfg]         = useState({ pn:true, coo:true, po:true, del:true, printer:'a4', layout:'bubble' });
```

Replace with:
```js
  const [cfg,          setCfg]         = useState({ pn:true, coo:true, po:true, del:true, printer:'a4', layout:'bubble', lblDel:true, lblPn:true, lblCoo:true });
```

(Session restore uses `setCfg(p=>({...p,...(s.cfg||{})}))`, so missing keys keep these defaults — no other change needed.)

- [ ] **Step 2: Add the "Show on label" checkboxes to the Print Settings modal**

In the Print Settings modal, find the end of the printer list — the `})}` that closes `Object.entries(PRINTERS).map(...)` followed by its container `</div>`:
```jsx
                );
              })}
            </div>
          </div>
        </div>
      )}
```

Replace with (inserts the toggle section after the printer list `</div>`):
```jsx
                );
              })}
            </div>
            <div style={{marginTop:16,paddingTop:14,borderTop:`1px solid ${C.border}`}}>
              <div style={{fontSize:10,fontWeight:800,color:C.muted,letterSpacing:'.1em',fontFamily:"'Fira Sans',sans-serif",marginBottom:8}}>SHOW ON LABEL</div>
              {[['lblDel','Delivery'],['lblPn','Part Number'],['lblCoo','COO']].map(([key,label])=>(
                <label key={key} style={{display:'flex',alignItems:'center',gap:10,padding:'7px 2px',cursor:'pointer'}}>
                  <input type="checkbox" checked={cfg[key]} onChange={e=>setCfg(c=>({...c,[key]:e.target.checked}))}
                    style={{width:16,height:16,accentColor:C.amber,cursor:'pointer',flexShrink:0}}/>
                  <span style={{fontSize:13,fontWeight:600,color:C.text,fontFamily:"'Fira Sans',sans-serif"}}>{label}</span>
                </label>
              ))}
            </div>
          </div>
        </div>
      )}
```

- [ ] **Step 3: Verify in browser**

1. Hard refresh. Click the gear (PRINT SETTINGS).
2. Expected: below the printer list, a **SHOW ON LABEL** section with **Delivery / Part Number / COO**, all three checked.
3. Uncheck COO, close the modal, reload the page, reopen settings. Expected: COO is **still unchecked** (persisted via the existing `packcalc` localStorage).
4. Re-check COO before moving on.

- [ ] **Step 4: Commit**

```bash
git add PackCalc.html
git commit -m "feat(label): worker toggles for DEL/PN/COO on label (default on, persisted)"
```

---

### Task 4: Apply toggles + DEL→PN→COO order on the label

**Files:**
- Modify: `PackCalc.html` — `doPrint` (~L376), `printBubble` signature + meta (~L318, ~L351), `printList` signature + PN/meta (~L239, ~L293-298), `handlePrint` (~L959), `handlePrintPallet` (~L972)

- [ ] **Step 1: Thread a `lbl` options object through `doPrint`**

Find:
```js
const doPrint = (pallet, printerId, layout, hasMismatch) =>
  layout === 'bubble'
    ? printBubble(pallet, printerId, hasMismatch)
    : printList(pallet, printerId, hasMismatch);
```

Replace with:
```js
const doPrint = (pallet, printerId, layout, hasMismatch, lbl) =>
  layout === 'bubble'
    ? printBubble(pallet, printerId, hasMismatch, lbl)
    : printList(pallet, printerId, hasMismatch, lbl);
```

- [ ] **Step 2: Accept `lbl` in `printBubble` and gate its meta fields in DEL→PN→COO order**

Change the `printBubble` signature line:
```js
const printBubble = async (pallet, printerId, hasMismatch) => {
```
to:
```js
const printBubble = async (pallet, printerId, hasMismatch, lbl={del:true,pn:true,coo:true}) => {
```

Then find the bubble template's meta block:
```js
    <div class="meta">
      ${meta.po?`<div class="mf"><div class="lbl">PO</div><div class="val">${meta.po}</div></div>`:''}
      ${meta.ref?`<div class="mf"><div class="lbl">DELIVERY</div><div class="val">${meta.ref}</div></div>`:''}
      <div class="mf"><div class="lbl">PN</div><div class="val">${meta.pn||'—'}</div></div>
      ${meta.coo?`<div class="mf"><div class="lbl">COO</div><div class="val">${meta.coo}</div></div>`:''}
      ${meta.worker?`<div class="mf"><div class="lbl">WORKER</div><div class="val">${meta.worker}</div></div>`:''}
    </div>
```
Replace with (PO always when present; then DEL → PN → COO gated by toggle **and** data; worker unchanged):
```js
    <div class="meta">
      ${meta.po?`<div class="mf"><div class="lbl">PO</div><div class="val">${meta.po}</div></div>`:''}
      ${lbl.del&&meta.ref?`<div class="mf"><div class="lbl">DELIVERY</div><div class="val">${meta.ref}</div></div>`:''}
      ${lbl.pn&&meta.pn?`<div class="mf"><div class="lbl">PN</div><div class="val">${meta.pn}</div></div>`:''}
      ${lbl.coo&&meta.coo?`<div class="mf"><div class="lbl">COO</div><div class="val">${meta.coo}</div></div>`:''}
      ${meta.worker?`<div class="mf"><div class="lbl">WORKER</div><div class="val">${meta.worker}</div></div>`:''}
    </div>
```

- [ ] **Step 3: Accept `lbl` in `printList` and gate the PN title + meta fields**

Change the `printList` signature line:
```js
const printList = async (pallet, printerId, hasMismatch) => {
```
to:
```js
const printList = async (pallet, printerId, hasMismatch, lbl={del:true,pn:true,coo:true}) => {
```

Find the PN title line:
```js
    ${meta.pn?`<div class="pn">${meta.pn}</div>`:`<div class="pn-mixed">— mixed / no PN —</div>`}
```
Replace with (gate the whole PN title by the toggle):
```js
    ${lbl.pn?(meta.pn?`<div class="pn">${meta.pn}</div>`:`<div class="pn-mixed">— mixed / no PN —</div>`):''}
```

Find the meta block:
```js
    <div class="meta">
      ${meta.coo?`<div class="mf"><div class="lbl">COO</div><div class="val">${meta.coo}</div></div>`:''}
      ${meta.po?`<div class="mf"><div class="lbl">PO</div><div class="val">${meta.po}</div></div>`:''}
      ${meta.ref?`<div class="mf"><div class="lbl">DELIVERY</div><div class="val">${meta.ref}</div></div>`:''}
    </div>
```
Replace with (PO always when present; DEL then COO gated; PN already handled as the title above):
```js
    <div class="meta">
      ${meta.po?`<div class="mf"><div class="lbl">PO</div><div class="val">${meta.po}</div></div>`:''}
      ${lbl.del&&meta.ref?`<div class="mf"><div class="lbl">DELIVERY</div><div class="val">${meta.ref}</div></div>`:''}
      ${lbl.coo&&meta.coo?`<div class="mf"><div class="lbl">COO</div><div class="val">${meta.coo}</div></div>`:''}
    </div>
```

- [ ] **Step 4: Pass the toggles from both print handlers**

In `handlePrint`, find:
```js
    await doPrint(pallet,cfg.printer,cfg.layout,mismatchIds.size>0);
```
Replace with:
```js
    await doPrint(pallet,cfg.printer,cfg.layout,mismatchIds.size>0,{del:cfg.lblDel,pn:cfg.lblPn,coo:cfg.lblCoo});
```

In `handlePrintPallet`, find:
```js
    await doPrint(ep,cfg.printer,cfg.layout,false);
```
Replace with:
```js
    await doPrint(ep,cfg.printer,cfg.layout,false,{del:cfg.lblDel,pn:cfg.lblPn,coo:cfg.lblCoo});
```

- [ ] **Step 5: Verify in browser**

1. Hard refresh. Add 3 boxes; in the left panel fill the packing-list fields so the pallet has PO, Delivery, PN and COO data.
2. Gear → all three SHOW ON LABEL checked → PRINT (bubble layout, ZT). Expected: meta shows **PO, then DELIVERY, then PN, then COO** (DEL→PN→COO order).
3. Gear → uncheck **Part Number** and **COO** → PRINT again. Expected: PN and COO rows are **gone**; DELIVERY still shows; PO still shows.
4. Switch layout to list (if exposed) and repeat the toggle check; PN title appears/disappears with its toggle.
5. Print a stored pallet via the per-pallet print button → same toggle behavior.

- [ ] **Step 6: Commit**

```bash
git add PackCalc.html
git commit -m "feat(label): honor DEL/PN/COO toggles in DEL→PN→COO order on printed label"
```

---

## Self-Review

**Spec coverage:**
- Monochrome thermal rendering → Task 2 ✓
- A4 stays color → Task 2 (A4 branch unchanged) ✓
- Worker toggles DEL/PN/COO, default ON, persisted → Task 3 ✓
- DEL→PN→COO order, print only if toggle ON + data; PO always; independent of VERIFY → Task 4 ✓
- Compact QR `PC1|…` quantities-only, always carries pallet header → Task 1 ✓
- QR stays on thermal + A4 (black) → unchanged (still rendered); Task 1 only changes the payload ✓
- Out of scope (combination, label update, SPQ) → not touched ✓

**Placeholder scan:** none — every step has concrete find/replace code and a manual verification with expected result.

**Type/name consistency:** `lbl` object shape `{del,pn,coo}` is identical in `doPrint`, `printBubble`, `printList`, and both handlers. `cfg.lblDel/lblPn/lblCoo` names match between Task 3 (defaults + UI) and Task 4 (handlers). `encodePallet` still returns a string consumed by `genQR` (unchanged caller).

**Note on commits:** this repo appears to auto-commit edits; the explicit `git commit` steps are the intended granularity but may already be captured. The executor should confirm each task's change is committed before moving on.
