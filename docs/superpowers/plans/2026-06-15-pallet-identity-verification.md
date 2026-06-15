# Pallet identity + verification options — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a per-pallet "Pallet Identity" card (PN scanned once, PO/Delivery/COO typed) that feeds the label, make per-box scanning count pieces only by default, and flag any box whose scanned PN ≠ the pallet PN.

**Architecture:** Single-file React-via-CDN app (`PackCalc.html`), no build step, no test runner — verification is **manual in the browser**. Identity is stored in the active pallet's existing `meta` object (today initialized empty and only read via the `meta.pn || distinct(...)` fallback in print). Populating `meta` from a new UI card makes the label use it with no print changes. Per-box "verify" reuses the existing VERIFY toggles + mismatch detection, enhanced to compare against `meta.pn`.

**Tech Stack:** HTML/CSS/JSX (Babel standalone), React 18 UMD.

**Spec:** `docs/superpowers/specs/2026-06-15-pallet-identity-verification-design.md`

---

## File Structure

Only `PackCalc.html` changes:
- `App` `cfg` state default (~L736) — per-box optional fields default to `false`.
- Scan area, top of `panel-left` content (~L1099) — new **Pallet Identity** card bound to `meta`.
- VERIFY toggle buttons (~L1106) — add clarifying `title` tooltips.
- `mismatchIds` `useMemo` (~L839-865) — compare PN against `meta.pn` when set.

Line numbers are approximate — search for the quoted code before editing.

---

### Task 1: Per-box default = count only; clarify VERIFY toggles

**Files:**
- Modify: `PackCalc.html` — `cfg` default (~L736), VERIFY toggle button (~L1106)

- [ ] **Step 1: Default the optional scan fields to off**

Find:
```js
  const [cfg,          setCfg]         = useState({ pn:true, coo:true, po:true, del:true, printer:'a4', layout:'bubble', lblDel:true, lblPn:true, lblCoo:true });
```
Replace with:
```js
  const [cfg,          setCfg]         = useState({ pn:false, coo:false, po:false, del:false, printer:'a4', layout:'bubble', lblDel:true, lblPn:true, lblCoo:true });
```

- [ ] **Step 2: Add clarifying tooltips to the VERIFY toggle buttons**

Find:
```jsx
                    <button key={f.key} className="ftoggle"
                      onClick={()=>setCfg(c=>({...c,[f.key]:!c[f.key]}))}
```
Replace with:
```jsx
                    <button key={f.key} className="ftoggle"
                      title={f.key==='pn'?'Verify PN on every box — flags any box ≠ pallet PN':`Scan ${f.short} on every box`}
                      onClick={()=>setCfg(c=>({...c,[f.key]:!c[f.key]}))}
```

- [ ] **Step 3: Verify in browser**

1. Open `PackCalc.html`, click **NEW SESSION** (to clear any saved cfg), confirm.
2. The **VERIFY** row chips (PO/DEL/PN/COO) are all **off** (gray, unchecked).
3. The scan prompt asks for **QTY only** ("TYPE OR SCAN PIECE QTY") — no PO/DEL/PN/COO steps.
4. Hover the PN chip → tooltip "Verify PN on every box — flags any box ≠ pallet PN".

- [ ] **Step 4: Commit**

```bash
git add PackCalc.html
git commit -m "feat(verify): per-box scanning defaults to QTY-only; tooltip on VERIFY toggles"
```

---

### Task 2: Pallet Identity card

**Files:**
- Modify: `PackCalc.html` — top of scan content (~L1099-1101)

- [ ] **Step 1: Insert the Pallet Identity card**

Find:
```jsx
            <div style={{width:'100%',maxWidth:760}}>

              {/* Config row */}
```
Replace with:
```jsx
            <div style={{width:'100%',maxWidth:760}}>

              {/* Pallet identity */}
              <div style={{background:C.surface,border:`1px solid ${C.border}`,borderRadius:10,padding:'12px 14px',marginBottom:12}}>
                <div style={{fontSize:10,fontWeight:800,color:C.muted,letterSpacing:'.1em',fontFamily:"'Fira Sans',sans-serif",marginBottom:8}}>PALLET IDENTITY</div>
                <div style={{display:'flex',gap:10,flexWrap:'wrap'}}>
                  <div style={{flex:'1 1 220px',minWidth:170}}>
                    <label style={{fontSize:9,fontWeight:700,letterSpacing:'.08em',color:C.sec,fontFamily:"'Fira Sans',sans-serif"}}>PART NUMBER · scan box 1</label>
                    <input value={meta.pn||''}
                      onChange={e=>updActive(p=>({...p,meta:{...p.meta,pn:e.target.value.toUpperCase()}}))}
                      onKeyDown={e=>{if(e.key==='Enter'){e.preventDefault();e.target.blur();inputRef.current?.focus();}}}
                      placeholder="scan PN barcode…"
                      style={{width:'100%',marginTop:4,padding:'8px 10px',minHeight:38,background:C.s2,border:`1px solid ${C.border}`,borderRadius:6,color:C.text,fontFamily:"'JetBrains Mono',monospace",fontSize:13,fontWeight:700,outline:'none'}}/>
                  </div>
                  {[['po','PO'],['ref','DELIVERY'],['coo','COO']].map(([key,label])=>(
                    <div key={key} style={{flex:'1 1 130px',minWidth:110}}>
                      <label style={{fontSize:9,fontWeight:700,letterSpacing:'.08em',color:C.sec,fontFamily:"'Fira Sans',sans-serif"}}>{label}</label>
                      <input value={meta[key]||''}
                        onChange={e=>updActive(p=>({...p,meta:{...p.meta,[key]:e.target.value.toUpperCase()}}))}
                        placeholder="type…"
                        style={{width:'100%',marginTop:4,padding:'8px 10px',minHeight:38,background:C.s2,border:`1px solid ${C.border}`,borderRadius:6,color:C.text,fontFamily:"'JetBrains Mono',monospace",fontSize:13,fontWeight:700,outline:'none'}}/>
                    </div>
                  ))}
                </div>
              </div>

              {/* Config row */}
```

(`meta` is the active pallet's identity — `const meta = ap.meta` already exists. `updActive` updates the active pallet. DELIVERY is stored in `meta.ref`.)

- [ ] **Step 2: Verify in browser**

1. Hard refresh. At the top of the scan area there is a **PALLET IDENTITY** card with PART NUMBER + PO + DELIVERY + COO inputs.
2. Click PART NUMBER and scan/type a PN (e.g. `MT40A1G8SA-075`); press Enter → focus returns to the scan input.
3. Type PO / DELIVERY / COO.
4. Add 2 boxes (enter quantities). Gear → **A4** → PRINT → the label shows the PN/PO/DELIVERY/COO you entered (DELIVERY/PN/COO respecting the "Show on label" toggles; PO always).
5. Add a **second pallet** (+ PALLET), switch to it → its Identity card is empty (independent per pallet). Switch back → values restored.
6. Reload the page → identity values persist per pallet.

- [ ] **Step 3: Commit**

```bash
git add PackCalc.html
git commit -m "feat(verify): per-pallet Pallet Identity card (PN scanned, PO/DEL/COO typed)"
```

---

### Task 3: Flag boxes whose PN ≠ pallet PN

**Files:**
- Modify: `PackCalc.html` — `mismatchIds` `useMemo` (~L852-865)

- [ ] **Step 1: Use the pallet PN as the reference for PN, and add it to deps**

Find:
```js
      modes[fk] = Object.entries(freq).sort((a,b)=>b[1]-a[1])[0][0];
    });

    boxes.forEach(b=>{
```
Replace with:
```js
      modes[fk] = Object.entries(freq).sort((a,b)=>b[1]-a[1])[0][0];
    });
    if(meta.pn && trackedKeys.includes('pn')) modes.pn = meta.pn.toUpperCase();

    boxes.forEach(b=>{
```

Then find the end of that same `useMemo`:
```js
    return { mismatchIds:ids, mismatchFields:bads, modesMap:modes };
  },[boxes, cfg]);
```
Replace with:
```js
    return { mismatchIds:ids, mismatchFields:bads, modesMap:modes };
  },[boxes, cfg, meta.pn]);
```

- [ ] **Step 2: Verify in browser**

1. Hard refresh, NEW SESSION. In PALLET IDENTITY set PART NUMBER = `AAA-111`.
2. Turn the **PN** VERIFY chip **on**.
3. Add a box scanning/typing PN `AAA-111` then a quantity → chip is normal (not red).
4. Add another box with PN `BBB-222` then a quantity → that box chip is **red** (mismatch), because its PN ≠ the pallet PN.
5. Turn PN VERIFY off → per-box prompt returns to QTY-only.

- [ ] **Step 3: Commit**

```bash
git add PackCalc.html
git commit -m "feat(verify): flag boxes whose scanned PN differs from the pallet PN"
```

---

## Self-Review

**Spec coverage:**
- Pallet Identity card (PN scanned, PO/DEL/COO typed, top of scan area, → `meta`, → label) → Task 2 ✓
- Per-box QTY-only by default → Task 1 (cfg defaults false) ✓
- Optional "Verify PN per box" flagging boxes ≠ pallet PN → Task 1 (toggle remains + tooltip) + Task 3 (meta.pn comparison) ✓
- PO/DEL/COO not scanned per box (visual) → not added to per-box flow; only typed in the card ✓
- Packing list form + cross-check unchanged → not touched ✓
- No print/label changes (label reads `meta`) → confirmed; no task touches print ✓

**Placeholder scan:** none — every step has concrete find/replace code and a manual verification with expected result.

**Type/name consistency:** Identity card writes `meta.pn/po/ref/coo` via `updActive(p=>({...p,meta:{...p.meta,...}}))`; `meta` is `ap.meta` (existing). DELIVERY ↔ `meta.ref` consistent with the print code's `effRef = meta.ref`. Task 3 reads `meta.pn` (same property) and adds it to the `useMemo` deps. `cfg.pn` is the same flag used by `steps`/`trackedKeys` and Task 1's default.

**Note on commits:** this repo auto-commits edits; explicit `git commit` steps are the intended granularity but may already be captured.
