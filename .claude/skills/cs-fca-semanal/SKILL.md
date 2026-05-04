---
name: cs-fca-semanal
description: Gera a análise comparativa semanal de FCA (Fato/Causa/Ação) dos squads de CS. Produz uma tabela agregada de flags entre as duas semanas mais recentes (todos os squads somados) e uma análise individual por squad com a tabela de flags, lista de clientes que mudaram de flag com motivos, e o principal gargalo (dimensão crítica + cliente em alerta máximo). Invocada via /cs-fca-semanal ou quando o usuário pede "análise FCA semanal", "comparativo FCA dos squads", "FCA da semana", e variantes.
area: cs
author: hellenoliveira-sys
version: 1.0.0
---

# Análise FCA Semanal — Skill

Você gera o relatório semanal de FCA (Fato/Causa/Ação) consolidado dos squads de CS.

## 1. Coleta dos dados

A usuária mantém um sheet (ou print) por squad e atualiza manualmente toda semana — não há automação. Conduza a coleta assim:

1. Pergunte: **"Quais squads vamos analisar nessa rodada?"** Liste os squads e seus coordenadores.
2. Para cada squad, peça os dados. Aceite qualquer formato:
   - CSV / texto tabular colado no chat
   - Link público de Google Sheet (use WebFetch no formato `https://docs.google.com/spreadsheets/d/{ID}/export?format=csv`)
   - Screenshot / print (leia visualmente)
3. Confirme que cada bloco contém as **duas datas mais recentes** antes de processar. Se só veio uma data, peça a anterior.

Não tente buscar o sheet de exemplo do Falcon que está na memória de referência sem confirmar antes — a usuária agora tem sheets separados por squad.

## 2. Estrutura esperada dos dados

Cada linha = (Cliente, Data) snapshot. Colunas usadas no cálculo:

- `Data`, `Cliente`, `Coordenador`, `Flag Calculada`
- 5 dimensões que originam a flag:
  - `Resultados` — nota 0–10 (≤6 = ruim)
  - `Ops tráfego` — TRUE/FALSE
  - `Entregas prazo` — TRUE/FALSE
  - `Entregas qualidade` — TRUE/FALSE
  - `Relacionamento` — TRUE/FALSE
- `Observação do Health Score` — texto livre (use no campo Motivos quando relevante)

A `Flag Calculada` já vem pronta. **Não recalcule** — só compare entre as duas semanas.

**Hierarquia das flags (melhor → pior):** Safe > Care > Danger > Critical. Critical é o pior estado.

## 3. Saída

A saída segue exatamente esta estrutura, em português:

### Bloco 1 — Comparativo geral (todos os squads agregados)

```
Comparativo Geral — Sem DD/MM vs Sem DD/MM

Flag       Sem DD/MM    Sem DD/MM    Variação
Safe       X            Y            ±N
Critical   X            Y            ±N
Care       X            Y            ±N
Danger     X            Y            ±N

Total      X            Y            = (ou ±N se diferente)
```

### Bloco 2 — Análise individual por squad

Repita para cada squad (na ordem em que a usuária forneceu):

```
[Nome do Squad]

Flag       Sem DD/MM    Sem DD/MM    Variação
Safe       X            Y            ±N
Critical   X            Y            ±N
Care       X            Y            ±N
Danger     X            Y            ±N

Total      X            Y            = (ou ±N)

Com base na comparação entre os dias [data anterior por extenso] e
[data atual por extenso], identifiquei [N] clientes que tiveram alteração
na sua Flag Calculada.

- NOME DO CLIENTE EM CAIXA ALTA
  - Mudança: De [Flag anterior] para [Flag atual].
  - Motivos: [explicação analítica citando dimensões que mudaram, com valores
    específicos. Se Observação do Health Score for relevante, incorpore.
    Quando uma dimensão melhorou mas a flag piorou (ou vice-versa), explique
    o trade-off.]

(repetir por cliente, em ordem de severidade da mudança: pioras grandes
primeiro — ex. Safe→Critical —, depois pioras menores, depois melhoras)

Principal gargalo do [Squad]:
- Dimensão crítica: [dimensão] — N de M clientes com falha na semana atual (X%).
  Comparação semanal: na semana anterior eram K clientes (Y%) — [piorou / melhorou / permaneceu igual].
- Cliente em alerta máximo: [NOME] — [flag anterior] → [flag atual].
  Comparação semanal: [se mudou de flag, descreva o salto e a principal dimensão por trás;
  se nenhum cliente piorou na semana, escreva "Nenhum cliente piorou — situação permaneceu
  igual à semana anterior" e indique se a flag pior do squad continua sendo a mesma de antes].
```

## 4. Como calcular o "principal gargalo" (combinação a + b)

### a) Dimensão crítica
Na **semana atual**, conte para cada uma das 5 dimensões quantos clientes do squad estão "ruins":
- Resultados ≤ 6 → conta como falha
- Ops tráfego = FALSE → falha
- Entregas prazo = FALSE → falha
- Entregas qualidade = FALSE → falha
- Relacionamento = FALSE → falha

A dimensão com a **maior taxa de falha** é o gargalo dimensional do squad. Em caso de empate, escolha a dimensão que mais piorou em relação à semana anterior. Se ainda houver empate, escolha na ordem: Resultados > Ops tráfego > Relacionamento > Entregas prazo > Entregas qualidade.

### b) Cliente em alerta máximo
O cliente cuja flag mais piorou entre as duas semanas. Use a hierarquia numérica (Safe=1, Care=2, Danger=3, Critical=4):
- Maior delta positivo = piora maior. Ex.: Safe→Critical (+3) é pior que Care→Danger (+1).
- Em caso de empate, prefira:
  1. Maior número de dimensões que pioraram
  2. Maior queda absoluta em Resultados
- Se **nenhum cliente piorou** na semana, escreva no campo Cliente em alerta máximo: "Nenhum cliente piorou — situação permaneceu igual à semana anterior" e indique qual cliente continua na pior flag do squad (se houver), pra manter monitoramento.

## 5. Regras

- Use sempre os valores reais (TRUE/FALSE, nota 0–10). Não invente.
- Não classifique flags por conta própria — a Flag Calculada vem pronta na base.
- Não invente dimensões fora das 5 listadas.
- Se a usuária forneceu dados de só 1 squad, gere mesmo assim a tabela comparativa geral (será igual à do squad).
- Antes de entregar a saída final, confira que o total de clientes em "semana anterior" e "semana atual" bate com o total na tabela comparativa geral — se não bater, há cliente que entrou ou saiu da base; sinalize isso explicitamente no final.
- Saída em português. Sem emojis.
