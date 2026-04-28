# Plano por Blocos — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rebuild `getPlanWeeks` to enforce strict Força/Endurance alternation, restructure session templates into 4-block Força days and 3-block Endurance days, and inject hiking modifiers for escora boat athletes.

**Architecture:** Single edit target (`index.html`). `PLAN_BADGE` and `scoreMap` gain two new keys (`forca_day`, `endurance_day`, `sailing_endurance`). `getPlanWeeks` replaces its pool logic and `buildWeek` with F/E alternation using a new `DAY_TEMPLATES` constant. `getSessionSections` gains `forca_day`, `endurance_day`, and `sailing_endurance` entries in its `S` object, with hiking modifier injection from `aval`.

**Tech Stack:** React 18 (CDN + Babel Standalone), single `index.html`, all state in localStorage, no build step.

---

## File Map

| File | Changes |
|---|---|
| `index.html:4803–4811` | `PLAN_BADGE` — add `forca_day`, `endurance_day`, `sailing_endurance` |
| `index.html:5107` | `scoreMap` in `getSessionSections` — add same 3 keys |
| `index.html:4851–4894` | `DAY_NAMES` — trim to `ativo` + `descanso` only |
| `index.html:4895` | Add `DAY_TEMPLATES` constant after `d()` helper |
| `index.html:4897–4920` | Delete pool-building logic |
| `index.html:4951–4966` | Replace `buildWeek` with F/E alternation version |
| `index.html:5116` | Add `hasHiking` + hiking helper arrays before `const S` |
| `index.html:5393` | Insert `forca_day` after `forca` section in `S` |
| `index.html:after forca_day` | Insert `endurance_day` + `sailing_endurance` in `S` |

---

## Task 1: Add `forca_day`, `endurance_day`, `sailing_endurance` to `PLAN_BADGE` and `scoreMap`

**Files:**
- Modify: `index.html:4803–4811` (PLAN_BADGE)
- Modify: `index.html:5107` (scoreMap)

- [ ] **Step 1: Update `PLAN_BADGE`**

In `index.html`, find and replace the `PLAN_BADGE` block (lines 4803–4811):

Old:
```js
    const PLAN_BADGE = {
        hiking:    {label:'HIKING',     bg:'rgba(74,143,217,0.12)',  c:'#4a8fd9'},
        trapezio:  {label:'TRAPÉZIO',   bg:'rgba(74,143,217,0.12)',  c:'#4a8fd9'},
        forca:     {label:'FORÇA',      bg:'rgba(232,122,69,0.12)',  c:'#e87a45'},
        endurance: {label:'ENDURANCE',  bg:'rgba(61,184,122,0.12)',  c:'#2d6b4f'},
        core:      {label:'CORE',       bg:'rgba(155,125,212,0.12)', c:'#9b7dd4'},
        ativo:     {label:'ATIVO',      bg:'rgba(61,184,122,0.08)',  c:'#3db87a'},
        descanso:  {label:'DESCANSO',   bg:'rgba(24,25,31,0.06)',    c:'rgba(24,25,31,0.35)'},
    };
```

New:
```js
    const PLAN_BADGE = {
        hiking:          {label:'HIKING',          bg:'rgba(74,143,217,0.12)',  c:'#4a8fd9'},
        trapezio:        {label:'TRAPÉZIO',        bg:'rgba(74,143,217,0.12)',  c:'#4a8fd9'},
        forca:           {label:'FORÇA',           bg:'rgba(232,122,69,0.12)',  c:'#e87a45'},
        endurance:       {label:'ENDURANCE',       bg:'rgba(61,184,122,0.12)',  c:'#2d6b4f'},
        core:            {label:'CORE',            bg:'rgba(155,125,212,0.12)', c:'#9b7dd4'},
        forca_day:       {label:'FORÇA',           bg:'rgba(232,122,69,0.12)',  c:'#e87a45'},
        endurance_day:   {label:'ENDURANCE',       bg:'rgba(61,184,122,0.12)',  c:'#2d6b4f'},
        sailing_endurance:{label:'ENDURANCE VELA', bg:'rgba(74,143,217,0.12)', c:'#4a8fd9'},
        ativo:           {label:'ATIVO',           bg:'rgba(61,184,122,0.08)',  c:'#3db87a'},
        descanso:        {label:'DESCANSO',        bg:'rgba(24,25,31,0.06)',    c:'rgba(24,25,31,0.35)'},
    };
```

- [ ] **Step 2: Update `scoreMap` in `getSessionSections`**

Find line 5107:
```js
        const scoreMap = { endurance:'endurance', hiking:'hiking', trapezio:'forca', forca:'forca', core:'core', ativo:'mobilidade' };
```

Replace with:
```js
        const scoreMap = { endurance:'endurance', hiking:'hiking', trapezio:'forca', forca:'forca', core:'core', ativo:'mobilidade', forca_day:'forca', endurance_day:'endurance', sailing_endurance:'endurance' };
```

- [ ] **Step 3: Open `http://localhost:8000` in browser and navigate to Plano tab**

The plan should still render (using old logic). No visible change yet — this step only adds badge/score mappings for types that don't appear in the plan yet.

- [ ] **Step 4: Commit**

```bash
cd "/Users/brunofontes/Projects/App de treino - we dont stop"
git add index.html
git commit -m "feat: add forca_day/endurance_day/sailing_endurance to PLAN_BADGE and scoreMap"
```

---

## Task 2: Trim `DAY_NAMES` + add `DAY_TEMPLATES` in `getPlanWeeks`

**Files:**
- Modify: `index.html:4851–4895` (DAY_NAMES + d() helper)

- [ ] **Step 1: Replace `DAY_NAMES` block (trim to `ativo` + `descanso` only)**

Find the entire `DAY_NAMES` const (lines 4851–4894):

```js
        const DAY_NAMES = {
            endurance: [
                {name:C.base,        sub:C.sub,                              dur:sd(40), cardioMode},
                {name:C.base,        sub:'Ritmo estável · mesma carga',      dur:sd(40), cardioMode},
                {name:C.var,         sub:'Variação de ritmo',                dur:sd(50), cardioMode},
                {name:C.leve,        sub:'Ritmo confortável · Z1–Z2',        dur:sd(30), cardioMode},
            ],
            hiking: [
                {name:'Hiking bench + Core',   sub:'Isometria + prancha EMOM',   dur:sd(35)},
                {name:'Hiking + Hollow rocks', sub:'Qualidade > quantidade',      dur:sd(35)},
                {name:'Hiking + série extra',  sub:'+1 série isométrica',         dur:sd(40)},
                {name:'Técnica · vol. baixo',  sub:'Qualidade · sem fadiga',      dur:sd(20)},
            ],
            trapezio: [
                {name:'Trapézio + Ombro',  sub:'Força de braço e estabilidade',   dur:sd(35)},
                {name:'Trapézio + Core',   sub:'Rotação e estabilidade lateral',   dur:sd(35)},
                {name:'Trapézio + volume', sub:'+1 série · maior densidade',       dur:sd(40)},
                {name:'Técnica trapézio',  sub:'Qualidade · sem fadiga',            dur:sd(20)},
            ],
            forca: [
                {name:forcaName,     sub:'Agachamento · remada · push-up',   dur:sd(40)},
                {name:forcaName,     sub:'Técnica limpa · mesma carga',      dur:sd(40)},
                {name:forcaNamePlus, sub:'Volume ligeiramente aumentado',    dur:sd(45)},
                {name:'Circuito reduzido',     sub:'Metade do volume habitual',   dur:sd(25)},
            ],
            core: [
                {name:'Core funcional',        sub:'Hollow hold · dead bug · prancha', dur:sd(30)},
                {name:'Core estabilidade',     sub:'Foco em técnica e respiração',     dur:sd(30)},
                {name:'Core + volume',         sub:'Série extra · maior densidade',    dur:sd(30)},
                {name:'Core + mobilidade',     sub:'Suave · sem intensidade alta',     dur:sd(20)},
            ],
            ativo: [
                {name:'Mobilidade ativa',      sub:'Rolamento + alongamento',     dur:'20 min'},
                {name:'Mobilidade ativa',      sub:'Mobilidade de quadril',       dur:'20 min'},
                {name:'Mobilidade ativa',      sub:'Trabalho de ombro',          dur:'20 min'},
                {name:'Caminhada ativa',       sub:'Recuperação leve',           dur:'20 min'},
            ],
            descanso: [
                {name:'Descanso', sub:'', dur:''},
                {name:'Descanso', sub:'', dur:''},
                {name:'Descanso', sub:'', dur:''},
                {name:'Descanso', sub:'', dur:''},
            ],
        };
        function d(type, wi) { return { type, ...DAY_NAMES[type][Math.min(wi,3)] }; }
```

Replace with:

```js
        const DAY_NAMES = {
            ativo: [
                {name:'Mobilidade ativa',      sub:'Rolamento + alongamento',     dur:'20 min'},
                {name:'Mobilidade ativa',      sub:'Mobilidade de quadril',       dur:'20 min'},
                {name:'Mobilidade ativa',      sub:'Trabalho de ombro',          dur:'20 min'},
                {name:'Caminhada ativa',       sub:'Recuperação leve',           dur:'20 min'},
            ],
            descanso: [
                {name:'Descanso', sub:'', dur:''},
                {name:'Descanso', sub:'', dur:''},
                {name:'Descanso', sub:'', dur:''},
                {name:'Descanso', sub:'', dur:''},
            ],
        };
        function d(type, wi) { return { type, ...DAY_NAMES[type][Math.min(wi,3)] }; }

        const DAY_TEMPLATES = {
            forca_day: {
                L: { name: forcaName,      sub: 'Mobilidade · Core · Força · Workout',       dur: sd(60) },
                M: { name: forcaName,      sub: 'Técnica limpa · mesma intensidade',          dur: sd(60) },
                H: { name: forcaNamePlus,  sub: 'Sobrecarga · +volume em Força e Workout',    dur: sd(70) },
            },
            endurance_day: {
                L: { name: C.base,         sub: 'Mobilidade · Core · ' + C.sub,               dur: sd(50) },
                M: { name: C.base,         sub: 'Técnica e ritmo constante · Z2',             dur: sd(55) },
                H: { name: C.var,          sub: 'Volume aumentado · Z2–3',                    dur: sd(60) },
            },
            sailing_endurance: {
                L: { name: 'Endurance Específico · Vela',  sub: 'Z2 + hiking isométrico',      dur: sd(45) },
                M: { name: 'Endurance Específico · Vela',  sub: 'Z2 + hiking isométrico',      dur: sd(50) },
                H: { name: 'Endurance Específico · Vela',  sub: 'Z2–3 + hiking alto volume',   dur: sd(55) },
            },
        };
```

- [ ] **Step 2: Verify `d()` is still called only with `'ativo'` and `'descanso'`**

Search in `index.html` for `d('` inside `getPlanWeeks`. The only calls should be `d('ativo', wi)` and `d('descanso', wi)` (they exist in the `buildWeek` function we'll rewrite in Task 3, but the old `buildWeek` also calls them). Confirm no other type strings are passed.

- [ ] **Step 3: Open browser and verify Plano tab still loads without error**

At this point the old `buildWeek` is still using the pool and old `DAY_NAMES` entries (`forca`, `endurance`, etc.) — those are now removed. This will break the plan tab. That's expected — Task 3 replaces `buildWeek` and the pool immediately.

If you see errors in the browser console about `DAY_NAMES[type]` being undefined, that confirms the old entries were removed. Proceed to Task 3 immediately.

- [ ] **Step 4: Commit**

```bash
cd "/Users/brunofontes/Projects/App de treino - we dont stop"
git add index.html
git commit -m "refactor: trim DAY_NAMES to ativo/descanso + add DAY_TEMPLATES for forca_day/endurance_day"
```

---

## Task 3: Delete pool logic + replace `buildWeek` with F/E alternation

**Files:**
- Modify: `index.html:4897–4966` (pool logic + old buildWeek)

- [ ] **Step 1: Delete pool-building logic (lines 4897–4920)**

Find this block (starts immediately after the blank line after `d()` / `DAY_TEMPLATES`):

```js
        // Build candidate pool by sport + objetivo: weak areas first, emphasis applied
        const objetivo   = aval?.objetivo || null;
        const enduranceW = 2;
        const forcaW     = esporte === 'crossfit' ? 2
                         : objetivo === 'Performance' || objetivo === 'Competir' ? 2 : 1;

        const scoreOf = t => scores?.[t === 'hiking' ? 'hiking' : t] ?? 50;

        const rawTypes = [
            ...(hasHiking   ? ['hiking']   : []),
            ...(hasTrapezio ? ['trapezio'] : []),
            ...Array(enduranceW).fill('endurance'),
            ...Array(forcaW).fill('forca'),
            'core',
        ];

        // Dedupe, sort unique types by score (weakest first), then re-expand
        const unique = [...new Set(rawTypes)].sort((a,b) => scoreOf(a) - scoreOf(b));
        const candidates = unique.flatMap(t => Array(rawTypes.filter(x=>x===t).length).fill(t));

        const pool = [];
        for (let i = 0; pool.length < dias; i++) {
            pool.push(candidates[i % candidates.length]);
        }
```

Replace with nothing (delete entirely).

- [ ] **Step 2: Replace old `buildWeek` with new F/E alternation version**

Find old `buildWeek` (lines 4951–4966):

```js
        function buildWeek(wi) {
            const seen = {};
            return Array.from({length:7}, (_,dayIdx) => {
                const ti = pat.train.indexOf(dayIdx);
                if (ti >= 0) {
                    const type = pool[ti] || 'forca';
                    const occ = seen[type] || 0;
                    seen[type] = occ + 1;
                    // Offset by occ*2 so repeated types in the same week use different DAY_NAMES entries
                    const nameIdx = (wi + occ * 2) % 4;
                    return { type, ...DAY_NAMES[type][nameIdx] };
                }
                if (pat.ativo.includes(dayIdx)) return d('ativo', wi);
                return d('descanso', wi);
            });
        }
```

Replace with:

```js
        function buildWeek(wi) {
            const isOverload = wi === 2;
            const dayCount = pat.train.length;
            const startsWithForce = (dayCount % 2 === 0) ? true : isOverload;
            const tier = wi === 2 ? 'H' : wi === 1 ? 'M' : 'L';
            return Array.from({ length: 7 }, (_, dayIdx) => {
                const ti = pat.train.indexOf(dayIdx);
                if (ti >= 0) {
                    const isForce = (ti % 2 === 0) === startsWithForce;
                    let type = isForce ? 'forca_day' : 'endurance_day';
                    if (!isForce && hasHiking && ti % 2 === 1) type = 'sailing_endurance';
                    return { type, ...DAY_TEMPLATES[type][tier] };
                }
                if (pat.ativo.includes(dayIdx)) return d('ativo', wi);
                return d('descanso', wi);
            });
        }
```

- [ ] **Step 3: Verify in browser**

Open `http://localhost:8000`, go to Plano tab.

Expected for a 4-day athlete (Mon/Tue/Thu/Fri):
- Semana 1: F · E · F · E (badges: FORÇA, ENDURANCE, FORÇA, ENDURANCE)
- Semana 2: F · E · F · E
- Semana 3: F · E · F · E
- Semana 4: F · E · F · E

Expected for a 3-day athlete (Mon/Wed/Fri):
- Semana 3 (Overload): FORÇA · ENDURANCE · FORÇA
- Semanas 1, 2, 4: ENDURANCE · FORÇA · ENDURANCE

Verify no JavaScript console errors. Plan cards should display names from `DAY_TEMPLATES`.

- [ ] **Step 4: Commit**

```bash
cd "/Users/brunofontes/Projects/App de treino - we dont stop"
git add index.html
git commit -m "feat: replace pool logic with strict F/E alternation in buildWeek"
```

---

## Task 4: Add `forca_day` session sections to `getSessionSections`

**Files:**
- Modify: `index.html:5113–5117` (add hasHiking + hiking helpers before `const S`)
- Modify: `index.html:5393` (insert `forca_day` in `S` object after `forca` section)

- [ ] **Step 1: Add `hasHiking` and hiking helper arrays before `const S`**

Find these lines (5113–5117):

```js
        const cardioDetail  = { L:'15 min', M:'30 min', H:'35 min' };
        const intervDetail  = { L:'–',      M:'3 × 2 min', H:'5 × 2 min' };
        const burpeesDetail = { L:'3 × 5 rep', M:'3 × 8 rep', H:'4 × 10 rep' };

        const S = {
```

Replace with:

```js
        const cardioDetail  = { L:'15 min', M:'30 min', H:'35 min' };
        const intervDetail  = { L:'–',      M:'3 × 2 min', H:'5 × 2 min' };
        const burpeesDetail = { L:'3 × 5 rep', M:'3 × 8 rep', H:'4 × 10 rep' };

        const hasHiking = aval?.esporte === 'velejador' && aval?.tipoBarco === 'hiking';
        const hikingSets = { L:2, M:3, H:4 }[tier] || 2;
        const hikingCoreAdd = hasHiking ? [
            {name:'Hiking isométrico centro', detail:`${hikingSets} × máximo`, rest:'90s'},
            {name:'Wall sit',                 detail:`${hikingSets} × 60s`,    rest:'60s'},
        ] : [];
        const hikingForcaAdd = hasHiking ? [
            {name:'Hiking bench lastrado',  detail:`${hikingSets} × máximo`,    rest:'2 min'},
            {name:'Step-up lastrado',       detail:`${hikingSets} × 12 cada`,   rest:'60s'},
        ] : [];
        const hikingWorkoutAdd = hasHiking ? [
            {name:'Hiking bench sprint', detail:`${hikingSets} × 30s`, rest:'45s'},
        ] : [];

        const S = {
```

- [ ] **Step 2: Insert `forca_day` into the `S` object after the `forca` section**

Find the closing of the `forca` section (after line 5393):

```js
            },
            // ── CORE ───────────────────────────────────────────────────────
            core: {
```

Replace with:

```js
            },
            // ── FORÇA DAY (4 blocks: Mobilidade → Core → Força → Workout) ─
            forca_day: {
                L: [
                    { title:'MOBILIDADE', color:'#e87a45', items:[
                        {name:'Abertura de quadril',   detail:'1 × 45s cada', rest:''},
                        {name:'Cat-cow',               detail:'1 × 10 rep',   rest:''},
                        {name:'Torção torácica',       detail:'1 × 8 rep',    rest:''},
                    ]},
                    { title:'CORE', color:'#9b7dd4', items:[
                        {name:'Prancha frontal',       detail:'2 × 20s',      rest:'30s'},
                        {name:'Dead bug',              detail:'2 × 8 rep',    rest:'30s'},
                        {name:'Prancha lateral',       detail:'1 × 20s cada', rest:'30s'},
                        ...hikingCoreAdd,
                    ]},
                    { title:'FORÇA', color:'#e87a45', items:[
                        {name:'Agachamento',           detail:'3 × 10 rep',   rest:'60s'},
                        {name:'Push-up (joelho)',       detail:'3 × 8 rep',    rest:'60s'},
                        {name:'Romanian deadlift',     detail:'2 × 10 rep',   rest:'75s'},
                        ...hikingForcaAdd,
                    ]},
                    { title:'WORKOUT', color:'#e87a45', items:[
                        {name:'Burpees',               detail:'2 × 5 rep',    rest:'60s'},
                        {name:'Polichinelo',           detail:'2 × 20 rep',   rest:'30s'},
                        ...hikingWorkoutAdd,
                    ]},
                ],
                M: [
                    { title:'MOBILIDADE', color:'#e87a45', items:[
                        {name:'Abertura de quadril',   detail:'2 × 45s cada', rest:''},
                        {name:'Cat-cow',               detail:'2 × 10 rep',   rest:''},
                        {name:'Torção torácica',       detail:'1 × 10 rep',   rest:''},
                        {name:'Mobilidade de ombro',   detail:'1 × 10 rep',   rest:''},
                    ]},
                    { title:'CORE', color:'#9b7dd4', items:[
                        {name:'Prancha frontal',       detail:'3 × 40s',      rest:'30s'},
                        {name:'Dead bug',              detail:'3 × 10 rep',   rest:'30s'},
                        {name:'Prancha lateral',       detail:'2 × 30s cada', rest:'30s'},
                        {name:'Hollow hold',           detail:'2 × 20s',      rest:'30s'},
                        ...hikingCoreAdd,
                    ]},
                    { title:'FORÇA', color:'#e87a45', items:[
                        {name:'Agachamento + remada',  detail:'3 × 12 rep',   rest:'90s'},
                        {name:'Push-up',               detail:'3 × 15 rep',   rest:'60s'},
                        {name:'Romanian deadlift',     detail:'3 × 10 rep',   rest:'90s'},
                        {name:'Inverted row',          detail:'3 × 10 rep',   rest:'75s'},
                        ...hikingForcaAdd,
                    ]},
                    { title:'WORKOUT', color:'#e87a45', items:[
                        {name:'Burpees',               detail:'3 × 8 rep',    rest:'60s'},
                        {name:'Mountain climbers',     detail:'2 × 20 rep',   rest:'45s'},
                        {name:'Polichinelo',           detail:'2 × 25 rep',   rest:'30s'},
                        ...hikingWorkoutAdd,
                    ]},
                ],
                H: [
                    { title:'MOBILIDADE', color:'#e87a45', items:[
                        {name:'Corrida leve',          detail:'5 min',        rest:''},
                        {name:'Abertura de quadril',   detail:'2 × 45s cada', rest:''},
                        {name:'Mobilidade de ombro',   detail:'2 × 10 rep',   rest:''},
                        {name:'Leg swing frontal',     detail:'2 × 10 cada',  rest:''},
                    ]},
                    { title:'CORE', color:'#9b7dd4', items:[
                        {name:'Hollow hold',           detail:'4 × 30s',      rest:'30s'},
                        {name:'Dead bug',              detail:'4 × 12 rep',   rest:'30s'},
                        {name:'Prancha lateral',       detail:'3 × 45s cada', rest:'30s'},
                        {name:'Ab wheel rollout',      detail:'3 × 8 rep',    rest:'60s'},
                        {name:'Mountain climbers',     detail:'2 × 25 rep',   rest:'45s'},
                        ...hikingCoreAdd,
                    ]},
                    { title:'FORÇA', color:'#e87a45', items:[
                        {name:'Agachamento + remada',  detail:'4 × 12 rep',   rest:'75s'},
                        {name:'Push-up',               detail:'4 × 20 rep',   rest:'60s'},
                        {name:'Barra / inverted row',  detail:'4 × 8 rep',    rest:'90s'},
                        {name:'Romanian deadlift',     detail:'4 × 10 rep',   rest:'90s'},
                        {name:'Farmer carry',          detail:'3 × 30m',      rest:'60s'},
                        ...hikingForcaAdd,
                    ]},
                    { title:'WORKOUT', color:'#e87a45', items:[
                        {name:'Burpees',               detail:'4 × 10 rep',   rest:'60s'},
                        {name:'Mountain climbers',     detail:'3 × 25 rep',   rest:'45s'},
                        {name:'Polichinelo',           detail:'3 × 30 rep',   rest:'30s'},
                        {name:'Jump squat',            detail:'3 × 10 rep',   rest:'60s'},
                        ...hikingWorkoutAdd,
                    ]},
                ],
            },
            // ── CORE ───────────────────────────────────────────────────────
            core: {
```

- [ ] **Step 3: Verify in browser**

Open `http://localhost:8000`. Navigate to Plano tab, click on a FORÇA day. Verify:
- 4 blocks appear: MOBILIDADE · CORE · FORÇA · WORKOUT
- No JavaScript errors in console
- For a hiking athlete: hiking exercises appear in CORE, FORÇA, and WORKOUT blocks

- [ ] **Step 4: Commit**

```bash
cd "/Users/brunofontes/Projects/App de treino - we dont stop"
git add index.html
git commit -m "feat: add forca_day session sections with 4 blocks + hiking modifier"
```

---

## Task 5: Add `endurance_day` + `sailing_endurance` session sections

**Files:**
- Modify: `index.html` — insert `endurance_day` + `sailing_endurance` after `forca_day` in `S` object (before `core` section)

- [ ] **Step 1: Insert `endurance_day` and `sailing_endurance` in `S` object**

Find the comment that precedes `core` (just added in Task 4):

```js
            // ── CORE ───────────────────────────────────────────────────────
            core: {
```

Replace with:

```js
            // ── ENDURANCE DAY (3 blocks: Mobilidade → Core → Endurance) ───
            endurance_day: {
                L: [
                    { title:'MOBILIDADE', color:'#3db87a', items:[
                        {name:'Abertura de quadril',   detail:'1 × 45s cada', rest:''},
                        {name:'Leg swing frontal',     detail:'1 × 10 cada',  rest:''},
                        {name:'Cat-cow',               detail:'1 × 10 rep',   rest:''},
                    ]},
                    { title:'CORE', color:'#9b7dd4', items:[
                        {name:'Prancha frontal',       detail:'2 × 20s',      rest:'30s'},
                        {name:'Dead bug',              detail:'2 × 8 rep',    rest:'30s'},
                        {name:'Burpees',               detail:'2 × 5 rep',    rest:'60s'},
                    ]},
                    { title:`ENDURANCE · ${mode.toUpperCase()} · Z2`, color:'#2d6b4f', items:[
                        {name:cardioExercise,          detail:'20 min Z2',    rest:''},
                        {name:'Recuperação ativa',     detail:'5 min',        rest:''},
                    ]},
                ],
                M: [
                    { title:'MOBILIDADE', color:'#3db87a', items:[
                        {name:'Abertura de quadril',   detail:'2 × 45s cada', rest:''},
                        {name:'Leg swing frontal',     detail:'2 × 10 cada',  rest:''},
                        {name:'Torção torácica',       detail:'1 × 10 rep',   rest:''},
                    ]},
                    { title:'CORE', color:'#9b7dd4', items:[
                        {name:'Hollow hold',           detail:'3 × 20s',      rest:'30s'},
                        {name:'Dead bug',              detail:'3 × 10 rep',   rest:'30s'},
                        {name:'Prancha lateral',       detail:'2 × 30s cada', rest:'30s'},
                        {name:'Burpees',               detail:'3 × 8 rep',    rest:'60s'},
                    ]},
                    { title:`ENDURANCE · ${mode.toUpperCase()} · Z2`, color:'#2d6b4f', items:[
                        {name:cardioExercise,          detail:'30 min Z2',    rest:''},
                        {name:'Variação de cadência',  detail:'3 × 2 min',    rest:'60s'},
                    ]},
                ],
                H: [
                    { title:'MOBILIDADE', color:'#3db87a', items:[
                        {name:'Abertura de quadril',   detail:'2 × 60s cada', rest:''},
                        {name:'Leg swing multi-plano', detail:'2 × 12 cada',  rest:''},
                        {name:`${cardioExercise} prog.`, detail:'8 min',      rest:''},
                    ]},
                    { title:'CORE', color:'#9b7dd4', items:[
                        {name:'Hollow hold',           detail:'4 × 30s',      rest:'30s'},
                        {name:'Dead bug',              detail:'4 × 12 rep',   rest:'30s'},
                        {name:'Prancha lateral',       detail:'3 × 40s cada', rest:'30s'},
                        {name:'Burpees',               detail:'4 × 10 rep',   rest:'60s'},
                        {name:'Push-up',               detail:'3 × 12 rep',   rest:'45s'},
                    ]},
                    { title:`ENDURANCE · ${mode.toUpperCase()} · Z2–3`, color:'#2d6b4f', items:[
                        {name:cardioExercise,          detail:'35 min Z2',    rest:''},
                        {name:'Intervalo Z3',          detail:'4 × 3 min',    rest:'90s'},
                        {name:'Variação de cadência',  detail:'5 × 2 min',    rest:'45s'},
                    ]},
                ],
            },
            // ── SAILING ENDURANCE (specific vela variant for hiking boats) ─
            sailing_endurance: {
                L: [
                    { title:'MOBILIDADE', color:'#4a8fd9', items:[
                        {name:'Abertura de quadril',   detail:'1 × 45s cada', rest:''},
                        {name:'Leg swing frontal',     detail:'1 × 10 cada',  rest:''},
                        {name:'Cat-cow',               detail:'1 × 10 rep',   rest:''},
                    ]},
                    { title:'CORE', color:'#9b7dd4', items:[
                        {name:'Prancha frontal',       detail:'2 × 20s',      rest:'30s'},
                        {name:'Dead bug',              detail:'2 × 8 rep',    rest:'30s'},
                    ]},
                    { title:'ENDURANCE ESPECÍFICO · VELA', color:'#4a8fd9', items:[
                        {name:cardioExercise,                  detail:'20 min Z2',    rest:''},
                        {name:'Hiking isométrico centro',      detail:'6 × 1 min',    rest:'45s'},
                        {name:'Hiking isométrico lateral',     detail:'4 × 45s cada', rest:'45s'},
                    ]},
                ],
                M: [
                    { title:'MOBILIDADE', color:'#4a8fd9', items:[
                        {name:'Abertura de quadril',   detail:'2 × 45s cada', rest:''},
                        {name:'Leg swing frontal',     detail:'2 × 10 cada',  rest:''},
                        {name:'Torção torácica',       detail:'1 × 10 rep',   rest:''},
                    ]},
                    { title:'CORE', color:'#9b7dd4', items:[
                        {name:'Hollow hold',           detail:'3 × 20s',      rest:'30s'},
                        {name:'Dead bug',              detail:'3 × 10 rep',   rest:'30s'},
                        {name:'Prancha lateral',       detail:'2 × 30s cada', rest:'30s'},
                    ]},
                    { title:'ENDURANCE ESPECÍFICO · VELA', color:'#4a8fd9', items:[
                        {name:cardioExercise,                  detail:'25 min Z2',    rest:''},
                        {name:'Hiking isométrico centro',      detail:'6 × 90s',      rest:'45s'},
                        {name:'Hiking isométrico lateral',     detail:'4 × 60s cada', rest:'45s'},
                    ]},
                ],
                H: [
                    { title:'MOBILIDADE', color:'#4a8fd9', items:[
                        {name:'Abertura de quadril',   detail:'2 × 60s cada', rest:''},
                        {name:'Leg swing multi-plano', detail:'2 × 12 cada',  rest:''},
                        {name:`${cardioExercise} prog.`, detail:'8 min',      rest:''},
                    ]},
                    { title:'CORE', color:'#9b7dd4', items:[
                        {name:'Hollow hold',           detail:'4 × 30s',      rest:'30s'},
                        {name:'Dead bug',              detail:'4 × 12 rep',   rest:'30s'},
                        {name:'Prancha lateral',       detail:'3 × 40s cada', rest:'30s'},
                    ]},
                    { title:'ENDURANCE ESPECÍFICO · VELA', color:'#4a8fd9', items:[
                        {name:cardioExercise,                  detail:'30 min Z2',    rest:''},
                        {name:'Hiking isométrico centro',      detail:'6 × 2 min',    rest:'45s'},
                        {name:'Hiking isométrico lateral',     detail:'4 × 90s cada', rest:'45s'},
                    ]},
                ],
            },
            // ── CORE ───────────────────────────────────────────────────────
            core: {
```

- [ ] **Step 2: Add `Hiking bench lastrado` to `equipFilter` substitutions**

Find the `equipFilter` function's `subs` object (around line 5525):

```js
            const subs = {
                'Farmer carry':         {name:'Agachamento sumô',      detail:'3 × 30s',      rest:'60s'},
                'Barra / inverted row': {name:'Inverted row (mesa)',   detail:null,            rest:null},
                'Step-up lastrado':     {name:'Step-up',               detail:null,            rest:null},
                'Ab wheel rollout':     {name:'Mountain climbers',     detail:'3 × 20 rep',    rest:'60s'},
            };
```

Replace with:

```js
            const subs = {
                'Farmer carry':           {name:'Agachamento sumô',      detail:'3 × 30s',      rest:'60s'},
                'Barra / inverted row':   {name:'Inverted row (mesa)',   detail:null,            rest:null},
                'Step-up lastrado':       {name:'Step-up',               detail:null,            rest:null},
                'Hiking bench lastrado':  {name:'Hiking bench',          detail:null,            rest:null},
                'Ab wheel rollout':       {name:'Mountain climbers',     detail:'3 × 20 rep',    rest:'60s'},
            };
```

- [ ] **Step 3: Verify in browser — full golden path**

1. Open `http://localhost:8000`
2. Navigate to Plano tab — verify all 4 weeks show FORÇA/ENDURANCE badges with correct alternation
3. Click a FORÇA day → verify 4 blocks: MOBILIDADE · CORE · FORÇA · WORKOUT
4. Click an ENDURANCE day → verify 3 blocks: MOBILIDADE · CORE · ENDURANCE
5. Check Semana 2 cards show "Técnica limpa · mesma intensidade" subtitle
6. Check Semana 3 cards show "Sobrecarga · +volume em Força e Workout" subtitle
7. Check no JavaScript console errors
8. If testing a hiking athlete: tap an ENDURANCE day on odd training slot → should show "ENDURANCE ESPECÍFICO · VELA" block

- [ ] **Step 4: Commit**

```bash
cd "/Users/brunofontes/Projects/App de treino - we dont stop"
git add index.html
git commit -m "feat: add endurance_day and sailing_endurance session sections"
```

---

## Success Criteria (from spec)

1. A 4-day week never shows two consecutive days of the same type
2. A 3-day week in Semana 3 has 2 Força days + 1 Endurance; Semanas 1/2/4 have 1 Força + 2 Endurance
3. Each Força day shows 4 blocks: Mobilidade → Core → Força → Workout
4. Each Endurance day shows 3 blocks: Mobilidade → Core → Endurance
5. For `hasHiking=true`: hiking exercises appear in Core and Força blocks; one in every two endurance days shows "ENDURANCE ESPECÍFICO · VELA"
6. All 4 cycle weeks render without error; Semana 4 (deload) uses L tier templates
7. No regression in session rendering (TelaSessionAtiva, TelaSessaoDetalhe)
8. Duration scaling (`sd()`) still applies to all exercise details
