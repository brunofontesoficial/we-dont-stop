# Personalized Training Targets — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make exercise prescriptions adapt to each athlete's real test results, and show the weakest training pillar as the focus of the week.

**Architecture:** Add one pure function `getPersonalizedTargets(aval)` that computes all targets from stored test data. Integrate it into `getSessionSections` via a post-processing `applyTargets` helper that rewrites exercise detail strings by name. Add `weakPillar` to the plan week metadata and render a focus banner in `TelaPlano`.

**Tech Stack:** React 18 (via CDN, no build step), Babel standalone, vanilla JS, single `index.html` file (~5943 lines). All changes go in `index.html`. No external dependencies. Verification by browser console assertions.

---

## File Map

| File | Change |
|------|--------|
| `index.html:808` | Insert `getPersonalizedTargets(aval)` after `getAvaliacao()` |
| `index.html:534` | Insert `.plano-focus-banner` CSS near other `.plano-*` styles |
| `index.html:5078` | Add 4 helper vars at top of `getSessionSections` |
| `index.html:5081` | Update `cardioDetail` to include pace |
| `index.html:5706` | Insert `applyTargets` function + wrap usage |
| `index.html:4942` | Add `weakPillar` to each week in `getPlanWeeks` |
| `index.html:5820` | Add focus banner JSX in `TelaPlano` |

---

## Task 1: Add `getPersonalizedTargets(aval)` function

**Files:**
- Modify: `index.html:803-807` (insert after `getAvaliacao`)

- [ ] **Step 1: Insert the function**

Find this block at line 803:
```js
    function getAvaliacao() {
        try {
            return JSON.parse(localStorage.getItem('regata_avaliacao') || 'null');
        } catch { return null; }
    }
```

Insert immediately after it (before `function getAvaliacaoHistory`):

```js
    function getPersonalizedTargets(aval) {
        if (!aval) return { pushTarget:null, plankTarget:null, hikingTarget:null, endurancePace:null, weakPillar:null };

        // Push-up: 60% of max, rounded to nearest 5, min 3
        const pushTarget = aval.pushReps != null
            ? Math.max(3, Math.round(aval.pushReps * 0.60 / 5) * 5)
            : null;

        // Plank: 65% of max, rounded to nearest 5, min 10
        const plankTarget = aval.plankSecs != null
            ? Math.max(10, Math.round(aval.plankSecs * 0.65 / 5) * 5)
            : null;

        // Hiking bench: 70% of max, rounded to nearest 5, min 10
        const hikingTarget = aval.hikingSecs != null
            ? Math.max(10, Math.round(aval.hikingSecs * 0.70 / 5) * 5)
            : null;

        // Endurance pace by protocol
        let endurancePace = null;
        if (aval.endType && aval.endDist != null) {
            if (aval.endType === 'cooper') {
                const dist = parseFloat(aval.endDist) || 0;
                if (dist > 0) {
                    const secPerKm = (12 * 60 / dist) * 1000 * 1.30;
                    const m = Math.floor(secPerKm / 60);
                    const s = Math.round(secPerKm % 60).toString().padStart(2, '0');
                    endurancePace = `${m}:${s}/km`;
                }
            } else if (aval.endType === 'remo') {
                const parts = String(aval.endDist).split(':');
                if (parts.length === 2) {
                    const totalSec = parseInt(parts[0]) * 60 + parseInt(parts[1]);
                    const per500 = (totalSec / 4) * 1.15;
                    const m = Math.floor(per500 / 60);
                    const s = Math.round(per500 % 60).toString().padStart(2, '0');
                    endurancePace = `${m}:${s}/500m`;
                }
            } else if (aval.endType === 'bike') {
                const ftp = parseFloat(aval.endDist) || 0;
                if (ftp > 0) endurancePace = `${(ftp * 0.65).toFixed(1)} w/kg`;
            } else if (aval.endType === 'burpees') {
                const parts = String(aval.endDist).split(':');
                if (parts.length === 2) {
                    const totalMin = parseInt(parts[0]) + parseInt(parts[1]) / 60;
                    if (totalMin > 0) endurancePace = `${Math.round((100 / totalMin) * 0.65)} burpees/min`;
                }
            }
        }

        // Weak pillar: lowest score, sailing athletes include hiking
        let weakPillar = null;
        if (aval.scores) {
            const isVelejador = aval.esporte === 'velejador';
            const pillars = [
                ...(isVelejador ? [{k:'hiking', n:'Hiking Bench'}] : []),
                {k:'core',       n:'Core'},
                {k:'forca',      n:'Força'},
                {k:'endurance',  n:'Endurance'},
                {k:'mobilidade', n:'Mobilidade'},
            ];
            let minScore = Infinity;
            for (const p of pillars) {
                const sc = aval.scores[p.k] ?? Infinity;
                if (sc < minScore) { minScore = sc; weakPillar = p.n; }
            }
        }

        return { pushTarget, plankTarget, hikingTarget, endurancePace, weakPillar };
    }
```

- [ ] **Step 2: Verify in browser console**

Open `index.html` in browser (or the live GitHub Pages URL), open DevTools console, paste:

```js
// Test 1: null input → all nulls
const r1 = getPersonalizedTargets(null);
console.assert(r1.pushTarget === null, 'T1 pushTarget');
console.assert(r1.plankTarget === null, 'T1 plankTarget');
console.assert(r1.weakPillar === null, 'T1 weakPillar');
console.log('T1:', JSON.stringify(r1));

// Test 2: full sailor profile
const r2 = getPersonalizedTargets({
    pushReps:25, plankSecs:45, hikingSecs:60,
    endType:'cooper', endDist:2400,
    esporte:'velejador',
    scores:{ hiking:60, core:28, forca:55, endurance:70, mobilidade:50 }
});
console.assert(r2.pushTarget === 15, 'T2 push 15');
console.assert(r2.plankTarget === 30, 'T2 plank 30');
console.assert(r2.hikingTarget === 40, 'T2 hiking 40');
console.assert(r2.endurancePace === '6:30/km', 'T2 pace 6:30/km');
console.assert(r2.weakPillar === 'Core', 'T2 weakPillar Core');
console.log('T2:', JSON.stringify(r2));

// Test 3: beginner runner, no plank/hiking tests
const r3 = getPersonalizedTargets({
    pushReps:10, plankSecs:null, hikingSecs:null,
    endType:'bike', endDist:'3.8',
    esporte:'corredor',
    scores:{ core:50, forca:55, endurance:40, mobilidade:60 }
});
console.assert(r3.pushTarget === 5, 'T3 push 5');
console.assert(r3.plankTarget === null, 'T3 plank null');
console.assert(r3.endurancePace === '2.5 w/kg', 'T3 bike pace');
console.assert(r3.weakPillar === 'Endurance', 'T3 weakPillar Endurance');
console.log('T3:', JSON.stringify(r3));

console.log('✅ All assertions above with no errors = PASS');
```

Expected: three `console.log` lines with the result objects, no assertion errors.

- [ ] **Step 3: Commit**

```bash
cd /Users/brunofontes/projetos/we-dont-stop
git add index.html
git commit -m "feat: add getPersonalizedTargets — computes exercise targets from test data"
```

---

## Task 2: Integrate targets into `getSessionSections`

**Files:**
- Modify: `index.html:5078-5082` (add helpers after tier, update cardioDetail)
- Modify: `index.html:5706-5708` (insert applyTargets + use it)

- [ ] **Step 1: Add helper variables at top of `getSessionSections`**

Find this line at ~5078:
```js
        const tier = score < 35 ? 'L' : score < 65 ? 'M' : 'H';
        const mode = cardioMode || 'corrida';
```

Insert between the two lines (after tier, before mode):
```js
        const tier = score < 35 ? 'L' : score < 65 ? 'M' : 'H';
        const { pushTarget, plankTarget, hikingTarget, endurancePace } = getPersonalizedTargets(aval);
        const cPace = endurancePace ? ` · ~${endurancePace}` : '';
        const mode = cardioMode || 'corrida';
```

- [ ] **Step 2: Update `cardioDetail` to include pace**

Find at ~5081:
```js
        const cardioDetail  = { L:'15 min', M:'30 min', H:'35 min' };
```

Replace with:
```js
        const cardioDetail  = { L:`15 min${cPace}`, M:`30 min${cPace}`, H:`35 min${cPace}` };
```

- [ ] **Step 3: Add `applyTargets` function and use it**

Find these lines at ~5706-5708:
```js
        const raw = (S[type] && S[type][tier]) || SESSION_TEMPLATES[type] || [];
        const sections = raw.map(sec => ({ ...sec, items: equipFilter(sec.items) }));
        return [...sections, RECOVERY_BLOCK];
```

Replace with:
```js
        function applyTargets(secs) {
            return secs.map(sec => ({
                ...sec,
                items: sec.items.map(item => {
                    let detail = item.detail;
                    if (pushTarget && (item.name === 'Push-up' || item.name === 'Push-up (joelho)')) {
                        detail = detail.replace(/\d+ rep$/, `${pushTarget} rep`);
                    } else if (plankTarget && (item.name === 'Prancha frontal' || item.name === 'Hollow hold')) {
                        detail = detail.replace(/\d+s$/, `${plankTarget}s`);
                    } else if (hikingTarget && (item.name === 'Hiking bench isométrico' || item.name === 'Hiking isométrico centro' || item.name === 'Hiking bench lastrado')) {
                        detail = detail.replace('máximo', `${hikingTarget}s`);
                    } else if (endurancePace && (item.name === 'Corrida Z2' || item.name === 'Bike Z2' || item.name === 'Ergômetro Z2')) {
                        if (!detail.includes('~')) detail += ` · ~${endurancePace}`;
                    }
                    return { ...item, detail };
                })
            }));
        }
        const raw = (S[type] && S[type][tier]) || SESSION_TEMPLATES[type] || [];
        const sections = applyTargets(raw).map(sec => ({ ...sec, items: equipFilter(sec.items) }));
        return [...sections, RECOVERY_BLOCK];
```

- [ ] **Step 4: Verify in browser**

With the app open, navigate to **Plano → pick any training day → tap the session**. In DevTools console, run:

```js
// Simulate an athlete with known test values
const testAval = {
    pushReps:25, plankSecs:45, hikingSecs:60,
    endType:'cooper', endDist:2400,
    esporte:'velejador', tipoBarco:'hiking',
    scores:{ hiking:60, core:28, forca:55, endurance:70, mobilidade:50 },
    equipamentos:[]
};
const sections = getSessionSections('forca_day', testAval, 'corrida');
const pushItem = sections.flatMap(s => s.items).find(i => i.name === 'Push-up');
const plankItem = sections.flatMap(s => s.items).find(i => i.name === 'Prancha frontal');
console.assert(pushItem?.detail.includes('15'), 'Push-up shows 15 reps');
console.assert(plankItem?.detail.includes('30'), 'Plank shows 30s');
console.log('push:', pushItem?.detail, '| plank:', plankItem?.detail);

const endSections = getSessionSections('endurance_day', testAval, 'corrida');
const cardioItem = endSections.flatMap(s => s.items).find(i => i.name === 'Corrida Z2');
console.assert(cardioItem?.detail.includes('6:30'), 'Cardio shows 6:30/km');
console.log('cardio:', cardioItem?.detail);
```

Expected output:
```
push: 3 × 15 rep | plank: 3 × 30s
cardio: 30 min · ~6:30/km
```

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: applyTargets personalizes exercise details from test results"
```

---

## Task 3: Add `weakPillar` to `getPlanWeeks`

**Files:**
- Modify: `index.html:4936-4942` (META array in getPlanWeeks)

- [ ] **Step 1: Add weakPillar to each week**

Find at ~4815:
```js
    function getPlanWeeks(aval) {
```

And find the return statement at ~4936-4942:
```js
        const META = [
            {id:1, tag:'SEMANA 1 DE 4', name:'Base & padrão', desc:'Volume base · 60–70% · postura e respiração'},
            {id:2, tag:'SEMANA 2 DE 4', name:'Estabilidade',  desc:'Mesma carga · foco total em técnica e qualidade'},
            {id:3, tag:'SEMANA 3 DE 4', name:'Overload leve', desc:'+1 série por exercício · progressão de volume'},
            {id:4, tag:'SEMANA 4 DE 4', name:'Deload',        desc:'Volume reduzido · absorção e prep para o próximo ciclo'},
        ];
        return META.map((m, wi) => ({ ...m, days: buildWeek(wi) }));
```

Replace with:
```js
        const { weakPillar } = getPersonalizedTargets(aval);
        const META = [
            {id:1, tag:'SEMANA 1 DE 4', name:'Base & padrão', desc:'Volume base · 60–70% · postura e respiração'},
            {id:2, tag:'SEMANA 2 DE 4', name:'Estabilidade',  desc:'Mesma carga · foco total em técnica e qualidade'},
            {id:3, tag:'SEMANA 3 DE 4', name:'Overload leve', desc:'+1 série por exercício · progressão de volume'},
            {id:4, tag:'SEMANA 4 DE 4', name:'Deload',        desc:'Volume reduzido · absorção e prep para o próximo ciclo'},
        ];
        return META.map((m, wi) => ({ ...m, days: buildWeek(wi), weakPillar }));
```

- [ ] **Step 2: Verify in browser console**

```js
const aval = JSON.parse(localStorage.getItem('regata_avaliacao') || 'null');
const weeks = getPlanWeeks(aval);
console.log('weakPillar on week 1:', weeks[0].weakPillar);
// If no aval: null. If aval has scores: name of weakest pillar.
```

Expected: a pillar name string (e.g., `"Core"`) or `null` if no evaluation done.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add weakPillar to plan weeks from getPersonalizedTargets"
```

---

## Task 4: Add focus banner CSS and JSX

**Files:**
- Modify: `index.html:555` (add CSS after last .plano-* rule)
- Modify: `index.html:5820` (add banner JSX in TelaPlano)

- [ ] **Step 1: Add CSS**

Find at ~line 555:
```css
        .plano-day-dur { font-size: 11px; color: rgba(24,25,31,0.4); font-weight: 600; flex-shrink: 0; }
```

Insert immediately after:
```css
        .plano-focus-banner { margin: 0 20px 12px; background: #fff; border: 1px solid rgba(24,25,31,0.08); border-radius: 16px; padding: 14px 16px; display: flex; align-items: center; gap: 12px; }
        .plano-focus-ol { font-size: 9px; font-weight: 700; letter-spacing: 1.6px; text-transform: uppercase; color: var(--orange); margin-bottom: 3px; }
        .plano-focus-name { font-family: var(--font-display); font-size: 20px; font-weight: 600; color: var(--text-dark); line-height: 1.1; }
        .plano-focus-sub { font-size: 11px; color: var(--muted-dark); margin-top: 2px; }
```

- [ ] **Step 2: Add banner JSX in TelaPlano**

Find at ~line 5820 in `TelaPlano`:
```jsx
                <div className="plano-summary">
                    <div className="plano-summ-tag">{w.tag}</div>
                    <div className="plano-summ-title">{w.name}</div>
                    <div className="plano-summ-desc">{w.desc}</div>
                </div>
                <div className="plano-days">
```

Insert between the two divs:
```jsx
                <div className="plano-summary">
                    <div className="plano-summ-tag">{w.tag}</div>
                    <div className="plano-summ-title">{w.name}</div>
                    <div className="plano-summ-desc">{w.desc}</div>
                </div>
                {w.weakPillar && (
                    <div className="plano-focus-banner">
                        <div style={{flex:1}}>
                            <div className="plano-focus-ol">FOCO</div>
                            <div className="plano-focus-name">{w.weakPillar}</div>
                            <div className="plano-focus-sub">Reforço incluído em cada sessão</div>
                        </div>
                        <svg width="28" height="28" viewBox="0 0 28 28" fill="none">
                            <circle cx="14" cy="14" r="13" stroke="var(--orange)" strokeWidth="1.5" opacity="0.3"/>
                            <path d="M14 8v6l3.5 3.5" stroke="var(--orange)" strokeWidth="1.8" strokeLinecap="round" strokeLinejoin="round"/>
                        </svg>
                    </div>
                )}
                <div className="plano-days">
```

- [ ] **Step 3: Verify visually in browser**

1. Open the app → navigate to **Plano** tab
2. If you have an evaluation with scores stored: banner with pillar name should appear between the week summary card and the day list
3. If no evaluation: banner should be absent (no gap, no broken layout)
4. Tap a different week tab: banner should still show (same pillar for all weeks)

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add Foco da Semana banner in TelaPlano"
```

---

## Task 5: End-to-end browser verification

- [ ] **Step 1: Simulate a full athlete profile and verify everything works**

In browser console (with the app loaded):

```js
// Set up a fake evaluation to test end-to-end
const fakeAval = {
    date: '2026-05-19',
    pushReps: 20, plankSecs: 60, hikingSecs: 90,
    endType: 'cooper', endDist: 2200,
    esporte: 'velejador', tipoBarco: 'hiking',
    equipamentos: [],
    diasSemana: 4, duracao: 60, idade: 30,
    scores: { hiking: 55, core: 32, forca: 60, endurance: 72, mobilidade: 48 }
};
localStorage.setItem('regata_avaliacao', JSON.stringify(fakeAval));
location.reload();
```

After reload, verify:

| Check | Expected |
|-------|----------|
| Plano → any force day → Push-up detail | contains "10 rep" (round(20×0.6/5)×5 = 10) |
| Plano → any force day → Prancha frontal | contains "40s" (round(60×0.65/5)×5 = 40) |
| Plano → hiking session → Hiking bench isométrico | contains "65s" (round(90×0.70/5)×5 = 65) |
| Plano → endurance session → Corrida Z2 detail | contains "~7:05/km" (2200m → pace 5:27/km × 1.30 ≈ 7:05/km) |
| Plano tab → Foco banner | shows "Core" (score 32 is lowest) |

- [ ] **Step 2: Verify zero-regression for users without evaluation**

```js
localStorage.removeItem('regata_avaliacao');
location.reload();
```

After reload:
- Plano tab: no focus banner (hidden)
- Open any session: exercises show same generic text as before (fallbacks working)
- No JS errors in console

- [ ] **Step 3: Restore your own evaluation data**

If you had real data before running the tests, restore it. Otherwise clear the fake:
```js
localStorage.removeItem('regata_avaliacao');
location.reload();
```

- [ ] **Step 4: Push to GitHub Pages**

```bash
git push origin main
```

Then open https://brunofontesoficial.github.io/we-dont-stop/ and repeat the visual checks from Step 1 (you'll need to set up the fakeAval via console since it's a fresh session).
