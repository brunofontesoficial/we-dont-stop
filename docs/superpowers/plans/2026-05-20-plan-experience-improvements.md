# Plan Experience Improvements — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fazer o pilar fraco impactar sessões reais (volume + tier boost), mostrar progresso semanal e marcar sessões concluídas no Plano.

**Architecture:** Três mudanças independentes em `index.html`: (1) extender `applyTargets` + `getSessionSections` para considerar `weakPillar` e `weekIdx`; (2) adicionar banner de progresso em `TelaPlano`; (3) adicionar checkmarks de sessão concluída em cada day card. Todas dependem de um helper `getWeekDates(weekId)` criado na Task 2.

**Tech Stack:** React 18 CDN, Babel standalone, single `index.html` (~6100 lines). Sem build step. Verificação via browser console assertions.

---

## File Map

| Arquivo | Linha aprox | Mudança |
|---------|-------------|---------|
| `index.html` | ~562 | CSS: `.plano-progress-wrap`, `.plano-progress-track`, `.plano-progress-bar`, `.plano-day-check` |
| `index.html` | ~962 | Nova função `getWeekDates(weekId)` |
| `index.html` | ~5161 | `getSessionSections`: adicionar `weekIdx` param + destructure `weakPillar` + tier boost (Feature 2C) |
| `index.html` | ~5790 | `applyTargets`: adicionar lógica de volume boost por pilar (Feature 2A) |
| `index.html` | ~5891 | `TelaPlano`: banner progresso (Feature 3) + checkmarks (Feature 4) |

---

## Task 1: CSS — novos estilos para progresso e checkmark

**Files:**
- Modify: `index.html:562`

- [ ] **Step 1: Adicionar CSS após `.plano-focus-sub`**

Encontre a linha:
```css
        .plano-focus-sub { font-size: 11px; color: var(--muted-dark); margin-top: 2px; }
```

Insira imediatamente após:
```css
        .plano-progress-wrap { margin: 0 20px 12px; }
        .plano-progress-label { font-size: 9px; font-weight: 700; letter-spacing: 1.6px; text-transform: uppercase; color: rgba(24,25,31,0.45); margin-bottom: 6px; }
        .plano-progress-track { height: 4px; background: rgba(24,25,31,0.08); border-radius: 2px; overflow: hidden; }
        .plano-progress-bar { height: 100%; background: #3db87a; border-radius: 2px; transition: width 0.3s ease; }
        .plano-day-check { font-size: 15px; color: #3db87a; font-weight: 700; flex-shrink: 0; line-height: 1; }
```

- [ ] **Step 2: Commit**

```bash
git add index.html
git commit -m "style: add progress bar and session check CSS for plan improvements"
```

---

## Task 2: Helper `getWeekDates(weekId)`

**Files:**
- Modify: `index.html:~962` (após `getWeekIdx`)

- [ ] **Step 1: Inserir função após `getWeekIdx`**

Encontre este bloco (termina em ~linha 962):
```js
    function getWeekIdx() {
        try {
            const s = localStorage.getItem('regata_plan_start');
```

Localize o fechamento da função `getWeekIdx` (linha com `}`) e insira a nova função imediatamente após:

```js
    function getWeekDates(weekId) {
        const raw = localStorage.getItem('regata_plan_start');
        const planStart = raw ? new Date(raw) : new Date();
        const planDow = (planStart.getDay() + 6) % 7; // 0=Seg…6=Dom
        const monday = new Date(planStart);
        monday.setDate(planStart.getDate() - planDow + (weekId - 1) * 7);
        return Array.from({ length: 7 }, (_, i) => {
            const d = new Date(monday);
            d.setDate(monday.getDate() + i);
            return d.toLocaleDateString('sv'); // YYYY-MM-DD
        });
    }
```

- [ ] **Step 2: Verificar no browser console**

```js
// Deve retornar 7 strings YYYY-MM-DD começando na segunda da semana atual
const dates = getWeekDates(getWeekIdx());
console.assert(dates.length === 7, 'deve ter 7 datas');
console.assert(/^\d{4}-\d{2}-\d{2}$/.test(dates[0]), 'formato YYYY-MM-DD');
// O índice de hoje (TODAY_IDX) deve estar no array
const today = new Date().toLocaleDateString('sv');
console.assert(dates.includes(today), 'hoje está no array');
console.log('dates:', dates);
console.log('✅ getWeekDates OK');
```

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add getWeekDates helper for plan week date mapping"
```

---

## Task 3: Feature 4 — Checkmark de sessão concluída no Plano

**Files:**
- Modify: `index.html:~5891` (TelaPlano)

- [ ] **Step 1: Adicionar memoized history e doneDates em TelaPlano**

Encontre em `TelaPlano`:
```js
    function TelaPlano({ openSession }) {
        const PLAN_WEEKS = React.useMemo(() => getPlanWeeks(getAvaliacao()), []);
        const [week, setWeek] = useState(getWeekIdx);
        const [selected, setSelected] = useState(null);
        const w = PLAN_WEEKS.find(x => x.id === week);
```

Substitua por:
```js
    function TelaPlano({ openSession }) {
        const PLAN_WEEKS = React.useMemo(() => getPlanWeeks(getAvaliacao()), []);
        const [week, setWeek] = useState(getWeekIdx);
        const [selected, setSelected] = useState(null);
        const w = PLAN_WEEKS.find(x => x.id === week);
        const history = React.useMemo(() => getHistory(), []);
        const doneDates = React.useMemo(() => new Set(history.map(e => e.date)), [history]);
        const weekDates = React.useMemo(() => getWeekDates(week), [week]);
```

- [ ] **Step 2: Adicionar checkmark nos day cards**

Encontre no JSX de TelaPlano:
```jsx
                                {d.dur && <div className="plano-day-dur">{d.dur}</div>}
```

Substitua por:
```jsx
                                {doneDates.has(weekDates[i])
                                    ? <div className="plano-day-check">✓</div>
                                    : d.dur && <div className="plano-day-dur">{d.dur}</div>
                                }
```

- [ ] **Step 3: Verificar no browser**

1. Complete uma sessão qualquer (abra uma sessão → marque todos os exercícios → submeta RPE)
2. Volte à tela Plano → o dia de hoje deve mostrar `✓` verde no lugar da duração
3. Dias sem sessão completada ainda mostram a duração normalmente
4. Sem erros no console

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: show green checkmark on completed session days in plan"
```

---

## Task 4: Feature 3 — Banner de progresso semanal

**Files:**
- Modify: `index.html:~5920` (TelaPlano, após week-scroll)

- [ ] **Step 1: Adicionar lógica de progresso em TelaPlano**

Logo após as linhas adicionadas na Task 3 (history, doneDates, weekDates), adicione:

```js
        const currentWeekIdx = getWeekIdx();
        const currentWeekDates = React.useMemo(() => getWeekDates(currentWeekIdx), []);
        const currentWeekPlan = PLAN_WEEKS[currentWeekIdx - 1] || PLAN_WEEKS[0];
        const trainDaysCount = currentWeekPlan.days.filter(d => d.type !== 'descanso' && d.type !== 'ativo').length;
        const doneSessions = currentWeekDates.filter(date => history.some(e => e.date === date && !e.isMinimo)).length;
        const progressPct = trainDaysCount > 0 ? Math.round((doneSessions / trainDaysCount) * 100) : 0;
```

- [ ] **Step 2: Adicionar banner JSX entre topbar e week-scroll**

Encontre:
```jsx
                <div className="plano-week-scroll">
```

Insira imediatamente antes:
```jsx
                {trainDaysCount > 0 && (
                    <div className="plano-progress-wrap">
                        <div className="plano-progress-label">ESTA SEMANA · {doneSessions} de {trainDaysCount} sessões</div>
                        <div className="plano-progress-track">
                            <div className="plano-progress-bar" style={{width:`${progressPct}%`}}/>
                        </div>
                    </div>
                )}
```

- [ ] **Step 3: Verificar no browser**

```js
// Simular histórico desta semana com 2 sessões
const today = new Date().toLocaleDateString('sv');
const yesterday = new Date(Date.now()-86400000).toLocaleDateString('sv');
const hist = JSON.parse(localStorage.getItem('regata_history') || '[]');
hist.push({date:today, weekIdx:1, dayType:'forca_day', sessionName:'Circuito funcional', totalMs:3600000, isMinimo:false, rpe:7, note:'', results:{}});
hist.push({date:yesterday, weekIdx:1, dayType:'endurance_day', sessionName:'Corrida base Z2', totalMs:1800000, isMinimo:false, rpe:5, note:'', results:{}});
localStorage.setItem('regata_history', JSON.stringify(hist));
location.reload();
```

Após reload: banner "ESTA SEMANA · X de Y sessões" aparece acima das tabs de semana com barra verde proporcional.

- [ ] **Step 4: Limpar dados de teste**

```js
// Remover as entradas falsas inseridas no teste acima
const hist = JSON.parse(localStorage.getItem('regata_history') || '[]');
localStorage.setItem('regata_history', JSON.stringify(hist.filter(e => e.note !== '' || e.rpe !== 7)));
location.reload();
```

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add weekly session progress banner to TelaPlano"
```

---

## Task 5: Feature 2A — Volume boost por pilar fraco em `applyTargets`

**Files:**
- Modify: `index.html:~5161` (getSessionSections — destructure weakPillar)
- Modify: `index.html:~5795` (applyTargets — nova lógica)

- [ ] **Step 1: Destructurar `weakPillar` em getSessionSections**

Encontre em `getSessionSections`:
```js
        const { pushTarget, plankTarget, hikingTarget, endurancePace } = getPersonalizedTargets(aval);
```

Substitua por:
```js
        const { pushTarget, plankTarget, hikingTarget, endurancePace, weakPillar } = getPersonalizedTargets(aval);
```

- [ ] **Step 2: Reescrever `applyTargets` com volume boost**

Encontre a função `applyTargets` completa:
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
```

Substitua por:
```js
        function applyTargets(secs) {
            const CORE_ITEMS     = ['Prancha frontal','Hollow hold','Mountain climbers','Prancha lateral','Hanging knee raise'];
            const FORCA_ITEMS    = ['Push-up','Push-up (joelho)'];
            const HIKING_ITEMS   = ['Hiking bench isométrico','Hiking isométrico centro','Hiking bench lastrado','Wall sit'];
            const CARDIO_ITEMS   = ['Corrida Z2','Bike Z2','Ergômetro Z2'];
            return secs.map(sec => ({
                ...sec,
                items: sec.items.map(item => {
                    let detail = item.detail;
                    // Cargas personalizadas dos testes
                    if (pushTarget && FORCA_ITEMS.includes(item.name)) {
                        detail = detail.replace(/\d+ rep$/, `${pushTarget} rep`);
                    } else if (plankTarget && (item.name === 'Prancha frontal' || item.name === 'Hollow hold')) {
                        detail = detail.replace(/\d+s$/, `${plankTarget}s`);
                    } else if (hikingTarget && HIKING_ITEMS.slice(0,3).includes(item.name)) {
                        detail = detail.replace('máximo', `${hikingTarget}s`);
                    }
                    // Pace de endurance
                    if (endurancePace && CARDIO_ITEMS.includes(item.name)) {
                        if (!detail.includes('~')) detail += ` · ~${endurancePace}`;
                    }
                    // Feature 2A: volume boost pelo pilar fraco
                    if (weakPillar) {
                        if (weakPillar === 'Core' && CORE_ITEMS.includes(item.name)) {
                            detail = detail.replace(/^(\d+) ×/, (_, n) => `${+n + 1} ×`);
                        } else if (weakPillar === 'Força' && FORCA_ITEMS.includes(item.name)) {
                            detail = detail.replace(/^(\d+) ×/, (_, n) => `${+n + 1} ×`);
                        } else if (weakPillar === 'Hiking Bench' && HIKING_ITEMS.includes(item.name)) {
                            detail = detail.replace(/^(\d+) ×/, (_, n) => `${+n + 1} ×`);
                        } else if (weakPillar === 'Endurance' && CARDIO_ITEMS.includes(item.name)) {
                            detail = detail.replace(/^(\d+) min/, (_, n) => `${+n + 5} min`);
                        }
                    }
                    return { ...item, detail };
                })
            }));
        }
```

- [ ] **Step 3: Verificar no browser console**

```js
// Atleta com Core fraco
const testAval = {
    pushReps:20, plankSecs:45, hikingSecs:60,
    endType:'cooper', endDist:2200,
    esporte:'velejador', tipoBarco:'hiking',
    equipamentos:[],
    scores:{ hiking:55, core:28, forca:60, endurance:72, mobilidade:48 }
};
const sections = getSessionSections('forca_day', testAval, 'corrida');
const plank = sections.flatMap(s=>s.items).find(i=>i.name==='Prancha frontal');
const push  = sections.flatMap(s=>s.items).find(i=>i.name==='Push-up');
// plank: plankTarget = round(45*0.65/5)*5 = 30s → "3 × 30s" → com Core fraco → "4 × 30s"
console.assert(plank?.detail.startsWith('4 ×'), `Prancha deve ter 4 séries, got: ${plank?.detail}`);
// push: pushTarget = round(20*0.6/5)*5 = 10 → "X × 10 rep" (sem boost de força pois pilar é Core)
console.assert(!push?.detail.startsWith('4 ×'), `Push-up não deve ter boost (pilar é Core), got: ${push?.detail}`);
console.log('plank:', plank?.detail, '| push:', push?.detail);
console.log('✅ Feature 2A OK');
```

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: applyTargets boosts volume for weak pillar exercises (Feature 2A)"
```

---

## Task 6: Feature 2C — Tier boost nas semanas 1–2 + push final

**Files:**
- Modify: `index.html:~5161` (getSessionSections — tier boost)
- Modify: `index.html:~2379` e `~5823` (call sites que passam weekIdx)

- [ ] **Step 1: Adicionar parâmetro `weekIdx` e tier boost em `getSessionSections`**

Encontre a assinatura da função:
```js
    function getSessionSections(type, aval, cardioMode) {
```

E as linhas seguintes até `const tier`:
```js
        const scores     = aval?.scores || null;
        const scoreMap   = { forca_day:'forca', endurance_day:'endurance', sailing_endurance:'endurance' };
        const key        = scoreMap[type];
        const score      = scores && key ? (scores[key] ?? 50) : 50;
        const tier       = score < 35 ? 'L' : score < 65 ? 'M' : 'H';
        const { pushTarget, plankTarget, hikingTarget, endurancePace, weakPillar } = getPersonalizedTargets(aval);
```

Substitua por:
```js
    function getSessionSections(type, aval, cardioMode, weekIdx = getWeekIdx()) {
        const scores     = aval?.scores || null;
        const scoreMap   = { forca_day:'forca', endurance_day:'endurance', sailing_endurance:'endurance' };
        const key        = scoreMap[type];
        const score      = scores && key ? (scores[key] ?? 50) : 50;
        let tier         = score < 35 ? 'L' : score < 65 ? 'M' : 'H';
        const { pushTarget, plankTarget, hikingTarget, endurancePace, weakPillar } = getPersonalizedTargets(aval);
        // Feature 2C: boost tier para o pilar fraco nas semanas 1–2
        const PILLAR_SESSION = { 'Core':'forca_day', 'Força':'forca_day', 'Hiking Bench':'sailing_endurance', 'Endurance':'endurance_day' };
        if (weakPillar && weekIdx <= 2 && PILLAR_SESSION[weakPillar] === type && tier === 'L') tier = 'M';
```

- [ ] **Step 2: Passar `weekIdx` nos call sites relevantes**

**Call site em TelaSessionAtiva (~linha 2379):**

Encontre:
```js
            return getSessionSections(session.day.type, getAvaliacao(), session.day.cardioMode || getCardioMode());
```

Substitua por:
```js
            return getSessionSections(session.day.type, getAvaliacao(), session.day.cardioMode || getCardioMode(), session.weekIdx);
```

**Call site em TelaSessaoDetalhe (~linha 5823):**

Encontre:
```js
        const sections = getSessionSections(session.type, getAvaliacao(), cardioMode);
```

Substitua por:
```js
        const sections = getSessionSections(session.type, getAvaliacao(), cardioMode, weekIdx);
```

- [ ] **Step 3: Verificar no browser console**

```js
// Atleta com Endurance fraco — semana 1 deve ter tier M em endurance_day
const testAval = {
    pushReps:15, plankSecs:30, hikingSecs:null,
    endType:'cooper', endDist:1800,
    esporte:'corredor', equipamentos:[],
    scores:{ core:55, forca:60, endurance:28, mobilidade:50 }
};
// Sem boost (semana 3): score endurance=28 → tier L → sessão de 15 min
const s3 = getSessionSections('endurance_day', testAval, 'corrida', 3);
const cardio3 = s3.flatMap(s=>s.items).find(i=>i.name==='Corrida Z2');
// Com boost (semana 1): tier deve ser M → sessão de 30 min
const s1 = getSessionSections('endurance_day', testAval, 'corrida', 1);
const cardio1 = s1.flatMap(s=>s.items).find(i=>i.name==='Corrida Z2');
console.log('sem 3 (sem boost):', cardio3?.detail);
console.log('sem 1 (com boost):', cardio1?.detail);
// Sem 1 deve ter duração maior que sem 3 (M vs L)
console.assert(cardio1?.detail !== cardio3?.detail, 'tier boost deve mudar a sessão');
console.log('✅ Feature 2C OK');
```

- [ ] **Step 4: Verificação end-to-end no browser**

1. Cole no console para simular atleta com Core fraco:
```js
const fakeAval = {
    date:'2026-05-20', pushReps:20, plankSecs:60, hikingSecs:null,
    endType:'cooper', endDist:2200, esporte:'corredor', equipamentos:[],
    diasSemana:4, duracao:60, idade:30,
    scores:{ core:28, forca:60, endurance:72, mobilidade:55 }
};
localStorage.setItem('regata_avaliacao', JSON.stringify(fakeAval));
location.reload();
```

2. Após reload, verifique:
   - Tela Plano → banner "Foco: Core" visível
   - Tela Plano → banner "ESTA SEMANA · 0 de X sessões" com barra vazia
   - Semana 1 → abra uma sessão de força → bloco de prancha mostra `4 ×` séries
   - Semana 3 → mesma sessão → bloco de prancha mostra `3 ×` séries (sem boost)
   - Nenhum erro JS no console

3. Teste sem avaliação:
```js
localStorage.removeItem('regata_avaliacao');
location.reload();
```
   - Banner Foco: ausente
   - Banner progresso: ainda visível (mostra sessões sem avaliação)
   - Sessões: comportamento genérico como antes

- [ ] **Step 5: Commit e push**

```bash
git add index.html
git commit -m "feat: boost session tier for weak pillar in weeks 1-2 (Feature 2C)"
git push origin main
```
