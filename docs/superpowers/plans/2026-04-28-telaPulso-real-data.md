# TelaPulso Real Data Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace all hardcoded mock data in TelaPulso with real values from `biometric_log` and `regata_history`, add FC Repouso to the daily check-in, and implement HRV "linha média" (today vs. 28-day rolling average with delta pill).

**Architecture:** All changes are in `index.html` (single-file React PWA). New helper functions are inserted after `getBiometricAlerts` (~line 941). The `Sparkline` component gains an `avgLine` prop. `TelaPulso` is rewritten to compute metrics from real data. `TelaBiometria` gains one new field. Readiness logic is extracted to `calcReadiness()` shared by both `TelaHoje` and `TelaPulso`.

**Tech Stack:** React 18 (CDN + Babel Standalone), localStorage, inline SVG charts, no build step.

---

## File Structure

Only one file changes: `index.html`

| Section | Lines | Change |
|---------|-------|--------|
| `Sparkline` component | ~611–635 | Add `avgLine` prop, change line chart → bar chart |
| After `getBiometricAlerts` | ~942 | Insert 5 new helper functions |
| `TelaHoje` readiness block | ~1207–1226 | Replace inline IIFE with `calcReadiness()` call |
| `TSBChart` component | ~1777–1798 | Rewrite with `data` prop + real SVG logic |
| `TelaPulso` function | ~1800–1903 | Full replacement with real computed data |
| `TelaBiometria` check-in | ~3287, 3343 | Add `fcRepouso` to draft default + new ScoreRow |

---

## Task 1: Add helper functions

**Files:**
- Modify: `index.html` (~line 942, after `getBiometricAlerts` closing brace)

Context: The app has `getBiometricAlerts()` ending around line 941. Add 5 new pure functions immediately after it. These functions have no UI and don't affect any existing behavior.

- [ ] **Step 1: Locate the insertion point**

Open `index.html`. Find this line (should be ~line 941):
```js
        return alerts;
    }
```
The line after this closing brace is where you insert (it should currently be `function getNotifEnabled()`).

- [ ] **Step 2: Insert `calcReadiness` after `getBiometricAlerts`**

After the closing `}` of `getBiometricAlerts` (after line 941), insert:

```js
    function calcReadiness() {
        const todayStr = new Date().toLocaleDateString('sv');
        const bioLog   = getBiometricLog();
        const bioToday = bioLog.slice().reverse().find(e => e.date === todayStr);
        if (bioToday) {
            const b = bioToday;
            let score = 0, parts = 0;
            if (b.hrv > 0) {
                const log = bioLog.filter(e => e.hrv > 0);
                const avg = log.length > 1 ? log.slice(0,-1).reduce((a,e) => a + e.hrv, 0) / (log.length - 1) : b.hrv;
                score += Math.min(Math.round((b.hrv / avg) * 65), 90); parts++;
            }
            if (b.disposicao > 0) { score += Math.round((b.disposicao / 5) * 85); parts++; }
            if (b.sono > 0)       { score += Math.min(Math.round((b.sono / 8) * 85), 90); parts++; }
            if (parts > 0) return Math.min(Math.round(score / parts), 99);
        }
        const aval = getAvaliacao();
        if (!aval?.scores) return null;
        const s = aval.scores;
        const vals = [s.core, s.forca, s.endurance, s.mobilidade].filter(v => typeof v === 'number');
        if (!vals.length) return null;
        return Math.round(40 + (vals.reduce((a, b) => a + b, 0) / vals.length) * 0.55);
    }
```

- [ ] **Step 3: Insert 4 data helpers immediately after `calcReadiness`**

```js
    function getMetricSeries(field, days) {
        return getBiometricLog()
            .filter(e => (e[field] || 0) > 0)
            .slice(-days);
    }

    function getMetricBaseline(field, days) {
        const series = getMetricSeries(field, days);
        if (!series.length) return null;
        return series.reduce((a, e) => a + (e[field] || 0), 0) / series.length;
    }

    function getDailyLoads() {
        const map = {};
        getHistory().forEach(h => {
            if (h.rpe && h.totalMs) {
                map[h.date] = (map[h.date] || 0) + h.rpe * (h.totalMs / 3600000);
            }
        });
        return map;
    }

    function buildTSBData(numDays) {
        const loads = getDailyLoads();
        const today = new Date();
        let ctl = 0, atl = 0;
        const result = [];
        for (let i = numDays - 1; i >= 0; i--) {
            const d = new Date(today);
            d.setDate(today.getDate() - i);
            const dateStr = d.toLocaleDateString('sv');
            const load = loads[dateStr] || 0;
            ctl = ctl + (load - ctl) * (1 - Math.exp(-1 / 42));
            atl = atl + (load - atl) * (1 - Math.exp(-1 / 7));
            result.push({
                date: dateStr,
                ctl:  Math.round(ctl  * 10) / 10,
                atl:  Math.round(atl  * 10) / 10,
                tsb:  Math.round((ctl - atl) * 10) / 10,
            });
        }
        return result;
    }
```

- [ ] **Step 4: Start dev server and verify no console errors**

```bash
python3 -m http.server 8000 --directory "/Users/brunofontes/Projects/App de treino - we dont stop"
```

Open http://localhost:8000/we-dont-stop/ and open browser DevTools console. Verify: no JavaScript errors. App loads normally. TelaHoje still shows readiness correctly.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add calcReadiness + biometric/TSB data helpers"
```

---

## Task 2: Upgrade Sparkline to bar chart with avgLine

**Files:**
- Modify: `index.html` (~lines 611–635)

Context: The `Sparkline` component currently draws a line chart with gradient fill. It's only used in `TelaPulso` (line 1845). Replace it with a bar chart that accepts an optional `avgLine` prop — when provided, draws a dashed horizontal reference line and colors bars orange when below `avgLine * 0.95`.

- [ ] **Step 1: Replace the `Sparkline` function**

Find the existing `Sparkline` function (starts around line 611):
```js
    function Sparkline({ data, color }) {
        const W = 100, H = 36;
        const min = Math.min(...data), max = Math.max(...data);
```

Replace the entire function (lines 611–635, from `function Sparkline` through its closing `}`) with:

```js
    function Sparkline({ data, color, avgLine }) {
        if (!data || !data.length) return null;
        const W = 100, H = 36;
        const max = Math.max(...data, avgLine || 0, 1);
        const bw  = Math.max(Math.floor(W / data.length) - 2, 2);
        return (
            <svg width="100%" height={H} viewBox={`0 0 ${W} ${H}`} preserveAspectRatio="none">
                {data.map((v, i) => {
                    const bh    = v > 0 ? Math.max(Math.round((v / max) * (H - 4)), 3) : 0;
                    const below = avgLine && v > 0 && v < avgLine * 0.95;
                    const isToday = i === data.length - 1;
                    const fill  = below ? '#e87a45' : isToday ? color : `${color}88`;
                    return <rect key={i} x={i * (bw + 2)} y={H - bh} width={bw} height={bh} rx="2" fill={fill}/>;
                })}
                {avgLine != null && (
                    <line
                        x1="0" y1={H - Math.round((avgLine / max) * (H - 4))}
                        x2={W} y2={H - Math.round((avgLine / max) * (H - 4))}
                        stroke={color} strokeWidth="1" strokeDasharray="4,3" opacity="0.6"
                    />
                )}
            </svg>
        );
    }
```

- [ ] **Step 2: Verify TelaPulso still renders**

Reload http://localhost:8000/we-dont-stop/ and tap the "Pulso" tab. Verify: the 4 metric cards still appear with sparklines (still using mock data for now — that's fine). No console errors.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: upgrade Sparkline to bar chart with avgLine dashed reference line"
```

---

## Task 3: Refactor TelaHoje + add FC Repouso to TelaBiometria

**Files:**
- Modify: `index.html` (~lines 1207–1226 for TelaHoje, ~lines 3287 and 3343–3345 for TelaBiometria)

### Part A — TelaHoje: use calcReadiness()

Context: `TelaHoje` has an inline IIFE (~lines 1207–1226) that computes readiness. Replace it with a call to the new `calcReadiness()` helper.

- [ ] **Step 1: Replace the readiness IIFE in TelaHoje**

Find this block in `TelaHoje` (~line 1207):
```js
        // Readiness: biometric-based (priority) or aval proxy
        const readiness = (() => {
            if (hasBioToday) {
                const b = bioToday;
                let score = 0, parts = 0;
                if (b.hrv > 0) {
                    const log = getBiometricLog().filter(e => e.hrv > 0);
                    const avg = log.length > 1 ? log.slice(0,-1).reduce((a,e) => a + e.hrv, 0) / (log.length - 1) : b.hrv;
                    score += Math.min(Math.round((b.hrv / avg) * 65), 90); parts++;
                }
                if (b.disposicao > 0) { score += Math.round((b.disposicao / 5) * 85); parts++; }
                if (b.sono > 0) { score += Math.min(Math.round((b.sono / 8) * 85), 90); parts++; }
                if (parts > 0) return Math.min(Math.round(score / parts), 99);
            }
            if (!aval?.scores) return null;
            const s = aval.scores;
            const vals = [s.core, s.forca, s.endurance, s.mobilidade].filter(v => typeof v === 'number');
            if (!vals.length) return null;
            const avg = vals.reduce((a,b) => a+b, 0) / vals.length;
            return Math.round(40 + avg * 0.55);
        })();
```

Replace it with:
```js
        // Readiness: shared helper (biometric-based or aval proxy)
        const readiness = calcReadiness();
```

- [ ] **Step 2: Verify TelaHoje readiness is unchanged**

Reload and check TelaHoje. The readiness circle and phrase ("Dia de atacar.", "Mar calmo.", etc.) must behave exactly as before.

### Part B — TelaBiometria: FC Repouso field

- [ ] **Step 3: Add fcRepouso to draft default**

Find the `todayEntry` and draft initialization in `TelaBiometria` (~line 3287):
```js
        const todayEntry = log.find(e => e.date === today) || { date:today, hrv:0, disposicao:0, sono:0 };
```

Replace with:
```js
        const todayEntry = log.find(e => e.date === today) || { date:today, hrv:0, fcRepouso:0, disposicao:0, sono:0 };
```

- [ ] **Step 4: Allow ScoreRow to render fcRepouso as numeric input**

Find the `ScoreRow` component inside `TelaBiometria` (~line 3309):
```js
                    {field === 'hrv' ? (
                        <input type="number" inputMode="numeric" placeholder="ex: 58" value={v||''} onChange={e => set(field, parseFloat(e.target.value)||0)}
                            style={{width:'100%',padding:'10px 14px',border:'1.5px solid rgba(24,25,31,0.12)',borderRadius:10,fontSize:15,color:'var(--text-dark)',background:'rgba(24,25,31,0.03)',boxSizing:'border-box',outline:'none'}}/>
```

Change `field === 'hrv'` to `field === 'hrv' || field === 'fcRepouso'`:
```js
                    {(field === 'hrv' || field === 'fcRepouso') ? (
                        <input type="number" inputMode="numeric" placeholder="ex: 58" value={v||''} onChange={e => set(field, parseFloat(e.target.value)||0)}
                            style={{width:'100%',padding:'10px 14px',border:'1.5px solid rgba(24,25,31,0.12)',borderRadius:10,fontSize:15,color:'var(--text-dark)',background:'rgba(24,25,31,0.03)',boxSizing:'border-box',outline:'none'}}/>
```

- [ ] **Step 5: Insert FC Repouso ScoreRow in the check-in form**

Find the ScoreRow lines in the check-in form (~line 3343):
```jsx
                        <ScoreRow label="HRV (ms) — do seu relógio ou app" field="hrv"        max={120} unit=" ms" color="#4a8fd9"/>
                        <ScoreRow label="Disposição (1–5)"                  field="disposicao" max={5}   unit="/5"  color="#e87a45"/>
```

Replace with (FC Repouso inserted between HRV and Disposição):
```jsx
                        <ScoreRow label="HRV (ms) — do seu relógio ou app" field="hrv"        max={120} unit=" ms"  color="#4a8fd9"/>
                        <ScoreRow label="FC Repouso (bpm)"                  field="fcRepouso"  max={100} unit=" bpm" color="#3db87a"/>
                        <ScoreRow label="Disposição (1–5)"                  field="disposicao" max={5}   unit="/5"   color="#e87a45"/>
```

- [ ] **Step 6: Verify check-in form shows 4 fields**

Reload. Go to TelaHoje → tap the check-in card (or tap "Check-in de hoje"). The form must now show 4 rows: HRV, FC Repouso, Disposição, Horas de sono. Fill in a value for FC Repouso and tap SALVAR. Verify: no errors, check-in saved successfully (green checkmark).

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat: extract calcReadiness to shared helper; add FC Repouso to daily check-in"
```

---

## Task 4: Rewrite TSBChart with real data

**Files:**
- Modify: `index.html` (~lines 1777–1798)

Context: `TSBChart` currently draws hardcoded lines. Replace with a version that accepts `data` (output of `buildTSBData()`). If no sessions with RPE exist, it returns `null`.

- [ ] **Step 1: Replace TSBChart**

Find the existing `TSBChart` function (~line 1777):
```js
    function TSBChart() {
        const W = 300, H = 60;
        const fit  = [62,63,64,65,65,66,67,68];
```

Replace the entire function (from `function TSBChart()` through its closing `}`, lines ~1777–1798) with:

```js
    function TSBChart({ data }) {
        if (!data || data.every(d => d.ctl === 0)) return null;
        const W = 300, H = 60;
        const maxLoad  = Math.max(...data.map(d => Math.max(d.ctl, d.atl)), 1);
        const minTSB   = Math.min(...data.map(d => d.tsb), -1);
        const maxTSB   = Math.max(...data.map(d => d.tsb), 1);
        const tsbRange = Math.max(maxTSB - minTSB, 1);
        function toX(i)     { return (i / (data.length - 1)) * W; }
        function toYLoad(v) { return H * 0.6 * (1 - v / maxLoad); }
        function toYTSB(v)  { return H * 0.6 + H * 0.4 * (1 - (v - minTSB) / tsbRange); }
        const ctlPath = data.map((d, i) => `${i === 0 ? 'M' : 'L'}${toX(i).toFixed(1)},${toYLoad(d.ctl).toFixed(1)}`).join(' ');
        const atlPath = data.map((d, i) => `${i === 0 ? 'M' : 'L'}${toX(i).toFixed(1)},${toYLoad(d.atl).toFixed(1)}`).join(' ');
        const tsbPath = data.map((d, i) => `${i === 0 ? 'M' : 'L'}${toX(i).toFixed(1)},${toYTSB(d.tsb).toFixed(1)}`).join(' ');
        const zero    = toYTSB(0).toFixed(1);
        return (
            <svg width="100%" height={H} viewBox={`0 0 ${W} ${H}`} preserveAspectRatio="none">
                <path d={ctlPath} stroke="rgba(61,184,122,0.8)"   strokeWidth="1.5" fill="none"/>
                <path d={atlPath} stroke="rgba(232,122,69,0.8)"   strokeWidth="1.5" fill="none" strokeDasharray="5,3"/>
                <line x1="0" y1={zero} x2={W} y2={zero}
                      stroke="rgba(245,240,232,0.15)" strokeWidth="1" strokeDasharray="3,3"/>
                <path d={tsbPath} stroke="rgba(245,240,232,0.55)" strokeWidth="2"   fill="none"/>
            </svg>
        );
    }
```

- [ ] **Step 2: Verify no crash**

Reload. Go to Pulso tab. The forma card area will be blank (TSBChart returns null because no session history has RPE yet) — that is expected and correct. No console errors.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: rewrite TSBChart with real CTL/ATL/TSB data from session history"
```

---

## Task 5: Rewrite TelaPulso with real computed data

**Files:**
- Modify: `index.html` (~lines 1800–1903)

Context: Replace the entire `TelaPulso` function. The current version has hardcoded `metrics = [...]`, a hardcoded readiness score of 72, hardcoded readiness rows, a hardcoded TSBChart call, and hardcoded forma label. Replace with real computed values using all the helpers from Tasks 1–4.

- [ ] **Step 1: Replace TelaPulso**

Find the existing function (~line 1800):
```js
    function TelaPulso() {
        const [period, setPeriod] = useState('7d');
        const metrics = [
            { name: 'HRV', ...
```

Replace the entire `TelaPulso` function (from `function TelaPulso()` to its closing `}` around line 1903) with:

```js
    function TelaPulso() {
        const [period, setPeriod] = useState('7d');
        const periodDays = { '7d': 7, '30d': 30, '90d': 90 }[period] || 7;

        const metrics = React.useMemo(() => {
            const loads = getDailyLoads();
            function makeMetric(field, name, color, avgDays, unit, invert) {
                const series   = getMetricSeries(field, periodDays);
                const latest   = series.length ? series[series.length - 1][field] : null;
                const baseline = getMetricBaseline(field, avgDays);
                const bRound   = baseline ? Math.round(baseline * 10) / 10 : null;
                const delta    = (latest && baseline) ? Math.round((latest / baseline - 1) * 100) : null;
                const deltaPos = invert ? (delta !== null && delta <= 0) : (delta !== null && delta >= 0);
                return { name, color, latest, baseline: bRound, delta, deltaPos, unit, series: series.map(e => e[field] || 0) };
            }
            const today = new Date();
            const cargaSeries = Array.from({ length: periodDays }, (_, i) => {
                const d = new Date(today);
                d.setDate(today.getDate() - (periodDays - 1 - i));
                return loads[d.toLocaleDateString('sv')] || 0;
            });
            const cargaLatest = cargaSeries[cargaSeries.length - 1] || null;
            const nonZero     = cargaSeries.filter(v => v > 0);
            const cargaAvg    = nonZero.length ? nonZero.reduce((a, b) => a + b, 0) / nonZero.length : 0;
            const cargaDelta  = cargaLatest && cargaAvg ? Math.round((cargaLatest / cargaAvg - 1) * 100) : null;
            // baseline is null when no load history — avoids drawing a 0-line in Sparkline
            const cargaBaseline = nonZero.length ? Math.round(cargaAvg * 10) / 10 : null;
            return [
                makeMetric('hrv',       'HRV',        '#4a8fd9', 28, 'ms',  false),
                makeMetric('fcRepouso', 'FC REPOUSO', '#3db87a', 28, 'bpm', true),
                makeMetric('sono',      'SONO',       '#9b7dd4', 7,  'h',   false),
                { name: 'CARGA', color: '#e87a45',
                  latest:   cargaLatest ? Math.round(cargaLatest * 10) / 10 : null,
                  baseline: cargaBaseline,
                  delta:    cargaDelta,
                  deltaPos: cargaDelta !== null && cargaDelta >= 0,
                  unit: 'UA', series: cargaSeries },
            ];
        }, [period]);

        const tsbData = React.useMemo(() => buildTSBData(periodDays), [period]);
        const readiness = calcReadiness();
        const latestTSB = tsbData.length ? tsbData[tsbData.length - 1].tsb : 0;
        const hasTSB    = tsbData.some(d => d.ctl > 0);
        const formaLabel = latestTSB > 5  ? 'Em ascensão.'
                         : latestTSB < -10 ? 'Acumulando fadiga.'
                         :                   'Estável.';
        const formaColor = latestTSB > 5  ? '#3db87a'
                         : latestTSB < -10 ? '#e87a45'
                         :                   'rgba(245,240,232,0.6)';
        const tsbVal = latestTSB > 0 ? `+${latestTSB}` : `${latestTSB}`;

        // Readiness detail rows (HRV, Sono, FC Repouso) from real metrics
        const [mHrv, mFc, mSono] = metrics;
        const readinessRows = [
            { k: 'HRV',        v: mHrv.latest  ? `${mHrv.latest}ms`  : '—',  d: mHrv.delta  !== null ? (mHrv.delta  >= 0 ? `+${mHrv.delta}%`  : `${mHrv.delta}%`)  : '—' },
            { k: 'Sono',       v: mSono.latest ? `${mSono.latest}h`   : '—',  d: mSono.delta !== null ? (mSono.delta >= 0 ? `+${mSono.delta}%` : `${mSono.delta}%`) : '—' },
            { k: 'FC repouso', v: mFc.latest   ? `${mFc.latest}`      : '—',  d: mFc.delta   !== null ? (mFc.delta   >= 0 ? `+${mFc.delta}%`   : `${mFc.delta}%`)   : '—' },
        ];
        const readinessPhrase = !readiness ? 'Bom dia.' : readiness >= 80 ? 'Dia de atacar.' : readiness >= 65 ? 'Mar calmo.' : readiness >= 50 ? 'Treino moderado.' : 'Escuta o corpo.';

        return (
            <div>
                <div className="page-header pad">
                    <div className="overline" style={{marginBottom:6}}>{getGreeting()} · {getTodayLabel()}</div>
                    <div className="page-title">Pulso</div>
                    <div className="readiness-big-card">
                        <ReadinessCircle score={readiness || 0} size={90}/>
                        <div className="readiness-detail">
                            <div className="card-label">READINESS</div>
                            <div className="readiness-hl">{readinessPhrase}</div>
                            <div className="readiness-rows">
                                {readinessRows.map((r, i) => (
                                    <div key={i} className="readiness-row">
                                        <span className="readiness-key">{r.k}</span>
                                        <span><span className="readiness-val">{r.v}</span><span className="readiness-delta">{r.d}</span></span>
                                    </div>
                                ))}
                            </div>
                        </div>
                    </div>
                    <div className="period-bar">
                        {['7d','30d','90d'].map(p => (
                            <button key={p} className={`period-btn ${period===p?'on':'off'}`} onClick={() => setPeriod(p)}>{p}</button>
                        ))}
                    </div>
                    <div className="metrics-grid">
                        {metrics.map((m, i) => (
                            <div key={i} className="metric-card">
                                <div className="metric-hdr">
                                    <span className="metric-name-txt">{m.name}</span>
                                </div>
                                {m.latest === null ? (
                                    <div style={{fontSize:12,color:'rgba(245,240,232,0.28)',margin:'6px 0 4px'}}>—</div>
                                ) : (
                                    <>
                                        <div style={{display:'flex',justifyContent:'space-between',alignItems:'flex-end',marginBottom:3}}>
                                            <div style={{display:'flex',alignItems:'baseline',gap:3}}>
                                                <span style={{fontSize:22,fontWeight:700,color:'#f5f0e8',fontFamily:'var(--font-display)'}}>{m.latest}</span>
                                                <span style={{fontSize:9,color:'rgba(245,240,232,0.38)'}}>{m.unit}</span>
                                            </div>
                                            {m.baseline !== null && (
                                                <div style={{textAlign:'right'}}>
                                                    <div style={{fontSize:12,fontWeight:600,color:m.color}}>{m.baseline}</div>
                                                    <div style={{fontSize:8,color:'rgba(245,240,232,0.28)'}}>MÉD</div>
                                                </div>
                                            )}
                                        </div>
                                        {m.delta !== null && (
                                            <div style={{
                                                display:'inline-block',
                                                background: m.deltaPos ? 'rgba(61,184,122,0.15)' : 'rgba(232,122,69,0.15)',
                                                border: `1px solid ${m.deltaPos ? 'rgba(61,184,122,0.3)' : 'rgba(232,122,69,0.3)'}`,
                                                borderRadius:20, padding:'2px 7px', fontSize:9, fontWeight:700,
                                                color: m.deltaPos ? '#3db87a' : '#e87a45', marginBottom:4
                                            }}>
                                                {m.deltaPos ? '▲' : '▼'} {Math.abs(m.delta)}%
                                            </div>
                                        )}
                                    </>
                                )}
                                <div className="sparkline-wrap">
                                    <Sparkline data={m.series} color={m.color} avgLine={m.baseline}/>
                                </div>
                                {m.latest === null && (
                                    <div style={{fontSize:8,color:'rgba(245,240,232,0.22)',marginTop:3}}>registre no check-in</div>
                                )}
                            </div>
                        ))}
                    </div>
                    {hasTSB ? (
                        <div className="forma-card">
                            <div style={{display:'flex',justifyContent:'space-between',alignItems:'flex-start',marginBottom:14}}>
                                <div>
                                    <div className="forma-lbl">FORMA</div>
                                    <div className="forma-mood" style={{color:formaColor}}>{formaLabel}</div>
                                </div>
                                <div style={{textAlign:'right'}}>
                                    <div className="forma-val">{tsbVal}</div>
                                    <div className="forma-unit">TSB</div>
                                </div>
                            </div>
                            <TSBChart data={tsbData}/>
                            <div style={{display:'flex',gap:14,marginTop:10}}>
                                {[
                                    {c:'rgba(61,184,122,0.8)',  l:`Fitness ${tsbData[tsbData.length-1].ctl.toFixed(1)}`},
                                    {c:'rgba(232,122,69,0.8)',  l:`Fadiga ${tsbData[tsbData.length-1].atl.toFixed(1)}`},
                                    {c:'rgba(245,240,232,0.45)',l:`Forma ${tsbVal}`},
                                ].map((it, i) => (
                                    <div key={i} style={{display:'flex',alignItems:'center',gap:4}}>
                                        <div style={{width:7,height:7,borderRadius:'50%',background:it.c}}/>
                                        <span style={{fontSize:10,color:'var(--muted-dark)'}}>{it.l}</span>
                                    </div>
                                ))}
                            </div>
                        </div>
                    ) : (
                        <div className="forma-card" style={{textAlign:'center',padding:'24px 16px'}}>
                            <div className="forma-lbl">FORMA</div>
                            <div style={{fontSize:12,color:'rgba(245,240,232,0.35)',marginTop:8,lineHeight:1.5}}>
                                Complete sessões com RPE<br/>para ver sua curva de forma.
                            </div>
                        </div>
                    )}
                    <div style={{background:'#fff',borderRadius:16,padding:'14px 16px',display:'flex',alignItems:'center',gap:12,marginBottom:10,cursor:'pointer'}}>
                        <div style={{display:'flex',position:'relative',height:28,width:76,flexShrink:0}}>
                            {[{l:'W',bg:'#1a7a4a'},{l:'G',bg:'#1a5fa8'},{l:'A',bg:'#d43333'},{l:'S',bg:'#e87a45'}].map((c,i)=>(
                                <div key={i} style={{width:28,height:28,borderRadius:'50%',background:c.bg,display:'flex',alignItems:'center',justifyContent:'center',fontSize:9,fontWeight:700,color:'#fff',border:'2px solid #fff',position:'absolute',left:i*18,zIndex:4-i}}>{c.l}</div>
                            ))}
                        </div>
                        <div style={{flex:1}}>
                            <div style={{fontSize:13,fontWeight:600,color:'var(--text-dark)',marginBottom:2}}>Conexões</div>
                            <div style={{fontSize:11,color:'var(--muted-light)'}}>Garmin · Whoop · Apple Health · em breve</div>
                        </div>
                        <span className="chevron">›</span>
                    </div>
                    <div style={{display:'flex',justifyContent:'space-between',alignItems:'center',marginBottom:8}}>
                        <div style={{fontSize:9,fontWeight:600,letterSpacing:'1.6px',textTransform:'uppercase',color:'var(--muted-dark)'}}>ÚLTIMA SAÍDA DO MAR</div>
                        <div style={{fontSize:9,fontWeight:600,letterSpacing:'1.6px',textTransform:'uppercase',color:'var(--orange)'}}>SAILMON</div>
                    </div>
                    <div style={{background:'#fff',borderRadius:16,padding:'14px 16px',display:'flex',alignItems:'center',gap:14,cursor:'pointer',marginBottom:16}}>
                        <div style={{width:48,height:48,borderRadius:13,background:'rgba(232,122,69,0.1)',display:'flex',alignItems:'center',justifyContent:'center',flexShrink:0}}>
                            <svg width="22" height="26" viewBox="0 0 22 26" fill="none">
                                <path d="M11 2L11 24L2 24Z" fill="#e87a45"/>
                                <path d="M11 2L11 17L20 17Z" fill="rgba(232,122,69,0.35)"/>
                            </svg>
                        </div>
                        <div style={{flex:1}}>
                            <div style={{fontFamily:'var(--font-display)',fontSize:17,fontWeight:600,color:'var(--text-dark)',marginBottom:3}}>Treino de barlavento</div>
                            <div style={{fontSize:11,color:'var(--muted-light)'}}>Integração Sailmon em breve</div>
                        </div>
                        <span className="chevron">›</span>
                    </div>
                </div>
            </div>
        );
    }
```

- [ ] **Step 2: Verify metric cards with no data**

Reload, go to Pulso tab. If no biometric check-ins exist yet, all 4 cards should show "—" with "registre no check-in". The readiness circle shows 0. TSBChart is hidden, forma card shows placeholder text. No console errors.

- [ ] **Step 3: Verify metric cards with data**

Do a biometric check-in (TelaHoje → Check-in). Enter HRV=62, FC Repouso=47, Disposição=4, Sono=7.5. Go back to Pulso tab. The HRV card should show "62 ms" as today's value. Since there's only 1 entry, baseline is 62, delta is 0%. No console errors.

- [ ] **Step 4: Verify period bar**

Switch between 7d / 30d / 90d. Sparklines should re-render (same data since we only have 1 entry, but no errors or crashes).

- [ ] **Step 5: Verify TSBChart after completing a session**

From TelaHoje, start and complete a session (tap through all exercises and tap "Encerrar"). Set RPE to 7. Then go to Pulso tab. The forma card should now show the TSBChart SVG with real lines (CTL, ATL, TSB). The forma label shows "Estável." (since TSB ≈ 0 after 1 session). TSB value shows the computed number.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: rewrite TelaPulso with real biometric + session load data"
```
