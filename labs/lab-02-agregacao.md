# Lab 02 — Agregação, Seleção e Parsing (DQL 200)

> **Foco:** `summarize`, `makeTimeseries`, `fields`, `fieldsSummary`, `expand`, `parse`  
> **Duração estimada:** 35–45 minutos  
> **Pré-requisito:** [Lab 01](lab-01-fundamentos.md) concluído

---

## Checkpoint Inicial

Execute esta query para garantir que você tem dados suficientes:

```dql
fetch logs, from: now() - 1h
| summarize count = count(), by: {loglevel}
| sort count, direction: "descending"
```

Se tiver dados, ótimo. Se não, amplie a janela para `now() - 6h` ou `now() - 24h`.

---

## Exercício 1 — Minha Primeira Agregação

**Objetivo:** Usar `summarize` para agregar dados.

**Tarefa:** Contar **quantos logs existem por nível** (`loglevel`) na última hora. Ordene do mais frequente para o menos frequente.

> 💡 Dica: `summarize count = count(), by: {campo}`.

**Sua solução:**
```dql
// Escreva aqui

```

**Pergunta:** Qual é o nível de log mais comum no seu ambiente? Faz sentido?

---

## Exercício 2 — Múltiplas Métricas em Um `summarize`

**Objetivo:** Calcular várias métricas de uma vez.

**Tarefa:** Para logs que possuem o campo `duration` (ou equivalente de duração no seu ambiente), calcule por serviço:
- Total de registros
- Duração média
- Duração máxima
- Percentil 95 da duração

Ordene pelos serviços com maior P95.

> 💡 Dica: use `isNotNull(duration)` para filtrar apenas registros com duração.

**Sua solução:**
```dql
// Escreva aqui

```

---

## Exercício 3 — Error Rate por Serviço

**Objetivo:** Usar `countIf` para contagem condicional.

**Tarefa:** Calcule o **error rate** por serviço (`dt.entity.service`) nas últimas 2 horas:
- Total de logs
- Quantidade de erros (loglevel == "ERROR")
- Error rate em porcentagem, arredondada com 2 casas decimais

Mostre apenas serviços com `error_rate > 0`. Ordene pelo maior error rate.

> 💡 Dica: `countIf(loglevel == "ERROR")` conta apenas os registros que satisfazem a condição.

**Sua solução:**
```dql
// Escreva aqui

```

---

## Exercício 4 — Seleção e Cálculo de Campos

**Objetivo:** Usar `fields` para criar campos calculados.

**Parte A:** Busque logs de erro da última hora e mostre **apenas** os campos:
- `timestamp`
- `loglevel`
- Uma versão truncada do conteúdo (ultimos 200 chars se disponível)
- `dt.entity.service`

```dql
// Parte A

```

**Parte B:** A partir do Exercício 3, adicione um campo calculado `health_status`:
- `"CRITICAL"` se error_rate > 5%
- `"DEGRADED"` se error_rate entre 1% e 5%
- `"HEALTHY"` se error_rate < 1%

> 💡 Dica: use `if(condição, "valor_true", if(condição2, "valor2", "valor3"))`.

```dql
// Parte B — construa sobre o resultado do Exercício 3

```

---

## Exercício 5 — `fieldsSummary` — Exploração de Schema

**Objetivo:** Descobrir campos disponíveis em um record source desconhecido.

**Tarefa:** Execute `fieldsSummary` nos seus logs de erro e responda:

1. Quais campos de **serviço** existem? (ex: `dt.entity.service`, `service.name`, etc.)
2. Existe algum campo de **exception**? Qual é o nome exato?
3. Existe algum campo de **duração** ou **latência**?

```dql
// Exploração com fieldsSummary
fetch logs, from: now() - 1h
| filter loglevel == "ERROR"
| fieldsSummary
```

**Anote os campos que você vai usar nos próximos exercícios:**
- Campo de serviço: `________________`
- Campo de exception: `________________`
- Campo de duração: `________________`

---

## Exercício 6 — Série Temporal de Erros

**Objetivo:** Usar `makeTimeseries` para criar gráfico de tendência.

**Parte A:** Crie uma série temporal com a **contagem de erros por minuto** nas últimas 2 horas.

```dql
// Parte A

```

**Parte B:** Adicione agrupamento por serviço (`by: {dt.entity.service}`) para ver a tendência de cada serviço separadamente.

```dql
// Parte B

```

**Parte C (Desafio):** Crie uma série temporal que mostre tanto erros quanto warnings lado a lado:

> 💡 Dica: use `countIf` dentro do `makeTimeseries`.

```dql
// Parte C

```

---

## Exercício 7 — Parse de Log Estruturado

**Objetivo:** Extrair campos de log de texto livre com `parse`.

**Contexto:** Muitas aplicações emitem logs assim:

```
2024-03-12T10:45:32Z level=ERROR service=payment-api status=500 duration=2345 msg=Payment gateway timeout
```

**Tarefa:** Ajuste a query abaixo para os seus logs reais. Se seus logs têm formato diferente, adapte o padrão:

```dql
fetch logs, from: now() - 1h
| filter contains(content, "duration=")   // ajuste para um campo existente
| parse content, "TIMESTAMP:req_ts ' level=' WORD:level ' service=' WORD:svc ' status=' INT:status_code ' duration=' LONG:dur_ms ' msg=' LD:message"
| filter isNotNull(status_code)
| fields req_ts, level, svc, status_code, dur_ms, message
| sort dur_ms, direction: "descending"
| limit 20
```

> 💡 Se seus logs têm um formato diferente, use `fieldsSummary` para entender a estrutura e ajuste o padrão `parse` de acordo.

**Sua solução adaptada:**
```dql
// Versão adaptada para seus logs

```

---

## Exercício 8 — Analisando Arrays com `expand`

**Objetivo:** Usar `expand` para trabalhar com campos de array.

**Tarefa:** Se você tem entidades Dynatrace com tags:

```dql
// Expanda e conte quantas entidades têm cada tag
fetch dt.entity.service
| expand tags
| filter isNotNull(tags)
| summarize service_count = count(), by: {tags}
| sort service_count, direction: "descending"
| limit 20
```

Execute a query e interprete o resultado. Quais as tags mais comuns?

**Bônus:** Use `arrayContains` ao invés de `expand` para verificar se uma entidade tem uma tag específica:

```dql
// Serviços com a tag "production" — adapte o nome da tag
fetch dt.entity.service
| filter arrayContains(tags, "production")
| fields entity.id, entity.name, tags
| limit 20
```

---

## Exercício 9 — Pipeline Completo DQL 200

**Objetivo:** Construir um pipeline que combina todas as habilidades do laboratório.

**Tarefa:** Crie uma query de análise de performance de endpoints que:

1. Busque logs das últimas **2 horas**
2. Filtre apenas logs que contêm informações de request HTTP (adapte ao pattern do seu ambiente)
3. Use `parse` para extrair: method, path, status_code, duration_ms
4. Filtre apenas registros onde o parse teve sucesso (campos não nulos)
5. Use `summarize` para calcular por `(method, path)`:
   - Total de requests
   - Requests com erro (status >= 500)
   - P95 de duração
6. Adicione um campo `slo_violated` = `true` se P95 > 1000ms
7. Ordene por P95 decrescente
8. Mostre top 15

```dql
// Pipeline completo — adapte os campos ao seu ambiente

```

---

## Checklist de Conclusão

- [ ] Criar agregações com `summarize` e múltiplas funções (count, avg, percentile)
- [ ] Usar `countIf` para contagem condicional
- [ ] Criar campos calculados com `fields` e `if()`
- [ ] Usar `fieldsSummary` para exploração
- [ ] Criar série temporal com `makeTimeseries` e agrupamento
- [ ] Extrair campos de log com `parse`
- [ ] Trabalhar com arrays usando `expand` e `arrayContains`

✅ **Quando estiver pronto:** [Lab 03 — Avançado](lab-03-avancado.md)

🔍 **Gabarito:** [Soluções do Lab 02](../solucoes/lab-02-solucoes.md)
