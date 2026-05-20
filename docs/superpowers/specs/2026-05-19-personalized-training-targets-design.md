# Design: Personalização de Cargas e Foco da Semana

**Data:** 2026-05-19  
**App:** We Don't Stop — PWA de treino para membros do Team Fontes  
**Público:** Praticantes e sedentários (não atletas de alto nível)  

---

## Problema

Os exercícios nas sessões de treino usam valores fixos (ex: "3 × 12 rep", "3 × 45s") independente dos resultados dos testes do atleta. O plano semanal não reflete os pontos de melhoria identificados na avaliação. Isso enfraquece o incentivo para fazer os testes: se o resultado não muda o treino, por que fazer?

---

## Solução

Duas melhorias complementares:

1. **Cargas personalizadas por sessão** — exercícios mostram alvos calculados a partir dos resultados reais dos testes, com linguagem simples (sem porcentagens, sem jargão técnico).
2. **Banner "Foco da semana"** — topo do plano semanal destaca o pilar mais fraco do atleta com mensagem motivacional.

---

## Arquitetura

### Nova função: `getPersonalizedTargets(aval)`

Função pura colocada logo após `getAvaliacao()` no `index.html`. Centraliza toda a lógica de cálculo de alvos em um único lugar.

**Entrada:** objeto `aval` (pode ser `null` se atleta não fez avaliação)

**Saída:**
```js
{
  pushTarget:     Number | null,  // reps alvo para push-up
  plankTarget:    Number | null,  // segundos alvo para plank
  hikingTarget:   Number | null,  // segundos alvo para hiking bench
  endurancePace:  String | null,  // ex: "6:30/km", "2:10/500m", "2.5 w/kg"
  enduranceRPE:   String,         // ex: "ritmo de conversa"
  weakPillar:     String | null,  // nome do pilar mais fraco, ex: "Core"
}
```

**Regra de fallback:** se o campo de teste correspondente for `null` ou ausente, o campo retorna `null`. Nenhum exercício quebra — o código consumidor mantém o texto genérico atual.

#### Fórmulas de cálculo

| Campo | Fórmula | Mínimo | Exemplo |
|-------|---------|--------|---------|
| `pushTarget` | `round(pushReps × 0.60 / 5) × 5` | 3 reps | 25 reps → 15 |
| `plankTarget` | `round(plankSecs × 0.65 / 5) × 5` | 10s | 45s → 30s |
| `hikingTarget` | `round(hikingSecs × 0.70 / 5) × 5` | 10s | 60s → 40s |
| `endurancePace` | Varia por protocolo (ver abaixo) | — | 2400m → "6:30/km" |
| `weakPillar` | Pilar com menor valor em `aval.scores` | — | core=28 → "Core" |

#### Cálculo de endurance por protocolo

| Protocolo | Cálculo | Formato de saída |
|-----------|---------|-----------------|
| Cooper 12min | Pace teste (s/km) × 1.30 → formata em "M:SS/km" | `"6:30/km"` (ex: 2400m → pace 5:00/km × 1.30 = 6:30/km) |
| 2km Remo | Pace/500m do teste (s) × 1.15 → formata em "M:SS/500m" | `"2:10/500m"` |
| FTP Bike | FTP (w/kg) × 0.65 → arredonda para 1 decimal | `"2.5 w/kg"` |
| 100 Burpees | 100 / tempo_total_min × 0.65 → arredonda | `"7 burpees/min"` |

`enduranceRPE` é sempre `"ritmo de conversa"` — independente do protocolo.

#### Cálculo do pilar mais fraco

- Lê `aval.scores` = `{ hiking, core, forca, endurance, mobilidade }`
- Para atletas não-velejadores (esporte ≠ 'velejador'), exclui `hiking` do ranking
- Retorna o nome de display do pilar com menor score
- Mapa de nomes: `{ hiking: "Hiking Bench", core: "Core", forca: "Força", endurance: "Endurance", mobilidade: "Mobilidade" }`
- Retorna `null` se `aval.scores` não existir
- Em caso de empate de score, prioridade: hiking > core > forca > endurance > mobilidade

---

### Mudança em `getSessionSections(type, aval, cardioMode)`

Chama `getPersonalizedTargets(aval)` no início da função e usa os campos para substituir os valores `detail` dos exercícios relevantes.

**Exercícios afetados:**

| Exercício | Texto atual | Texto novo |
|-----------|-------------|------------|
| Push-up | `"3 × 12 rep"` | `"3 × ${pushTarget} rep"` (ou mantém `"3 × 12 rep"` se `null`) |
| Push-up (joelho) | `"2 × 8 rep"` | `"2 × ${pushTarget} rep"` (mesmo alvo, escala com teste) |
| Prancha frontal | `"3 × 45s"` | `"3 × ${plankTarget}s"` |
| Hollow hold | `"4 × 30s"` | `"4 × ${plankTarget}s"` |
| Hiking bench isométrico | `"3 × máximo"` | `"3 × ${hikingTarget}s"` |
| Wall sit | `"2 × 60s"` | mantém fixo (não tem teste direto) |
| Cardio Z2 (detail) | `"30 min"` | `"30 min · ~${endurancePace}"` |
| Cardio Z2 (rest) | `""` | `"${enduranceRPE}"` (campo `rest` usado como subtexto de ritmo) |
| Prancha lateral | qualquer | mantém fixo — exercício diferente do teste de plank frontal |

**Regra de substituição:** se o alvo for `null` (teste não feito), mantém exatamente o texto atual. Nenhuma regressão.

**Nota sobre séries:** apenas o valor de tempo/reps muda. O número de séries (ex: `"3 ×"`) permanece conforme definido pelo tier L/M/H.

---

### Mudança em `getPlanWeeks(aval)`

- Chama `getPersonalizedTargets(aval)` uma vez no início
- Adiciona `weakPillar: targets.weakPillar` a cada objeto de semana retornado no array `META`

---

### Mudança em `TelaPlano`

Adiciona banner logo após o bloco `.plano-summary`, condicionado a `w.weakPillar != null`.

**Layout do banner:**
```
┌─────────────────────────────────────┐
│  FOCO  ·  Core                      │
│  Reforço incluído em cada sessão    │
└─────────────────────────────────────┘
```

**CSS:** fundo branco, borda `1px solid rgba(24,25,31,0.08)`, border-radius 16px, padding 14px 16px. Overline "FOCO" em laranja (`var(--orange)`). Texto neutro. Sem ícone de alerta.

---

## Fluxo de dados

```
getAvaliacao()
    └── getPersonalizedTargets(aval)
            ├── → getSessionSections()  →  exercícios com alvos numéricos
            └── → getPlanWeeks()        →  semanas com weakPillar
                        └── → TelaPlano  →  banner "Foco da semana"
```

---

## Fora de escopo

- Progressão automática de cargas ao longo das semanas (fase futura)
- Push-ups negativos / variações avançadas
- Personalização de Wall sit e Dead Hang (sem teste direto correspondente)
- Prancha lateral (exercício diferente do teste de plank frontal — não personalizado)
- Notificação quando os testes ficam desatualizados (fase futura)

---

## Critérios de sucesso

1. Atleta que fez todos os testes vê alvos numéricos personalizados em todo exercício mapeado
2. Atleta que não fez nenhum teste vê exatamente o mesmo app de hoje (zero regressão)
3. Banner "Foco da semana" aparece apenas para quem tem scores registrados
4. Linguagem simples: nenhum número técnico exposto, só o alvo final
