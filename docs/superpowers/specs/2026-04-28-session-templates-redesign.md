# Session Templates Redesign — Design Spec

> **For agentic workers:** Single-file React PWA (`index.html`). All logic lives in that file. No build step. All changes are inside `getSessionSections`.

## Goal

Redesign the exercise content of `forca_day`, `endurance_day`, and `sailing_endurance` session types to match Bruno Fontes' real training methodology:
1. Core block → Tabata format (rotating pairs per 4-week cycle)
2. Workout block on força days → ESD cardio intervals (rower/bike, tier-based)
3. Recovery block → fixed "Alongamento Bruno Fontes · 5 min" appended to **all** session types

## Architecture

All changes are inside `getSessionSections(type, aval, cardioMode)` in `index.html`. No new functions, no new localStorage keys, no changes to the function signature or call sites.

Three additions at the top of the function body:
- `weekIdx` + `TABATA_PAIRS` — week-based pair selection
- `ESD` lookup — tier-based ESD block config
- `RECOVERY_BLOCK` constant — appended at return

---

## Section 1 — Core Tabata block

### What changes

The CORE block in `forca_day`, `endurance_day`, and `sailing_endurance` (all tiers: L, M, H) replaces the current sets/reps exercises with a single Tabata item. The pair rotates every calendar week, cycling through 4 pairs (one per week of the 4-week training block).

### Implementation

Add at the top of `getSessionSections`, after existing variable declarations:

```js
const weekIdx = Math.floor(Date.now() / (7 * 24 * 3600 * 1000)) % 4;
const TABATA_PAIRS = [
    ['Snow angel',  'Plank elbows'],  // week 0
    ['Ab wheel',    'V-UP'],           // week 1
    ['Hollow rock', 'V-UP'],           // week 2
    ['Tuck-up',     'Hollow hold'],    // week 3
];
const [tabataA, tabataB] = TABATA_PAIRS[weekIdx];
```

The CORE block in all three types and all three tiers becomes:

```js
{ title: 'CORE · TABATA', color: '#9b7dd4', items: [
    { name: `${tabataA} × ${tabataB}`, detail: '4 rounds · 0:20/0:10', rest: '1 min' },
    ...hikingCoreAdd,   // only present in forca_day (existing variable)
]}
```

`hikingCoreAdd` is already defined earlier in the function and is an empty array for non-hiking athletes — no change needed there.

### Backward compatibility

`weekIdx` is deterministic from `Date.now()` — same result for all renders within the same calendar week. No localStorage needed.

---

## Section 2 — ESD block (força days only)

### What changes

The WORKOUT block in `forca_day` (all tiers) is replaced with an ESD cardio interval block. The format (Short/Threshold/Sprint) scales with tier. `endurance_day` and `sailing_endurance` are **not affected** — their cardio block remains unchanged.

### Implementation

Add after the `ESD` lookup, at the top of `getSessionSections`:

```js
const ESD = {
    L: { label: 'SHORT',     sets: 4,  duration: '2–3 min',   rpe: 'RPE 3–7'  },
    M: { label: 'THRESHOLD', sets: 7,  duration: '2–6 min',   rpe: 'RPE 3–7'  },
    H: { label: 'SPRINT',    sets: 13, duration: '0:30–5 min', rpe: 'RPE 2–10' },
}[tier];
const esdExercise = mode === 'bike' ? 'Bike' : 'Ergômetro (remo)';
```

The WORKOUT block in all three tiers of `forca_day` becomes:

```js
{ title: `ESD · ${ESD.label}`, color: '#e87a45', items: [
    { name: esdExercise, detail: `${ESD.sets} × ${ESD.duration} · ${ESD.rpe}`, rest: 'completo entre séries' },
    ...hikingWorkoutAdd,  // existing variable, empty for non-hiking athletes
]}
```

---

## Section 3 — Recovery block (all sessions)

### What changes

A fixed recovery block is appended to **every** session type returned by `getSessionSections`. This includes: `forca_day`, `endurance_day`, `sailing_endurance`, `endurance`, `hiking`, `trapezio`, `core`, `forca`, `ativo`.

### Implementation

Define the constant after the other new variables:

```js
const RECOVERY_BLOCK = { title: 'RECUPERAÇÃO', color: '#3db87a', items: [
    { name: 'Alongamento Bruno Fontes', detail: '5 min', rest: '' }
]};
```

Change the return statement from:

```js
return S[type]?.[tier] || S.endurance[tier];
```

to:

```js
const sections = S[type]?.[tier] || S.endurance[tier] || [];
return [...sections, RECOVERY_BLOCK];
```

---

## Files

- **Modify:** `index.html` — all changes inside `getSessionSections`

## What does NOT change

- Function signature: `getSessionSections(type, aval, cardioMode)` — unchanged
- All call sites remain the same
- `endurance_day` and `sailing_endurance` cardio/endurance blocks — unchanged
- `hikingCoreAdd`, `hikingForcaAdd`, `hikingWorkoutAdd` variables — unchanged
- The warmup (MOBILIDADE) blocks — unchanged
- No new localStorage keys

## Success criteria

1. All session types end with "Alongamento Bruno Fontes · 5 min"
2. CORE block in `forca_day`, `endurance_day`, `sailing_endurance` shows a Tabata pair
3. Tabata pair changes each calendar week, cycling through 4 pairs
4. WORKOUT block in `forca_day` shows ESD cardio intervals (Bike or Ergômetro)
5. ESD format matches tier: L=Short (4×2–3min), M=Threshold (7×2–6min), H=Sprint (13×0:30–5min)
6. No regression in other session types
