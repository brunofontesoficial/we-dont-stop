# Session Templates Redesign Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the CORE blocks with Tabata format (rotating weekly), the WORKOUT block in força days with ESD cardio intervals, and append a fixed recovery block to every session type.

**Architecture:** All changes are inside `getSessionSections` in `index.html`. No new functions, no new localStorage keys, no signature changes. New constants are added at the top of the function body; the return statement is widened to append `RECOVERY_BLOCK`.

**Tech Stack:** Single-file React 18 PWA — JSX/JS inline, Babel CDN, no build step. All state in localStorage. Manual browser verification (no test framework).

---

## File Map

| File | Change |
|------|--------|
| `index.html` | All changes — inside `getSessionSections` only |

`getSessionSections` lives at **line 5073**. Key landmarks:
- Line 5097: end of `hikingWorkoutAdd` block — insert new constants **here** (before `const S = {`)
- Lines 5384–5389: CORE block L in `forca_day`
- Lines 5409–5414: CORE block M in `forca_day`
- Lines 5437–5443: CORE block H in `forca_day`
- Lines 5396–5400: WORKOUT block L in `forca_day`
- Lines 5423–5428: WORKOUT block M in `forca_day`
- Lines 5453–5459: WORKOUT block H in `forca_day`
- Lines 5470–5474: CORE block L in `endurance_day`
- Lines 5486–5491: CORE block M in `endurance_day`
- Lines 5503–5509: CORE block H in `endurance_day`
- Lines 5525–5528: CORE block L in `sailing_endurance`
- Lines 5541–5545: CORE block M in `sailing_endurance`
- Lines 5558–5562: CORE block H in `sailing_endurance`
- Line 5714: return statement — change to append `RECOVERY_BLOCK`

---

## Task 1: Add constants (weekIdx, TABATA_PAIRS, ESD, RECOVERY_BLOCK)

**Files:**
- Modify: `index.html` — after line 5097, before `const S = {`

- [ ] **Step 1: Insert the new constants**

Find this exact text in `index.html` (line 5097):

```js
        ] : [];

        const S = {
```

Replace with:

```js
        ] : [];

        const weekIdx = Math.floor(Date.now() / (7 * 24 * 3600 * 1000)) % 4;
        const TABATA_PAIRS = [
            ['Snow angel',  'Plank elbows'],
            ['Ab wheel',    'V-UP'],
            ['Hollow rock', 'V-UP'],
            ['Tuck-up',     'Hollow hold'],
        ];
        const [tabataA, tabataB] = TABATA_PAIRS[weekIdx];
        const ESD = {
            L: { label:'SHORT',     sets:4,  duration:'2–3 min',   rpe:'RPE 3–7'  },
            M: { label:'THRESHOLD', sets:7,  duration:'2–6 min',   rpe:'RPE 3–7'  },
            H: { label:'SPRINT',    sets:13, duration:'0:30–5 min', rpe:'RPE 2–10' },
        }[tier];
        const esdExercise = mode === 'bike' ? 'Bike' : 'Ergômetro (remo)';
        const RECOVERY_BLOCK = { title:'RECUPERAÇÃO', color:'#3db87a', items:[
            { name:'Alongamento Bruno Fontes', detail:'5 min', rest:'' }
        ]};

        const S = {
```

- [ ] **Step 2: Verify syntax in browser**

Open `index.html` in browser (or `python3 -m http.server 8000` then `localhost:8000`).
Open DevTools console.
Expected: **no errors**. If there's a SyntaxError, fix the insertion point.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add weekIdx, TABATA_PAIRS, ESD, RECOVERY_BLOCK constants to getSessionSections"
```

---

## Task 2: Replace CORE blocks in forca_day (all 3 tiers)

**Files:**
- Modify: `index.html` — three replacements inside `S.forca_day`

The current CORE blocks contain sets/reps exercises. Replace all three tiers with the Tabata format. The `hikingCoreAdd` spread must stay.

- [ ] **Step 1: Replace CORE block — tier L**

Find (lines ~5384–5389):

```js
                    { title:'CORE', color:'#9b7dd4', items:[
                        {name:'Prancha frontal',       detail:'2 × 20s',      rest:'30s'},
                        {name:'Dead bug',              detail:'2 × 8 rep',    rest:'30s'},
                        {name:'Prancha lateral',       detail:'1 × 20s cada', rest:'30s'},
                        ...hikingCoreAdd,
                    ]},
                    { title:'FORÇA', color:'#e87a45', items:[
```

Replace with:

```js
                    { title:'CORE · TABATA', color:'#9b7dd4', items:[
                        { name:`${tabataA} × ${tabataB}`, detail:'4 rounds · 0:20/0:10', rest:'1 min' },
                        ...hikingCoreAdd,
                    ]},
                    { title:'FORÇA', color:'#e87a45', items:[
```

- [ ] **Step 2: Replace CORE block — tier M**

Find (lines ~5409–5414):

```js
                    { title:'CORE', color:'#9b7dd4', items:[
                        {name:'Prancha frontal',       detail:'3 × 40s',      rest:'30s'},
                        {name:'Dead bug',              detail:'3 × 10 rep',   rest:'30s'},
                        {name:'Prancha lateral',       detail:'2 × 30s cada', rest:'30s'},
                        {name:'Hollow hold',           detail:'2 × 20s',      rest:'30s'},
                        ...hikingCoreAdd,
                    ]},
                    { title:'FORÇA', color:'#e87a45', items:[
```

Replace with:

```js
                    { title:'CORE · TABATA', color:'#9b7dd4', items:[
                        { name:`${tabataA} × ${tabataB}`, detail:'4 rounds · 0:20/0:10', rest:'1 min' },
                        ...hikingCoreAdd,
                    ]},
                    { title:'FORÇA', color:'#e87a45', items:[
```

- [ ] **Step 3: Replace CORE block — tier H**

Find (lines ~5437–5443):

```js
                    { title:'CORE', color:'#9b7dd4', items:[
                        {name:'Hollow hold',           detail:'4 × 30s',      rest:'30s'},
                        {name:'Dead bug',              detail:'4 × 12 rep',   rest:'30s'},
                        {name:'Prancha lateral',       detail:'3 × 45s cada', rest:'30s'},
                        {name:'Ab wheel rollout',      detail:'3 × 8 rep',    rest:'60s'},
                        {name:'Mountain climbers',     detail:'2 × 25 rep',   rest:'45s'},
                        ...hikingCoreAdd,
                    ]},
                    { title:'FORÇA', color:'#e87a45', items:[
```

Replace with:

```js
                    { title:'CORE · TABATA', color:'#9b7dd4', items:[
                        { name:`${tabataA} × ${tabataB}`, detail:'4 rounds · 0:20/0:10', rest:'1 min' },
                        ...hikingCoreAdd,
                    ]},
                    { title:'FORÇA', color:'#e87a45', items:[
```

- [ ] **Step 4: Verify in browser**

Navigate to a `forca_day` session in the app (plan tab → tap any força day → open session).
Expected: CORE block shows "CORE · TABATA" title, one item like "Snow angel × Plank elbows · 4 rounds · 0:20/0:10".
Expected: Console has no errors.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: replace CORE blocks in forca_day with Tabata format"
```

---

## Task 3: Replace WORKOUT blocks in forca_day with ESD (all 3 tiers)

**Files:**
- Modify: `index.html` — three replacements inside `S.forca_day`

- [ ] **Step 1: Replace WORKOUT block — tier L**

Find (lines ~5396–5400):

```js
                    { title:'WORKOUT', color:'#e87a45', items:[
                        {name:'Burpees',               detail:'2 × 5 rep',    rest:'60s'},
                        {name:'Polichinelo',           detail:'2 × 20 rep',   rest:'30s'},
                        ...hikingWorkoutAdd,
                    ]},
```

Replace with:

```js
                    { title:`ESD · ${ESD.label}`, color:'#e87a45', items:[
                        { name:esdExercise, detail:`${ESD.sets} × ${ESD.duration} · ${ESD.rpe}`, rest:'completo entre séries' },
                        ...hikingWorkoutAdd,
                    ]},
```

- [ ] **Step 2: Replace WORKOUT block — tier M**

Find (lines ~5423–5428):

```js
                    { title:'WORKOUT', color:'#e87a45', items:[
                        {name:'Burpees',               detail:'3 × 8 rep',    rest:'60s'},
                        {name:'Mountain climbers',     detail:'2 × 20 rep',   rest:'45s'},
                        {name:'Polichinelo',           detail:'2 × 25 rep',   rest:'30s'},
                        ...hikingWorkoutAdd,
                    ]},
```

Replace with:

```js
                    { title:`ESD · ${ESD.label}`, color:'#e87a45', items:[
                        { name:esdExercise, detail:`${ESD.sets} × ${ESD.duration} · ${ESD.rpe}`, rest:'completo entre séries' },
                        ...hikingWorkoutAdd,
                    ]},
```

- [ ] **Step 3: Replace WORKOUT block — tier H**

Find (lines ~5453–5459):

```js
                    { title:'WORKOUT', color:'#e87a45', items:[
                        {name:'Burpees',               detail:'4 × 10 rep',   rest:'60s'},
                        {name:'Mountain climbers',     detail:'3 × 25 rep',   rest:'45s'},
                        {name:'Polichinelo',           detail:'3 × 30 rep',   rest:'30s'},
                        {name:'Jump squat',            detail:'3 × 10 rep',   rest:'60s'},
                        ...hikingWorkoutAdd,
                    ]},
```

Replace with:

```js
                    { title:`ESD · ${ESD.label}`, color:'#e87a45', items:[
                        { name:esdExercise, detail:`${ESD.sets} × ${ESD.duration} · ${ESD.rpe}`, rest:'completo entre séries' },
                        ...hikingWorkoutAdd,
                    ]},
```

- [ ] **Step 4: Verify in browser**

Open a `forca_day` session. Scroll to the last active block (before any recovery).
Expected: Block title is "ESD · SHORT" (tier L), "ESD · THRESHOLD" (M), or "ESD · SPRINT" (H).
Expected: Item shows "Ergômetro (remo)" or "Bike" depending on athlete's cardioMode, with sets × duration · RPE.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: replace WORKOUT blocks in forca_day with ESD cardio intervals"
```

---

## Task 4: Replace CORE blocks in endurance_day (all 3 tiers)

**Files:**
- Modify: `index.html` — three replacements inside `S.endurance_day`

- [ ] **Step 1: Replace CORE block — tier L**

Find (lines ~5470–5474):

```js
                    { title:'CORE', color:'#9b7dd4', items:[
                        {name:'Prancha frontal',       detail:'2 × 20s',      rest:'30s'},
                        {name:'Dead bug',              detail:'2 × 8 rep',    rest:'30s'},
                        {name:'Burpees',               detail:'2 × 5 rep',    rest:'60s'},
                    ]},
                    { title:`ENDURANCE · ${mode.toUpperCase()} · Z2`, color:'#2d6b4f', items:[
```

Replace with:

```js
                    { title:'CORE · TABATA', color:'#9b7dd4', items:[
                        { name:`${tabataA} × ${tabataB}`, detail:'4 rounds · 0:20/0:10', rest:'1 min' },
                    ]},
                    { title:`ENDURANCE · ${mode.toUpperCase()} · Z2`, color:'#2d6b4f', items:[
```

- [ ] **Step 2: Replace CORE block — tier M**

Find (lines ~5486–5491):

```js
                    { title:'CORE', color:'#9b7dd4', items:[
                        {name:'Hollow hold',           detail:'3 × 20s',      rest:'30s'},
                        {name:'Dead bug',              detail:'3 × 10 rep',   rest:'30s'},
                        {name:'Prancha lateral',       detail:'2 × 30s cada', rest:'30s'},
                        {name:'Burpees',               detail:'3 × 8 rep',    rest:'60s'},
                    ]},
                    { title:`ENDURANCE · ${mode.toUpperCase()} · Z2`, color:'#2d6b4f', items:[
```

Replace with:

```js
                    { title:'CORE · TABATA', color:'#9b7dd4', items:[
                        { name:`${tabataA} × ${tabataB}`, detail:'4 rounds · 0:20/0:10', rest:'1 min' },
                    ]},
                    { title:`ENDURANCE · ${mode.toUpperCase()} · Z2`, color:'#2d6b4f', items:[
```

- [ ] **Step 3: Replace CORE block — tier H**

Find (lines ~5503–5509):

```js
                    { title:'CORE', color:'#9b7dd4', items:[
                        {name:'Hollow hold',           detail:'4 × 30s',      rest:'30s'},
                        {name:'Dead bug',              detail:'4 × 12 rep',   rest:'30s'},
                        {name:'Prancha lateral',       detail:'3 × 40s cada', rest:'30s'},
                        {name:'Burpees',               detail:'4 × 10 rep',   rest:'60s'},
                        {name:'Push-up',               detail:'3 × 12 rep',   rest:'45s'},
                    ]},
                    { title:`ENDURANCE · ${mode.toUpperCase()} · Z2–3`, color:'#2d6b4f', items:[
```

Replace with:

```js
                    { title:'CORE · TABATA', color:'#9b7dd4', items:[
                        { name:`${tabataA} × ${tabataB}`, detail:'4 rounds · 0:20/0:10', rest:'1 min' },
                    ]},
                    { title:`ENDURANCE · ${mode.toUpperCase()} · Z2–3`, color:'#2d6b4f', items:[
```

- [ ] **Step 4: Verify in browser**

Open an `endurance_day` session.
Expected: CORE block shows "CORE · TABATA" with the same pair as força day (same calendar week → same `weekIdx`).
Expected: ENDURANCE block below is unchanged (shows bike/remo Z2 cardio).

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: replace CORE blocks in endurance_day with Tabata format"
```

---

## Task 5: Replace CORE blocks in sailing_endurance (all 3 tiers)

**Files:**
- Modify: `index.html` — three replacements inside `S.sailing_endurance`

- [ ] **Step 1: Replace CORE block — tier L**

Find (lines ~5525–5528):

```js
                    { title:'CORE', color:'#9b7dd4', items:[
                        {name:'Prancha frontal',       detail:'2 × 20s',      rest:'30s'},
                        {name:'Dead bug',              detail:'2 × 8 rep',    rest:'30s'},
                    ]},
                    { title:'ENDURANCE ESPECÍFICO · VELA', color:'#4a8fd9', items:[
```

Replace with:

```js
                    { title:'CORE · TABATA', color:'#9b7dd4', items:[
                        { name:`${tabataA} × ${tabataB}`, detail:'4 rounds · 0:20/0:10', rest:'1 min' },
                    ]},
                    { title:'ENDURANCE ESPECÍFICO · VELA', color:'#4a8fd9', items:[
```

- [ ] **Step 2: Replace CORE block — tier M**

Find (lines ~5541–5545):

```js
                    { title:'CORE', color:'#9b7dd4', items:[
                        {name:'Hollow hold',           detail:'3 × 20s',      rest:'30s'},
                        {name:'Dead bug',              detail:'3 × 10 rep',   rest:'30s'},
                        {name:'Prancha lateral',       detail:'2 × 30s cada', rest:'30s'},
                    ]},
                    { title:'ENDURANCE ESPECÍFICO · VELA', color:'#4a8fd9', items:[
```

Replace with:

```js
                    { title:'CORE · TABATA', color:'#9b7dd4', items:[
                        { name:`${tabataA} × ${tabataB}`, detail:'4 rounds · 0:20/0:10', rest:'1 min' },
                    ]},
                    { title:'ENDURANCE ESPECÍFICO · VELA', color:'#4a8fd9', items:[
```

- [ ] **Step 3: Replace CORE block — tier H**

Find (lines ~5558–5562):

```js
                    { title:'CORE', color:'#9b7dd4', items:[
                        {name:'Hollow hold',           detail:'4 × 30s',      rest:'30s'},
                        {name:'Dead bug',              detail:'4 × 12 rep',   rest:'30s'},
                        {name:'Prancha lateral',       detail:'3 × 40s cada', rest:'30s'},
                    ]},
                    { title:'ENDURANCE ESPECÍFICO · VELA', color:'#4a8fd9', items:[
```

Replace with:

```js
                    { title:'CORE · TABATA', color:'#9b7dd4', items:[
                        { name:`${tabataA} × ${tabataB}`, detail:'4 rounds · 0:20/0:10', rest:'1 min' },
                    ]},
                    { title:'ENDURANCE ESPECÍFICO · VELA', color:'#4a8fd9', items:[
```

- [ ] **Step 4: Verify in browser**

Open a `sailing_endurance` session (requires hiking athlete profile).
Expected: CORE · TABATA block appears between MOBILIDADE and ENDURANCE ESPECÍFICO · VELA.
Expected: No console errors.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: replace CORE blocks in sailing_endurance with Tabata format"
```

---

## Task 6: Append RECOVERY_BLOCK to all session types

**Files:**
- Modify: `index.html` — return statement at line ~5714

- [ ] **Step 1: Replace the return statement**

Find (line ~5714):

```js
        const raw = (S[type] && S[type][tier]) || SESSION_TEMPLATES[type] || [];
        return raw.map(sec => ({ ...sec, items: equipFilter(sec.items) }));
```

Replace with:

```js
        const raw = (S[type] && S[type][tier]) || SESSION_TEMPLATES[type] || [];
        const sections = raw.map(sec => ({ ...sec, items: equipFilter(sec.items) }));
        return [...sections, RECOVERY_BLOCK];
```

- [ ] **Step 2: Verify in browser — força day**

Open any `forca_day` session and scroll to the bottom.
Expected: Last block is "RECUPERAÇÃO" with "Alongamento Bruno Fontes · 5 min".

- [ ] **Step 3: Verify in browser — endurance day**

Open any `endurance_day` session and scroll to the bottom.
Expected: Last block is "RECUPERAÇÃO" with "Alongamento Bruno Fontes · 5 min".

- [ ] **Step 4: Verify in browser — old session types**

Open an `endurance` or `hiking` session (older types, still used for non-plan sessions).
Expected: Last block is also "RECUPERAÇÃO".

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: append RECOVERY_BLOCK to all session types"
```

---

## Self-Review Checklist

- [x] **Spec coverage:**
  - Section 1 (Tabata CORE): Tasks 2, 4, 5 ✓
  - Section 2 (ESD WORKOUT): Task 3 ✓
  - Section 3 (RECOVERY_BLOCK): Task 6 ✓
  - Constants: Task 1 ✓
- [x] **No placeholders:** All code blocks are complete and literal.
- [x] **Type consistency:** `TABATA_PAIRS`, `tabataA`, `tabataB`, `ESD`, `esdExercise`, `RECOVERY_BLOCK` are defined in Task 1 and used consistently in Tasks 2–6.
- [x] **hikingCoreAdd** present in Task 2 (forca_day CORE), absent in Tasks 4–5 (endurance/sailing CORE) — matches existing pattern.
- [x] **hikingWorkoutAdd** present in Task 3 (forca_day WORKOUT) — matches existing pattern.
