# TelaPulso — Dados Reais Design

> **For agentic workers:** This spec describes UI and data changes in a single-file React PWA (`index.html`). All logic, state, and components live in that file. There are no separate modules or build steps.

## Goal

Replace all hardcoded mock data in TelaPulso with real values sourced from the biometric check-in log (`biometric_log`) and session history (`regata_history`). Add FC Repouso to TelaBiometria's daily check-in form. Implement HRV "linha média" display (today vs. 28-day rolling average with delta pill).

## Architecture

- **Data layer**: New helper functions on top of existing `getBiometricLog()` and `getHistory()`.
- **TelaBiometria**: One new field (`fcRepouso`) added to the check-in form and saved into `biometric_log`.
- **TelaPulso**: The 4 hardcoded metric objects, the `TSBChart` component, and the readiness score are all replaced with computed values. Period bar (7d/30d/90d) becomes functional.
- **No new localStorage keys**: `biometric_log` entries gain an optional `fcRepouso` field; existing entries without it are handled gracefully.

## Tech Stack

- React 18 (CDN + Babel Standalone), single `index.html`
- All state in localStorage
- No external chart library — SVG drawn inline (existing pattern)

---

## Section 1 — TelaBiometria: FC Repouso field

### What changes
Add a numeric input for "FC Repouso (bpm)" between HRV and Disposição in the check-in form.

### Implementation
- Field key: `fcRepouso`
- Input type: `number`, `inputMode="numeric"`, placeholder `"ex: 47"`
- Stored via the existing `saveBiometricEntry()` call — no changes needed there
- Initial draft value: `todayEntry.fcRepouso || 0` (backward-compatible)
- ScoreRow config: `{ label:"FC Repouso (bpm)", field:"fcRepouso", max:100, unit:" bpm", color:"#3db87a" }`
- Bar fill: `Math.min(v / 100, 1)` — lower is better for FC, but bar is just for visual proportion

### Backward compatibility
Old entries without `fcRepouso` return `undefined`; treat as `0` (no data).

---

## Section 2 — Helper functions

Four new pure functions added near the biometric/history helpers (after `getBiometricAlerts`).

### `getMetricSeries(field, days)`
Returns last `days` biometric entries that have a positive value for `field`.

```js
function getMetricSeries(field, days) {
    return getBiometricLog()
        .filter(e => (e[field] || 0) > 0)
        .slice(-days);
}
```

### `getMetricBaseline(field, days)`
Rolling average of `field` across the last `days` entries with a value.

```js
function getMetricBaseline(field, days) {
    const series = getMetricSeries(field, days);
    if (!series.length) return null;
    return series.reduce((a, e) => a + (e[field] || 0), 0) / series.length;
}
```

### `getDailyLoads()`
Returns an object mapping `YYYY-MM-DD` → training load (RPE × duration hours) for each day in `regata_history`. Days without sessions have no key.

```js
function getDailyLoads() {
    const map = {};
    getHistory().forEach(h => {
        if (h.rpe && h.totalMs) {
            map[h.date] = (map[h.date] || 0) + h.rpe * (h.totalMs / 3600000);
        }
    });
    return map;
}
```

### `buildTSBData(numDays)`
Computes CTL, ATL, TSB for the last `numDays` days using exponential moving averages.

```js
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
        result.push({ date: dateStr, ctl: Math.round(ctl * 10) / 10, atl: Math.round(atl * 10) / 10, tsb: Math.round((ctl - atl) * 10) / 10 });
    }
    return result;
}
```

---

## Section 3 — TelaPulso: metric cards

### Data flow
```
period state ('7d'|'30d'|'90d')
  → periodDays = { '7d':7, '30d':30, '90d':90 }
  → getMetricSeries(field, periodDays) for sparkline data
  → getMetricBaseline(field, 28) for HRV/FC baseline (always 28d regardless of period)
  → getMetricBaseline('sono', 7) for sleep (7-day average is clinically standard)
  → getDailyLoads() filtered to period for CARGA
```

### Metric object structure (replaces hardcoded array)
Each metric is computed inside `TelaPulso` via `React.useMemo`:

```js
const periodDays = { '7d': 7, '30d': 30, '90d': 90 }[period] || 7;

const metrics = React.useMemo(() => {
    const bioLog = getBiometricLog();
    const loads  = getDailyLoads();

    function makeMetric(field, name, color, avgDays, unit, invert) {
        const series  = getMetricSeries(field, periodDays);
        const latest  = series.length ? series[series.length - 1][field] : null;
        const baseline = getMetricBaseline(field, avgDays);
        const delta   = (latest && baseline) ? Math.round((latest / baseline - 1) * 100) : null;
        const deltaPos = invert ? delta <= 0 : delta >= 0; // FC: lower is better
        // series is mapped to plain numbers for Sparkline
        return { name, color, latest, baseline: baseline ? Math.round(baseline * 10) / 10 : null, delta, deltaPos, unit, series: series.map(e => e[field] || 0) };
    }

    // CARGA: build from load history for the period (already plain numbers)
    const today = new Date();
    const cargaSeries = Array.from({ length: periodDays }, (_, i) => {
        const d = new Date(today);
        d.setDate(today.getDate() - (periodDays - 1 - i));
        return loads[d.toLocaleDateString('sv')] || 0;
    });
    const cargaLatest  = cargaSeries[cargaSeries.length - 1] || null;
    const cargaAvg     = cargaSeries.filter(v => v > 0).reduce((a, b) => a + b, 0) / (cargaSeries.filter(v => v > 0).length || 1);
    const cargaDelta   = cargaLatest && cargaAvg ? Math.round((cargaLatest / cargaAvg - 1) * 100) : null;

    return [
        makeMetric('hrv',        'HRV',        '#4a8fd9', 28, 'ms',   false),
        makeMetric('fcRepouso',  'FC REPOUSO', '#3db87a', 28, 'bpm',  true),
        makeMetric('sono',       'SONO',       '#9b7dd4', 7,  'h',    false),
        { name:'CARGA', color:'#e87a45', latest: cargaLatest ? Math.round(cargaLatest*10)/10 : null,
          baseline: Math.round(cargaAvg*10)/10, delta: cargaDelta, deltaPos: cargaDelta >= 0,
          unit:'UA', series: cargaSeries },
    ];
}, [period]);
```

### Card render (Design B)
Each metric card renders:
1. Label (overline)
2. Row: `latest` large left + `baseline` smaller right (with avg label)
3. Delta pill: `▲ N%` (green) or `▼ N%` (orange) vs. average
4. Sparkline SVG: bars proportional to series max, dashed horizontal line at baseline
5. If `latest === null`: show "—" and "registre no check-in"

### Sparkline with média line
The existing `Sparkline` component only draws bars. It gains an optional `avgLine` prop:
- When provided, draws an SVG `<line>` at `y = height * (1 - avgLine/maxVal)` with `stroke-dasharray="4,3"`.

```js
function Sparkline({ data, color, avgLine }) {
    const max = Math.max(...data, avgLine || 0, 1);
    const h = 36, w = 100;
    const bw = Math.floor(w / data.length) - 2;
    return (
        <svg width="100%" height={h} viewBox={`0 0 ${w} ${h}`} preserveAspectRatio="none">
            {data.map((v, i) => {
                const bh = Math.max(Math.round((v / max) * (h - 4)), v > 0 ? 3 : 0);
                const isToday = i === data.length - 1;
                const below = avgLine && v > 0 && v < avgLine * 0.95;
                const fill = below ? '#e87a45' : isToday ? color : `${color}88`;
                return <rect key={i} x={i * (bw + 2)} y={h - bh} width={bw} height={bh} rx="2" fill={fill} />;
            })}
            {avgLine && (
                <line x1="0" y1={h - Math.round((avgLine / max) * (h - 4))}
                      x2={w}  y2={h - Math.round((avgLine / max) * (h - 4))}
                      stroke={color} strokeWidth="1" strokeDasharray="4,3" opacity="0.6"/>
            )}
        </svg>
    );
}
```

---

## Section 4 — TelaPulso: TSBChart real data

### Replace hardcoded TSBChart

`TelaPulso` computes `tsbData` once and passes it as a prop (avoids double computation):

```js
// inside TelaPulso, alongside the metrics useMemo:
const tsbData = React.useMemo(() => buildTSBData(periodDays), [period]);
```

`TSBChart` accepts a `data` prop (array from `buildTSBData`):

```js
function TSBChart({ data }) {
    if (!data || data.every(d => d.ctl === 0)) return null; // no data yet

    const W = 300, H = 60;
    const maxLoad = Math.max(...data.map(d => Math.max(d.ctl, d.atl)), 1);
    const minTSB  = Math.min(...data.map(d => d.tsb), -1);
    const maxTSB  = Math.max(...data.map(d => d.tsb), 1);
    const tsbRange = Math.max(maxTSB - minTSB, 1);

    // top 60% of canvas = CTL/ATL lines; bottom 40% = TSB line
    function toX(i)     { return Math.round((i / (data.length - 1)) * W); }
    function toYLoad(v) { return Math.round(H * 0.6 * (1 - v / maxLoad)); }
    function toYTSB(v)  { return Math.round(H * 0.6 + H * 0.4 * (1 - (v - minTSB) / tsbRange)); }

    const ctlPath = data.map((d, i) => `${i === 0 ? 'M' : 'L'}${toX(i)},${toYLoad(d.ctl)}`).join(' ');
    const atlPath = data.map((d, i) => `${i === 0 ? 'M' : 'L'}${toX(i)},${toYLoad(d.atl)}`).join(' ');
    const tsbPath = data.map((d, i) => `${i === 0 ? 'M' : 'L'}${toX(i)},${toYTSB(d.tsb)}`).join(' ');

    return (
        <svg width="100%" height={H} viewBox={`0 0 ${W} ${H}`} preserveAspectRatio="none">
            <path d={ctlPath} stroke="rgba(61,184,122,0.8)"    strokeWidth="1.5" fill="none"/>
            <path d={atlPath} stroke="rgba(232,122,69,0.8)"    strokeWidth="1.5" fill="none"/>
            <line x1="0" y1={toYTSB(0)} x2={W} y2={toYTSB(0)}
                  stroke="rgba(245,240,232,0.15)" strokeWidth="1" strokeDasharray="3,3"/>
            <path d={tsbPath} stroke="rgba(245,240,232,0.6)"   strokeWidth="2"   fill="none"/>
        </svg>
    );
}
```

Render: `<TSBChart data={tsbData} />`

### TSB interpretation label (inside TelaPulso, uses same `tsbData`)
```js
const latestTSB = tsbData.length ? tsbData[tsbData.length - 1].tsb : 0;
const formaLabel = latestTSB > 5  ? 'Em ascensão.'
                 : latestTSB < -10 ? 'Acumulando fadiga.'
                 :                   'Estável.';
const formaColor = latestTSB > 5  ? '#3db87a'
                 : latestTSB < -10 ? '#e87a45'
                 :                   'rgba(245,240,232,0.6)';
```

If `tsbData.every(d => d.ctl === 0)`: `TSBChart` returns `null`; the forma card shows "Complete sessões para ver sua curva de forma." instead.

---

## Section 5 — TelaPulso: Readiness circle

The readiness score in TelaPulso is currently hardcoded as `72`. Wire it to the same readiness calculation already used in TelaHoje:

- Extract the readiness logic from `TelaHoje` into a standalone `calcReadiness()` helper function placed after `getBiometricAlerts`
- Replace the inline readiness block in `TelaHoje` with a call to `calcReadiness()`
- `TelaPulso` also calls `calcReadiness()` instead of using hardcoded `72`
- No logic duplication

```js
function calcReadiness() {
    const todayStr   = new Date().toLocaleDateString('sv');
    const bioLog     = getBiometricLog();
    const bioToday   = bioLog.slice().reverse().find(e => e.date === todayStr);
    const hasBioToday = !!bioToday;
    if (hasBioToday) {
        const b = bioToday;
        let score = 0, parts = 0;
        if (b.hrv > 0) {
            const log = bioLog.filter(e => e.hrv > 0);
            const avg = log.length > 1 ? log.slice(0, -1).reduce((a, e) => a + e.hrv, 0) / (log.length - 1) : b.hrv;
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

---

## Section 6 — Period bar wiring

The period bar buttons (`7d`, `30d`, `90d`) already set `period` state. The `periodDays` variable derived from `period` drives:
- Metric series window (sparkline data points shown)
- TSBChart days prop

---

## What does NOT change

- The mock "5 conexões ativas" card and "Última saída do mar · Sailmon" card remain as-is — they are future features and removing them would change visual weight without a replacement.
- The `biometric_log` schema is additive only — no migration needed.
- All existing `TelaBiometria` behavior is preserved; only a new ScoreRow is inserted.

---

## Success criteria

1. TelaBiometria check-in shows FC Repouso field and saves it
2. TelaPulso metric cards show real values from biometric log, not hardcoded
3. HRV card shows today's value + 28-day average + delta pill (Design B)
4. Bars in sparklines turn orange when today's value is >5% below the average
5. TSBChart is hidden when no session history exists; shows real trend otherwise
6. Readiness circle matches TelaHoje readiness score
7. Period bar (7d/30d/90d) changes data window for all sparklines and TSB chart
8. No regression in TelaHoje biometric check-in or readiness display
