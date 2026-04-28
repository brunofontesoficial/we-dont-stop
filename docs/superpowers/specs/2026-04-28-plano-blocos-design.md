# Plano por Blocos — Design

> **For agentic workers:** This spec describes changes to the training plan generation logic in a single-file React PWA (`index.html`). All logic lives in that file. There are no separate modules or build steps.

## Goal

Rebuild the weekly training plan scheduling logic (`getPlanWeeks`) to enforce strict Force/Endurance alternation, reshape session templates into 4 defined blocks with correct proportions, and integrate hiking as a modality for escora boats. The exercise library content is preserved; only structure and scheduling change.

## Architecture

- **Scheduling layer**: `getPlanWeeks()` rebuilt with F/E alternation + odd-day rule by cycle week
- **Session templates**: força and endurance day templates restructured into 4 named blocks with proportions matching coach spec
- **Hiking modifier**: `hasHiking` flag injects hiking exercises into Core, Força, and Workout blocks; adds specific endurance variant
- **No new localStorage keys, no UI changes**: rendering layer is untouched

## Tech Stack

React 18 (CDN + Babel Standalone), single `index.html`, all state in localStorage

---

## Section 1 — Session Composition

### Força Day (4 blocks)

| Block | Label | Proportion | Purpose |
|---|---|---|---|
| 1 | Mobilidade | 10% | Mobility warm-up |
| 2 | Core | 25% | Core stability |
| 3 | Força | 45% | Main strength work |
| 4 | Workout | 15% | Short intense finisher (HIIT) |

Remaining ~5% absorbed by transitions/cool-down at end of Workout block.

### Endurance Day (3 blocks)

| Block | Label | Proportion | Purpose |
|---|---|---|---|
| 1 | Mobilidade | 10% | Mobility warm-up |
| 2 | Core | 20% | Core stability |
| 3 | Endurance | 70% | Z2 cardio (or specific sailing variant) |

### 4-Week Variants

Both day types use 3 intensity tiers mapped to cycle weeks:

| Cycle Week | Tier | Characteristic |
|---|---|---|
| Semana 1 (Base) | L | Base volume, ~60–70% intensity |
| Semana 2 (Estabilidade) | M | Same load, technical focus |
| Semana 3 (Overload) | H | +1 série per exercise in Força and Workout blocks |
| Semana 4 (Deload) | L | Reduced volume (~50%), recovery |

Semanas 1 and 4 both use tier L but with different subtitles (week 4 explicitly labels "deload").

---

## Section 2 — Weekly Scheduling

### Alternation Rule

Strict Force/Endurance alternation. No consecutive days of the same type.

| Days/week | Semana 3 (Overload) | Semanas 1, 2, 4 |
|---|---|---|
| 1 | F | F |
| 2 | F · E | F · E |
| 3 | F · E · **F** | E · F · **E** |
| 4 | F · E · F · E | F · E · F · E |
| 5 | F · E · F · E · **F** | E · F · E · F · **E** |
| 6 | F · E · F · E · F · E | F · E · F · E · F · E |

Bold = the "extra" day awarded based on cycle week:
- Semana 3: Força gets the extra day (overload week prioritizes strength)
- Semanas 1, 2, 4: Endurance gets the extra day (endurance is most voluminous overall)

### Day Position Mapping

The existing day-of-week slots (`pat.train`) are preserved. For `diasSemana=4`, slots are Mon/Tue/Thu/Fri. The F/E pattern is applied in order to these slots, not to fixed weekdays.

### Rest and Active Recovery

Unchanged: `pat.ativo` days get `ativo` type, remaining days get `descanso`.

### `buildWeek(wi)` — New Logic

```js
function buildWeek(wi) {
    // wi: 0-3 (week index in the 4-week cycle)
    const isOverload = wi === 2; // Semana 3
    const dayCount = pat.train.length;
    // For odd counts: semana 3 starts with F, others start with E
    const startsWithForce = (dayCount % 2 === 0) ? true : isOverload;

    return Array.from({ length: 7 }, (_, dayIdx) => {
        const ti = pat.train.indexOf(dayIdx);
        if (ti >= 0) {
            const type = (ti % 2 === 0) === startsWithForce ? 'forca_day' : 'endurance_day';
            const tier = wi === 2 ? 'H' : 'L'; // Overload=H, Base/Stability/Deload=L
            // Semana 2 uses M tier
            const finalTier = wi === 1 ? 'M' : tier;
            return { type, ...DAY_TEMPLATES[type][finalTier] };
        }
        if (pat.ativo.includes(dayIdx)) return d('ativo', wi);
        return d('descanso', wi);
    });
}
```

---

## Section 3 — Força Day Templates

Each tier (L/M/H) defines 4 blocks in sequence. Content derived from existing força, core, and new workout finisher exercises.

### Tier L (Semanas 1 & 4)

```js
forca_day: {
  L: [
    { title: 'MOBILIDADE', color: '#e87a45', items: [
        { name: 'Abertura de quadril',  detail: '1 × 45s cada', rest: '' },
        { name: 'Cat-cow',              detail: '1 × 10 rep',   rest: '' },
        { name: 'Torção torácica',      detail: '1 × 8 rep',    rest: '' },
    ]},
    { title: 'CORE', color: '#9b7dd4', items: [
        { name: 'Prancha frontal',      detail: '2 × 20s',      rest: '30s' },
        { name: 'Dead bug',             detail: '2 × 8 rep',    rest: '30s' },
        { name: 'Prancha lateral',      detail: '1 × 20s cada', rest: '30s' },
    ]},
    { title: 'FORÇA', color: '#e87a45', items: [
        { name: 'Agachamento',          detail: '3 × 10 rep',   rest: '60s' },
        { name: 'Push-up (joelho)',      detail: '3 × 8 rep',    rest: '60s' },
        { name: 'Romanian deadlift',    detail: '2 × 10 rep',   rest: '75s' },
    ]},
    { title: 'WORKOUT', color: '#e87a45', items: [
        { name: 'Burpees',              detail: '2 × 5 rep',    rest: '60s' },
        { name: 'Polichinelo',          detail: '2 × 20 rep',   rest: '30s' },
    ]},
  ],
```

### Tier M (Semana 2)

```js
  M: [
    { title: 'MOBILIDADE', color: '#e87a45', items: [
        { name: 'Abertura de quadril',  detail: '2 × 45s cada', rest: '' },
        { name: 'Cat-cow',              detail: '2 × 10 rep',   rest: '' },
        { name: 'Torção torácica',      detail: '1 × 10 rep',   rest: '' },
        { name: 'Mobilidade de ombro',  detail: '1 × 10 rep',   rest: '' },
    ]},
    { title: 'CORE', color: '#9b7dd4', items: [
        { name: 'Prancha frontal',      detail: '3 × 40s',      rest: '30s' },
        { name: 'Dead bug',             detail: '3 × 10 rep',   rest: '30s' },
        { name: 'Prancha lateral',      detail: '2 × 30s cada', rest: '30s' },
        { name: 'Hollow hold',          detail: '2 × 20s',      rest: '30s' },
    ]},
    { title: 'FORÇA', color: '#e87a45', items: [
        { name: 'Agachamento + remada', detail: '3 × 12 rep',   rest: '90s' },
        { name: 'Push-up',              detail: '3 × 15 rep',   rest: '60s' },
        { name: 'Romanian deadlift',    detail: '3 × 10 rep',   rest: '90s' },
        { name: 'Inverted row',         detail: '3 × 10 rep',   rest: '75s' },
    ]},
    { title: 'WORKOUT', color: '#e87a45', items: [
        { name: 'Burpees',              detail: '3 × 8 rep',    rest: '60s' },
        { name: 'Mountain climbers',    detail: '2 × 20 rep',   rest: '45s' },
        { name: 'Polichinelo',          detail: '2 × 25 rep',   rest: '30s' },
    ]},
  ],
```

### Tier H (Semana 3 — Overload)

```js
  H: [
    { title: 'MOBILIDADE', color: '#e87a45', items: [
        { name: 'Corrida leve',         detail: '5 min',        rest: '' },
        { name: 'Abertura de quadril',  detail: '2 × 45s cada', rest: '' },
        { name: 'Mobilidade de ombro',  detail: '2 × 10 rep',   rest: '' },
        { name: 'Leg swing frontal',    detail: '2 × 10 cada',  rest: '' },
    ]},
    { title: 'CORE', color: '#9b7dd4', items: [
        { name: 'Hollow hold',          detail: '4 × 30s',      rest: '30s' },
        { name: 'Dead bug',             detail: '4 × 12 rep',   rest: '30s' },
        { name: 'Prancha lateral',      detail: '3 × 45s cada', rest: '30s' },
        { name: 'Ab wheel rollout',     detail: '3 × 8 rep',    rest: '60s' },
        { name: 'Mountain climbers',    detail: '2 × 25 rep',   rest: '45s' },
    ]},
    { title: 'FORÇA', color: '#e87a45', items: [
        { name: 'Agachamento + remada', detail: '4 × 12 rep',   rest: '75s' },
        { name: 'Push-up',              detail: '4 × 20 rep',   rest: '60s' },
        { name: 'Barra / inverted row', detail: '4 × 8 rep',    rest: '90s' },
        { name: 'Romanian deadlift',    detail: '4 × 10 rep',   rest: '90s' },
        { name: 'Farmer carry',         detail: '3 × 30m',      rest: '60s' },
    ]},
    { title: 'WORKOUT', color: '#e87a45', items: [
        { name: 'Burpees',              detail: '4 × 10 rep',   rest: '60s' },
        { name: 'Mountain climbers',    detail: '3 × 25 rep',   rest: '45s' },
        { name: 'Polichinelo',          detail: '3 × 30 rep',   rest: '30s' },
        { name: 'Jump squat',           detail: '3 × 10 rep',   rest: '60s' },
    ]},
  ],
}
```

---

## Section 4 — Endurance Day Templates

### Tier L (Semanas 1 & 4)

```js
endurance_day: {
  L: [
    { title: 'MOBILIDADE', color: '#3db87a', items: [
        { name: 'Abertura de quadril',  detail: '1 × 45s cada', rest: '' },
        { name: 'Leg swing frontal',    detail: '1 × 10 cada',  rest: '' },
        { name: 'Cat-cow',              detail: '1 × 10 rep',   rest: '' },
    ]},
    { title: 'CORE', color: '#9b7dd4', items: [
        { name: 'Prancha frontal',      detail: '2 × 20s',      rest: '30s' },
        { name: 'Dead bug',             detail: '2 × 8 rep',    rest: '30s' },
        { name: 'Burpees',              detail: '2 × 5 rep',    rest: '60s' },
    ]},
    { title: `ENDURANCE · ${mode} · Z2`, color: '#2d6b4f', items: [
        { name: cardioExercise,         detail: '20 min Z2',    rest: '' },
        { name: 'Recuperação ativa',    detail: '5 min',        rest: '' },
    ]},
  ],
```

### Tier M (Semana 2)

```js
  M: [
    { title: 'MOBILIDADE', color: '#3db87a', items: [
        { name: 'Abertura de quadril',  detail: '2 × 45s cada', rest: '' },
        { name: 'Leg swing frontal',    detail: '2 × 10 cada',  rest: '' },
        { name: 'Torção torácica',      detail: '1 × 10 rep',   rest: '' },
    ]},
    { title: 'CORE', color: '#9b7dd4', items: [
        { name: 'Hollow hold',          detail: '3 × 20s',      rest: '30s' },
        { name: 'Dead bug',             detail: '3 × 10 rep',   rest: '30s' },
        { name: 'Prancha lateral',      detail: '2 × 30s cada', rest: '30s' },
        { name: 'Burpees',              detail: '3 × 8 rep',    rest: '60s' },
    ]},
    { title: `ENDURANCE · ${mode} · Z2`, color: '#2d6b4f', items: [
        { name: cardioExercise,         detail: '30 min Z2',    rest: '' },
        { name: 'Variação de cadência', detail: '3 × 2 min',    rest: '60s' },
    ]},
  ],
```

### Tier H (Semana 3 — Overload)

```js
  H: [
    { title: 'MOBILIDADE', color: '#3db87a', items: [
        { name: 'Abertura de quadril',  detail: '2 × 60s cada', rest: '' },
        { name: 'Leg swing multi-plano',detail: '2 × 12 cada',  rest: '' },
        { name: `${cardioExercise} prog.`, detail: '8 min',     rest: '' },
    ]},
    { title: 'CORE', color: '#9b7dd4', items: [
        { name: 'Hollow hold',          detail: '4 × 30s',      rest: '30s' },
        { name: 'Dead bug',             detail: '4 × 12 rep',   rest: '30s' },
        { name: 'Prancha lateral',      detail: '3 × 40s cada', rest: '30s' },
        { name: 'Burpees',              detail: '4 × 10 rep',   rest: '60s' },
        { name: 'Push-up',              detail: '3 × 12 rep',   rest: '45s' },
    ]},
    { title: `ENDURANCE · ${mode} · Z2–3`, color: '#2d6b4f', items: [
        { name: cardioExercise,         detail: '35 min Z2',    rest: '' },
        { name: 'Intervalo Z3',         detail: '4 × 3 min',    rest: '90s' },
        { name: 'Variação de cadência', detail: '5 × 2 min',    rest: '45s' },
    ]},
  ],
}
```

---

## Section 5 — Hiking Modifier (Barcos de Escora)

When `hasHiking === true`, inject hiking exercises into the existing blocks. This is additive — existing exercises remain.

### Força Day — Hiking Additions

**Core block** (add after existing core exercises):
```js
{ name: 'Hiking isométrico centro',  detail: '2–4 × máximo', rest: '90s' },
{ name: 'Wall sit',                  detail: '2–3 × 60s',    rest: '60s' },
```

**Força block** (add at end):
```js
{ name: 'Hiking bench lastrado',     detail: '3 × máximo',   rest: '2 min' },
{ name: 'Step-up lastrado',          detail: '3 × 12 cada',  rest: '60s' },
```

**Workout block** (add at end):
```js
{ name: 'Hiking bench sprint',       detail: '4 × 30s',      rest: '45s' },
```

Volume scales with tier: L uses 2× sets, M uses 3×, H uses 4×.

### Endurance Day — Specific Sailing Variant

Every other endurance day (alternating by `ti` index) uses the specific sailing variant:

**Standard endurance** (generic): cardio Z2 as normal  
**Specific sailing endurance** (`ti % 2 === 1`):

```js
{ title: 'ENDURANCE ESPECÍFICO · VELA', color: '#4a8fd9', items: [
    { name: cardioExercise,               detail: '20–30 min Z2', rest: '' },
    { name: 'Hiking isométrico centro',   detail: '6 × 1 min',    rest: '45s' },
    { name: 'Hiking isométrico lateral',  detail: '4 × 45s cada', rest: '45s' },
]},
```

Duration of the hiking component scales with tier (L: 1 min/set, M: 90s/set, H: 2 min/set).

---

## Section 6 — Unchanged

- `DAY_NAMES` for `ativo` and `descanso` types — unchanged
- `pat` (train/ativo day slot arrays) — unchanged
- `sd()` duration scaling function — unchanged
- Age factor adjustments — unchanged
- `cardioExercise` and `cardioDetail` derivation — unchanged
- `META` array (week labels/descriptions) — unchanged
- All session rendering components (`TelaSessaoDetalhe`, `TelaSessionAtiva`) — unchanged
- localStorage schema — unchanged (sessions store the rendered exercise list)

### What is removed

- `pool` array and pool-building logic (score-based session type selection)
- `DAY_NAMES` entries for `forca`, `endurance`, `hiking`, `core`, `trapezio` — replaced by `DAY_TEMPLATES.forca_day` and `DAY_TEMPLATES.endurance_day`
- Old session types `hiking` (standalone day) and `core` (standalone day) — hiking becomes a modifier, core is embedded in all sessions

---

## Success Criteria

1. A 4-day week never shows two consecutive days of the same type (Força/Endurance)
2. A 3-day week in Semana 3 has 2 Força days + 1 Endurance day; in Semanas 1/2/4 has 1 Força + 2 Endurance
3. Each Força day shows 4 blocks: Mobilidade → Core → Força → Workout
4. Each Endurance day shows 3 blocks: Mobilidade → Core → Endurance
5. For `hasHiking=true`: hiking isometric exercises appear in Core and Força blocks; one in every two endurance days shows "ENDURANCE ESPECÍFICO · VELA"
6. All 4 cycle weeks render without error; Semana 4 (deload) uses L tier templates
7. No regression in session rendering (TelaSessionAtiva, TelaSessaoDetalhe)
8. Duration scaling (`sd()`) still applies to all exercise details
