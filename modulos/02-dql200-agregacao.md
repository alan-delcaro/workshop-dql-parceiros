# Módulo 02 — DQL 200: Agregação, Seleção e Parsing

> **Duração:** ~25 minutos  
> **Objetivo:** Transformar dados brutos em insights — agregar, selecionar campos e extrair estrutura de texto livre

---

## 1. O Comando `summarize`

O `summarize` **agrega** os registros, análogo ao `GROUP BY` + funções de agregação do SQL. Ele reduz N linhas a um conjunto menor de resultados.

### Sintaxe básica

```dql
fetch logs, from: now() - 1h
| summarize count = count()
```

### Com agrupamento (`by:`)

```dql
// Contagem de logs por nível, agrupados
fetch logs, from: now() - 1h
| summarize count = count(), by: {loglevel}
```

```dql
// Agrupamento por múltiplos campos
fetch logs, from: now() - 1h
| summarize count = count(), by: {loglevel, dt.entity.service}
```

### Múltiplas métricas em um único `summarize`

```dql
fetch logs, from: now() - 1h
| filter isNotNull(duration)
| summarize
    total      = count(),
    avg_dur    = avg(duration),
    max_dur    = max(duration),
    p95_dur    = percentile(duration, 95),
    by: {dt.entity.service}
```

### Funções de agregação disponíveis

| Função | Descrição | Exemplo |
|--------|-----------|---------|
| `count()` | Total de registros | `count()` |
| `countDistinct(campo)` | Valores únicos | `countDistinct(user.id)` |
| `countIf(condição)` | Registros que satisfazem condição | `countIf(loglevel == "ERROR")` |
| `avg(campo)` | Média | `avg(duration)` |
| `sum(campo)` | Soma | `sum(bytes_sent)` |
| `min(campo)` | Mínimo | `min(response_time)` |
| `max(campo)` | Máximo | `max(response_time)` |
| `percentile(campo, P)` | Percentil | `percentile(latency, 99)` |
| `collectDistinct(campo)` | Lista de valores únicos | `collectDistinct(error_code)` |
| `takeLast(campo)` | Último valor | `takeLast(status)` |

### Exemplo: Error rate por serviço

```dql
fetch logs, from: now() - 1h
| filter isNotNull(dt.entity.service)
| summarize
    total   = count(),
    errors  = countIf(loglevel == "ERROR"),
    by: {dt.entity.service}
| fields dt.entity.service, total, errors,
         error_rate = round(errors / total * 100, 2)
| sort error_rate, direction: "descending"
```

---

## 2. O Comando `makeTimeseries`

O `makeTimeseries` cria uma **série temporal a partir de qualquer record source** — não apenas métricas. É ideal para visualizar tendências de logs ou eventos ao longo do tempo.

### Diferença em relação ao `timeseries`

| | `timeseries` | `makeTimeseries` |
|-|-------------|-----------------|
| Fonte dos dados | Apenas métricas (dt.host.*, etc.) | Qualquer record source |
| Usa `fetch`? | Não (comando standalone) | Sim (precedido de `fetch`) |
| Ideal para | CPU, memória, métricas OOTB | Contagem de erros, logs, bizevents |

### Sintaxe básica

```dql
fetch logs, from: now() - 2h
| filter loglevel == "ERROR"
| makeTimeseries count = count(), interval: 5m
```

### Com agrupamento

```dql
// Uma série de erro por serviço, buckets de 10 minutos
fetch logs, from: now() - 2h
| filter loglevel == "ERROR"
| makeTimeseries error_count = count(),
    interval: 10m,
    by: {dt.entity.service}
```

### Com diferentes agregações

```dql
// Latência média de spans ao longo do tempo
fetch spans, from: now() - 1h
| filter isNotNull(duration)
| makeTimeseries {
    avg_latency = avg(duration),
    p99_latency = percentile(duration, 99)
  },
    interval: 5m,
    by: {dt.entity.service}
```

---

## 3. O Comando `fields`

O `fields` controla **quais campos** aparecem no resultado e permite criar campos calculados. Equivale ao `SELECT` do SQL.

### Selecionar campos específicos

```dql
fetch logs, from: now() - 1h
| filter loglevel == "ERROR"
| fields timestamp, loglevel, content, dt.entity.service
```

### Renomear campos

```dql
fetch logs, from: now() - 1h
| fields
    ts       = timestamp,
    level    = loglevel,
    message  = content,
    service  = dt.entity.service
```

### Adicionar campos calculados

```dql
fetch logs, from: now() - 1h
| filter isNotNull(duration)
| fields
    timestamp,
    content,
    duration_sec = duration / 1000.0,   // converte ms para segundos
    is_slow      = duration > 5000       // flag booleana
```

### Remover campos (prefixo `-`)

```dql
fetch logs, from: now() - 1h
| fields -trace_id, -span_id, -aws.region   // remove campos desnecessários
```

### `fieldsSummary` — análise exploratória de campos

Muito útil para entender quais campos existem em um log source desconhecido:

```dql
fetch logs, from: now() - 1h
| filter loglevel == "ERROR"
| fieldsSummary
```

Retorna uma tabela com nome dos campos, tipo, contagem de valores não-nulos e exemplos de valores. **Ótimo para explorar dados novos.**

---

## 4. O Comando `expand`

O `expand` "explode" um campo que contém um **array** em múltiplos registros — um por elemento do array.

### Cenário típico

Quando logs ou eventos têm um campo de lista (ex: `tags`, `errors`, `items`):

```dql
// Supondo que o campo "tags" é um array de strings
fetch dt.entity.service
| expand tags
| summarize count = count(), by: {tags}
| sort count, direction: "descending"
| limit 20
```

### Com `expand` e alias

```dql
fetch bizevents, from: now() - 1h
| expand item = cart.items      // cada item do carrinho vira uma linha
| summarize
    total_vendido = sum(item.price),
    qtd_vendida   = count(),
    by: {item.category}
```

### Funções de array (sem expand)

Ao invés de expandir, às vezes você quer inspecionar o array diretamente:

```dql
fetch logs, from: now() - 1h
| filter arraySize(tags) > 0                    // apenas registros com tags
| filter arrayContains(tags, "critical")        // tem a tag "critical"
| fields timestamp, content, tags,
         num_tags = arraySize(tags),
         tags_str = arrayJoin(tags, ", ")       // converte array em string
```

---

## 5. O Comando `parse`

O `parse` extrai **campos estruturados** de um campo de texto livre usando padrões. É o coração da análise de logs.

### Por que usar `parse`?

Logs chegam como texto, mas os dados úteis estão _dentro_ do texto:

```
2024-03-12 10:45:32 ERROR [payment-service] statusCode=500 duration=2345ms userId=usr_7823 endpoint=/api/checkout
```

O `parse` transforma esse texto em colunas que podem ser filtradas, agregadas e analisadas.

### Padrões GROK — os mais comuns

| Padrão | O que captura | Exemplo no log |
|--------|--------------|----------------|
| `DATA` | Qualquer caractere (não-greedy) | Nome, mensagem |
| `LD` | Dados até o próximo delimitador | Valor de campo |
| `INT` | Número inteiro | `42`, `-7` |
| `LONG` | Número inteiro longo | `1234567890` |
| `FLOAT` | Número decimal | `3.14` |
| `WORD` | Sequência sem espaço | `ERROR`, `userId` |
| `SPACE` | Um ou mais espaços | ` ` |
| `TIMESTAMP` | Timestamp ISO | `2024-03-12T10:45:32` |
| `IPADDR` | Endereço IP | `192.168.1.1` |

### Sintaxe básica

```dql
fetch logs, from: now() - 1h
| parse content, "LD 'statusCode=' INT:status_code ' duration=' LONG:duration_ms 'ms'"
| fields timestamp, status_code, duration_ms, content
```

**Como ler o padrão:**
- Texto entre `'...'` é literal — deve aparecer exatamente assim no log
- `LD` captura tudo até o próximo padrão
- `INT:status_code` — captura um inteiro e nomeia o campo `status_code`

### Exemplo completo — parsing de log HTTP

Log original:
```
[2024-03-12 10:45:32] 192.168.1.10 POST /api/checkout 200 1523ms
```

Query:
```dql
fetch logs, from: now() - 1h
| filter contains(content, "POST") or contains(content, "GET")
| parse content, "'[' TIMESTAMP:req_time '] ' IPADDR:client_ip ' ' WORD:method ' ' LD:endpoint ' ' INT:status_code ' ' LONG:duration_ms 'ms'"
| fields req_time, client_ip, method, endpoint, status_code, duration_ms
| filter status_code >= 500
| sort duration_ms, direction: "descending"
```

### Parsing de JSON

Quando o campo de log já é um JSON string:

```dql
fetch logs, from: now() - 1h
| filter startsWith(content, "{")
| parse content, "JSON{user_id: string:user_id, action: string:action, amount: double:amount}"
| fields timestamp, user_id, action, amount
```

### Parsing de chave=valor

```dql
fetch logs, from: now() - 1h
| parse content, "KVP{':','\\n':kvp}"    // key=value pairs separados por nova linha
| fields timestamp, kvp
```

### `parse` como função inline em `fields`

```dql
fetch logs, from: now() - 1h
| fields
    timestamp,
    content,
    parsed = parse(content, "LD 'level=' WORD:level ' msg=' LD:message")
```

> 💡 Teste padrões de parse no [Dynatrace DQL Log Pattern Tester](https://docs.dynatrace.com/docs/shortlink/dql-parse) antes de usar em produção.

---

## 6. Ordenação com `sort`

```dql
fetch logs, from: now() - 1h
| filter loglevel == "ERROR"
| sort timestamp, direction: "descending"    // mais recente primeiro
| limit 50
```

```dql
// Ordenação por múltiplos campos
fetch logs, from: now() - 1h
| summarize count = count(), by: {loglevel, dt.entity.service}
| sort count, direction: "descending"
| sort loglevel, direction: "ascending"
```

---

## 7. Exemplos Integrados

### Exemplo 1: Dashboard de saúde — erros por serviço (últimas 4h)

```dql
fetch logs, from: now() - 4h
| filter loglevel in ["ERROR", "FATAL"]
| filter isNotNull(dt.entity.service)
| summarize
    errors   = count(),
    distinct = countDistinct(exception.type),
    by: {dt.entity.service}
| sort errors, direction: "descending"
| limit 15
```

### Exemplo 2: Tendência de erros por minuto (gráfico de linha)

```dql
fetch logs, from: now() - 2h
| filter loglevel == "ERROR"
| makeTimeseries error_count = count(), interval: 1m
```

### Exemplo 3: Análise de latência de endpoint

```dql
// Extrai método e latência de logs de request e calcula p95 por endpoint
fetch logs, from: now() - 1h
| filter contains(content, "method=")
| parse content, "LD 'method=' WORD:method ' path=' LD:path ' status=' INT:status ' duration=' LONG:latency_ms 'ms'"
| filter isNotNull(latency_ms)
| summarize
    req_count = count(),
    avg_ms    = avg(latency_ms),
    p95_ms    = percentile(latency_ms, 95),
    by: {method, path}
| sort p95_ms, direction: "descending"
| limit 20
```

### Exemplo 4: Análise exploratória de campos desconhecidos

```dql
// O que tem nesses logs? Quais campos existem?
fetch logs, from: now() - 30m
| filter contains(content, "checkout")
| fieldsSummary
```

---

## Resumo DQL 200

| Comando / Função | O que faz |
|-----------------|-----------|
| `summarize agg = func(), by: {campo}` | Agrega dados com agrupamento |
| `countIf(condição)` | Conta registros condicionalmente |
| `countDistinct(campo)` | Conta valores únicos |
| `percentile(campo, P)` | Calcula percentil P |
| `makeTimeseries agg = func(), interval: Xm` | Série temporal de qualquer fonte |
| `fields c1, c2, novo = expr` | Seleciona e calcula campos |
| `fields -campo` | Remove campo do resultado |
| `fieldsSummary` | Análise exploratória de campos |
| `expand campo` | Explode array em múltiplas linhas |
| `arraySize(campo)` | Tamanho do array |
| `arrayContains(campo, val)` | Verifica se array contém valor |
| `parse conteúdo, "PADRÃO"` | Extrai campos de texto livre |
| `sort campo, direction: "desc"` | Ordena resultados |

---

## Próximo Módulo

➡️ [DQL 300 — Enriquecimento e Conexão de Dados](03-dql300-avancado.md)
