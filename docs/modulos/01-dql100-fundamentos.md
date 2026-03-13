# Módulo 01 — DQL 100: Fundamentos

> **Duração:** ~25 minutos  
> **Objetivo:** Dominar os comandos básicos para buscar, filtrar e limitar dados no Grail

---

## 1. O Comando `fetch`

O `fetch` é o **ponto de partida** de quase toda query DQL. Ele define qual record source você quer consultar.

### Sintaxe básica

```dql
fetch logs
```

```dql
fetch metrics
```

```dql
fetch spans
```

```dql
fetch bizevents
```

### Especificando janela de tempo

Por padrão, o `fetch` usa a janela global do Notebook. Para query independente, use `from:` e opcionalmente `to:`:

```dql
// Últimas 2 horas
fetch logs, from: now() - 2h

// Últimos 30 minutos
fetch logs, from: now() - 30m

// Últimos 7 dias
fetch logs, from: now() - 7d

// Intervalo específico
fetch logs, from: now() - 4h, to: now() - 1h
```

**Unidades de tempo suportadas:**
| Sufixo | Significado |
|--------|-------------|
| `s` | segundos |
| `m` | minutos |
| `h` | horas |
| `d` | dias |
| `w` | semanas |

> 💡 Sempre defina `from:` em queries de produção para garantir resultados previsíveis, independente do contexto do Notebook.

---

## 2. O Comando `filter`

O `filter` mantém apenas os registros que **satisfazem** a condição. É o equivalente ao `WHERE` do SQL.

### Operadores de comparação

```dql
fetch logs, from: now() - 1h
| filter loglevel == "ERROR"          // igual
| filter status != "ok"               // diferente
| filter duration > 1000              // maior que
| filter duration >= 500              // maior ou igual
| filter timestamp < now() - 30m     // menor que
```

### Operadores lógicos

```dql
// AND implícito (múltiplos filter em sequência)
fetch logs, from: now() - 1h
| filter loglevel == "ERROR"
| filter dt.entity.service != null

// AND explícito em um único filter
fetch logs, from: now() - 1h
| filter loglevel == "ERROR" and dt.entity.service != null

// OR
fetch logs, from: now() - 1h
| filter loglevel == "ERROR" or loglevel == "WARN"

// NOT
fetch logs, from: now() - 1h
| filter not(loglevel == "DEBUG")

// IN — verifica se o valor está em uma lista
fetch logs, from: now() - 1h
| filter loglevel in ["ERROR", "WARN", "CRITICAL"]
```

### Verificando nulos

```dql
fetch logs, from: now() - 1h
| filter isNotNull(dt.entity.service)

fetch logs, from: now() - 1h
| filter isNull(exception.type)
```

---

## 3. O Comando `filterOut`

O `filterOut` **exclui** os registros que satisfazem a condição — é o inverso do `filter`.

```dql
// Excluir logs de saúde (healthcheck) do resultado
fetch logs, from: now() - 1h
| filterOut contains(content, "/health") or contains(content, "/ping")

// Remover logs de infrequência e nível baixo
fetch logs, from: now() - 1h
| filterOut loglevel in ["DEBUG", "TRACE", "INFO"]
```

> 💡 Use `filterOut` quando for mais natural descrever o que você **não quer** do que o que quer. Frequentemente melhora legibilidade.

---

## 4. Funções de Match em Strings

Para campos de texto livre (como `content` de logs), o DQL oferece funções de correspondência específicas:

### `contains(campo, "valor")`

Verifica se o campo **contém** a substring. Simples e direto.

```dql
fetch logs, from: now() - 1h
| filter contains(content, "NullPointerException")
```

### `matchesValue(campo, "padrão")`

Verifica correspondência **exata** de valor, com suporte a wildcards (`*` e `?`).

```dql
// Exato
fetch logs, from: now() - 1h
| filter matchesValue(loglevel, "ERROR")

// Com wildcard — qualquer serviço que começa com "payment"
fetch logs, from: now() - 1h
| filter matchesValue(dt.entity.service_name, "payment*")

// Com wildcard — qualquer exception que termina com "Exception"
fetch logs, from: now() - 1h
| filter matchesValue(exception.type, "*Exception")
```

### `matchesPhrase(campo, "frase")`

Busca uma **frase completa** dentro de um campo de texto, respeitando delimitadores de palavra. É mais preciso que `contains` para frases.

```dql
fetch logs, from: now() - 1h
| filter matchesPhrase(content, "connection refused")
// ✅ encontra "connection refused to database"
// ❌ não encontra "connectionRefused" (sem espaço)
```

### `startsWith` e `endsWith`

```dql
fetch logs, from: now() - 1h
| filter startsWith(content, "ERROR:")

fetch logs, from: now() - 1h
| filter endsWith(dt.entity.service_name, "-prod")
```

### Quadro comparativo

| Função | Wildcards | Case-sensitive | Uso ideal |
|--------|-----------|----------------|-----------|
| `contains` | Não | Não | Substring simples |
| `matchesValue` | `*` e `?` | Não | Match exato ou com padrão |
| `matchesPhrase` | Não | Não | Frases em texto livre |
| `startsWith` / `endsWith` | Não | Não | Prefixo / sufixo |

---

## 5. Controles de Volume

O Grail scanneia dados para responder queries. Para queries exploratórias, limite o escopo:

### `limit`

Retorna no máximo N registros. Análogo ao `LIMIT` do SQL.

```dql
fetch logs, from: now() - 1h
| filter loglevel == "ERROR"
| limit 100
```

> ⚠️ O `limit` no DQL é aplicado **após o pipeline** — a query ainda scanneia todos os dados, apenas trunca a saída. Use `scanLimitGBytes` para limitar o scan real.

### `samplingRatio`

Retorna uma amostra aleatória dos dados, de 0 a 1 (0% a 100%).

```dql
// Analisa apenas 10% dos registros — útil para exploração rápida
fetch logs, from: now() - 1h
| filter loglevel == "ERROR"
| samplingRatio 0.1
| summarize count = count(), by: {dt.entity.service}
```

> ⚠️ Resultados com `samplingRatio < 1` são **aproximações**. Nunca use em alertas ou relatórios de produção.

### `scanLimitGBytes`

Limita quantos GB de dados o Grail vai scanner. Lança erro se o limite for atingido antes de completar.

```dql
// Scanneia no máximo 1 GB — seguro para exploração
fetch logs, from: now() - 7d, scanLimitGBytes: 1
| filter loglevel == "ERROR"
| summarize count = count(), by: {dt.entity.service}
```

> 💡 **Boas práticas de performance:**
> 1. Sempre especifique `from:` com o menor intervalo necessário
> 2. Use `filter` o mais cedo possível no pipeline (antes de `summarize`)
> 3. Use `scanLimitGBytes` em queries experimentais de longa janela de tempo

---

## 6. O Comando `timeseries`

O `timeseries` é o comando especializado para consultar **métricas ao longo do tempo**. Diferente do `fetch metrics`, ele agrega automaticamente em buckets de tempo.

### Sintaxe básica

```dql
timeseries avg_cpu = avg(dt.host.cpu.usage)
```

```dql
timeseries {
    avg_cpu    = avg(dt.host.cpu.usage),
    max_cpu    = max(dt.host.cpu.usage),
    p95_cpu    = percentile(dt.host.cpu.usage, 95)
}
```

### Especificando intervalo e janela

```dql
timeseries avg_cpu = avg(dt.host.cpu.usage),
    interval: 5m,           // bucket de 5 minutos
    from: now() - 2h        // últimas 2 horas
```

### Agrupando por entidade

```dql
// CPU por host — uma série para cada host
timeseries avg_cpu = avg(dt.host.cpu.usage),
    interval: 5m,
    from: now() - 2h,
    by: {dt.entity.host}
```

### Filtrando métricas por atributos

```dql
// CPU apenas dos hosts com tag "production"
timeseries avg_cpu = avg(dt.host.cpu.usage),
    filter: {tags contains "production"},
    interval: 5m,
    from: now() - 2h
```

### Funções de agregação disponíveis no `timeseries`

| Função | Descrição |
|--------|-----------|
| `avg(métrica)` | Média |
| `sum(métrica)` | Soma |
| `min(métrica)` | Mínimo |
| `max(métrica)` | Máximo |
| `percentile(métrica, p)` | Percentil P (ex: 95, 99) |
| `count()` | Contagem de amostras |

---

## 7. Exemplos Integrados

### Exemplo 1: Erros recentes com contexto

```dql
// Top 50 logs de erro da última hora, com campos relevantes
fetch logs, from: now() - 1h
| filter loglevel == "ERROR"
| filter isNotNull(dt.entity.service)
| filterOut contains(content, "health") or contains(content, "readiness")
| sort timestamp, direction: "descending"
| limit 50
```

### Exemplo 2: Análise de exceções Java

```dql
// Logs contendo exceções Java, agrupados por tipo de exceção
fetch logs, from: now() - 2h
| filter matchesValue(exception.type, "*Exception")
| summarize count = count(), by: {exception.type}
| sort count, direction: "descending"
| limit 20
```

### Exemplo 3: Tendência de CPU dos hosts de produção

```dql
// Média e P95 de CPU dos hosts de produção a cada 10 minutos
timeseries {
    avg_cpu = avg(dt.host.cpu.usage),
    p95_cpu = percentile(dt.host.cpu.usage, 95)
},
    filter: {tags contains "production"},
    interval: 10m,
    from: now() - 6h
```

### Exemplo 4: Combinando filtros

```dql
// Logs de erro com keywords de banco de dados das últimas 4h
fetch logs, from: now() - 4h, scanLimitGBytes: 2
| filter loglevel in ["ERROR", "FATAL"]
| filter contains(content, "database")
    or contains(content, "connection pool")
    or matchesPhrase(content, "query timeout")
| fields timestamp, loglevel, content, dt.entity.service
| sort timestamp, direction: "descending"
| limit 100
```

---

## Resumo DQL 100

| Comando / Função | O que faz |
|-----------------|-----------|
| `fetch <source>` | Busca dados de uma fonte |
| `from: now() - Xh` | Define a janela de tempo |
| `filter <condição>` | Mantém registros que passam na condição |
| `filterOut <condição>` | Remove registros que satisfazem a condição |
| `contains(campo, "valor")` | Substring simples |
| `matchesValue(campo, "padrão")` | Match com wildcard |
| `matchesPhrase(campo, "frase")` | Frase em texto livre |
| `limit N` | Trunca o resultado em N linhas |
| `samplingRatio N` | Amostragem de N% dos dados |
| `scanLimitGBytes N` | Limita o scan a N GB |
| `timeseries agg = func(métrica)` | Série temporal de métricas |

---

## Próximo Módulo

➡️ [DQL 200 — Agregação, Seleção e Parsing](02-dql200-agregacao.md)
