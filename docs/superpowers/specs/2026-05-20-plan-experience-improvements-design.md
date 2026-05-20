# Plan Experience Improvements — Design Spec

**Goal:** Tornar o plano de treino visivelmente personalizado (pilar fraco impacta sessões reais), mostrar progresso semanal e marcar sessões concluídas — três melhorias que juntas fazem o atleta sentir que o plano foi feito pra ele.

**Tech Stack:** React 18 CDN, Babel standalone, single `index.html` (~6100 lines). Sem build step. All changes go in `index.html`.

---

## Feature 2 — Pilar fraco impacta sessões

### O que muda

Atualmente `applyTargets` personaliza cargas (reps, tempo, pace) com base nos testes. Ele não considera o `weakPillar`.

Adicionar dois comportamentos:

**A — Mais volume na sessão (todas as semanas):**

| Pilar fraco | O que muda na sessão |
|-------------|----------------------|
| Core | Itens com `detail` no padrão `N × Xs` (prancha, hollow hold) sobem 1 série: `3 → 4` |
| Força | Itens de push-up sobem 1 série no padrão `N × X rep` |
| Hiking Bench | Itens de hiking bench sobem 1 série |
| Endurance | `cardioDetail` durations sobem 5 min: `15→20`, `30→35`, `35→40` |
| Mobilidade | Sem mudança de volume (exercícios de mobilidade já estão em blocos de descanso) |

**C — Primeiras 2 semanas priorizam o pilar (wi=0 e wi=1):**

`getSessionSections` recebe `weekIdx` como novo parâmetro. Para semanas 1 e 2 com `weakPillar` definido, sessões do tipo do pilar fraco usam tier um nível acima:

| Pilar fraco | Tipo de sessão afetada | Tier nas sem 1–2 |
|-------------|------------------------|-----------------|
| Core / Força | `forca_day` | L→M (sem 1), M→M (sem 2) |
| Endurance | `endurance_day` | L→M (sem 1), M→M (sem 2) |
| Hiking Bench | `sailing_endurance` | L→M (sem 1), M→M (sem 2) |

### Mudanças de código

- `getSessionSections(type, aval, cardioMode, weekIdx = getWeekIdx())` — `weekIdx` opcional com default
- `applyTargets(secs, weakPillar, weekIdx)` — adicionar parâmetros
- Call sites que passam `weekIdx` explicitamente: `TelaSessaoDetalhe` (já tem prop) e `TelaSessionAtiva` (usa `session.weekIdx`)
- Call site de contagem em `TelaHoje` usa o default (sem mudança)
- Regex para bumpar série: `detail.replace(/^(\d+) ×/, (_, n) => \`${+n + 1} ×\`)`
- Bump de `cardioDetail` apenas se `weakPillar === 'Endurance'`

### O que não muda

- Estrutura das semanas (dias treino/descanso)
- Exercícios listados — apenas séries e durações
- Usuários sem avaliação: comportamento idêntico ao atual

---

## Feature 3 — Progresso da semana

### O que muda

Banner entre `plano-topbar` e `plano-week-scroll` em `TelaPlano`:

```
ESTA SEMANA
3 de 4 sessões  ████████░░  75%
```

### Lógica

1. Computar as 7 datas da semana atual (segunda a domingo) a partir de `TODAY_IDX`
2. Contar dias da semana atual que têm tipo de treino (não `descanso`, não `ativo`) em `PLAN_WEEKS[getWeekIdx()-1].days`
3. Contar entradas em `getHistory()` onde `e.date` está dentro das datas da semana e `!e.isMinimo`
4. Exibir `X de Y sessões`

### CSS

`.plano-progress-wrap` — container entre topbar e week-scroll
`.plano-progress-bar` — barra verde preenchida proporcionalmente

### Comportamento de edge cases

- Sem histórico: mostra `0 de Y sessões` com barra vazia
- Semana completa: barra cheia, texto `Y de Y sessões`
- Esconde se `Y === 0` (semana sem treinos — improvável mas defensivo)

---

## Feature 4 — Check de sessão concluída no Plano

### O que muda

Cada card de dia não-descanso no Plano mostra um checkmark `✓` verde quando existe entrada no histórico para aquela data.

### Lógica

1. Computar a data absoluta de cada dia `i` da semana exibida (`week`) a partir de `regata_plan_start` e `week`
2. Checar `getHistory().some(e => e.date === dateForDay)` para cada day card
3. Se `done`, exibir ícone `✓` no canto direito do card (substituindo `plano-day-dur`)

### Cálculo de data por dia

```
planStart = new Date(localStorage.getItem('regata_plan_start'))
planStartMon = planStart - planStart.getDay() (ajustado pra segunda-feira)
dayDate = planStartMon + (week - 1) * 7 + dayIndex   // dayIndex: 0=Seg…6=Dom
→ toLocaleDateString('sv')  // YYYY-MM-DD para comparar com history
```

Helper `getWeekDates(weekId)` retorna array[7] de strings YYYY-MM-DD para a semana do plano `weekId`.

### CSS

`.plano-day-check` — ✓ verde, `font-size: 14px`, `color: #3db87a`, `font-weight: 700`

### Comportamento

- Semanas futuras: nenhum check (histórico não tem datas futuras)
- Dias de descanso: sem check (tipo `descanso`)
- Dias de recuperação ativa: check aparece se `isMinimo` for false no histórico

---

## File Map

| Arquivo | Linha aprox | Mudança |
|---------|-------------|---------|
| `index.html` | ~5161 | `getSessionSections` recebe `weekIdx`; lógica de tier boost C |
| `index.html` | ~5790 | `applyTargets` recebe `weakPillar` e `weekIdx`; lógica volume A |
| `index.html` | ~5165 | cardioDetail bump para Endurance fraco |
| `index.html` | ~5895 | `TelaPlano` — banner progresso (Feature 3) |
| `index.html` | ~5946 | `TelaPlano` — day cards com check (Feature 4) |
| `index.html` | ~5000 | Helper `getWeekDates(weekId)` para calcular datas dos dias |
| `index.html` | ~555 | CSS: `.plano-progress-wrap`, `.plano-progress-bar`, `.plano-day-check` |
