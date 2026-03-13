# Soluções — Lab 01: Fundamentos DQL

> Use apenas após tentar resolver os exercícios por conta própria.

---

## Exercício 1 — Seu Primeiro Fetch

```dql
fetch logs, from: now() - 1h
| sort timestamp, direction: "descending"
| limit 20
```

---

## Exercício 2 — Filtrar por Nível de Log

```dql
fetch logs, from: now() - 2h
| filter loglevel in ["ERROR", "FATAL"]
| sort timestamp, direction: "descending"
| limit 50
```

**Alternativa com `or`:**
```dql
fetch logs, from: now() - 2h
| filter loglevel == "ERROR" or loglevel == "FATAL"
| sort timestamp, direction: "descending"
| limit 50
```

**Como excluir DEBUG e TRACE:**
```dql
fetch logs, from: now() - 2h
| filterOut loglevel in ["DEBUG", "TRACE"]
| sort timestamp, direction: "descending"
| limit 50
```

---

## Exercício 3 — Busca em Texto Livre

```dql
fetch logs, from: now() - 1h
| filter contains(content, "exception")
| limit 50
```

**Bônus — exception OU timeout:**
```dql
fetch logs, from: now() - 1h
| filter contains(content, "exception") or contains(content, "timeout")
| limit 50
```

---

## Exercício 4 — Combinando Filtros

```dql
fetch logs, from: now() - 1h
| filter loglevel == "ERROR"
| filter contains(content, "database") or contains(content, "connection")
| filter isNotNull(dt.entity.service)
| sort timestamp, direction: "descending"
| limit 30
```

**Alternativa com um único `filter`:**
```dql
fetch logs, from: now() - 1h
| filter loglevel == "ERROR"
    and (contains(content, "database") or contains(content, "connection"))
    and isNotNull(dt.entity.service)
| sort timestamp, direction: "descending"
| limit 30
```

---

## Exercício 5 — Usando `filterOut`

```dql
fetch logs, from: now() - 1h
| filterOut contains(content, "/health")
    or contains(content, "/ping")
    or contains(content, "/readiness")
| filterOut loglevel in ["DEBUG", "TRACE"]
| sort timestamp, direction: "descending"
| limit 100
```

**Alternativa com um único `filterOut`:**
```dql
fetch logs, from: now() - 1h
| filterOut contains(content, "/health")
    or contains(content, "/ping")
    or contains(content, "/readiness")
    or loglevel in ["DEBUG", "TRACE"]
| sort timestamp, direction: "descending"
| limit 100
```

> **Reflexão:** `filterOut` é mais legível quando você precisa excluir uma lista de coisas indesejadas — evita o uso de `not(cond1 or cond2 or ...)` que pode ficar confuso.

---

## Exercício 6 — Match com Wildcard

```dql
// Ajuste o padrão para um nome real do seu ambiente
fetch logs, from: now() - 1h
| filter matchesValue(dt.entity.service_name, "payment*")
| sort timestamp, direction: "descending"
| limit 50
```

**Variações:**
```dql
// Termina com "-prod"
fetch logs, from: now() - 1h
| filter matchesValue(dt.entity.service_name, "*-prod")

// Contém "api" em qualquer posição
fetch logs, from: now() - 1h
| filter matchesValue(dt.entity.service_name, "*api*")
```

---

## Exercício 7 — Controles de Volume

**Parte A:**
```dql
fetch logs, from: now() - 7d, scanLimitGBytes: 0.5
| summarize count = count()
```

**Parte B:**
```dql
fetch logs, from: now() - 7d, scanLimitGBytes: 0.5
| samplingRatio 0.1
| summarize count = count()
```

> O resultado com `samplingRatio: 0.1` será aproximadamente 10% do resultado da Parte A. A diferença mostra que `samplingRatio` é uma **estimativa** — adequado apenas para exploração.

---

## Exercício 8 — Timeseries Básico

**Parte A:**
```dql
timeseries avg_cpu = avg(dt.host.cpu.usage),
    interval: 5m,
    from: now() - 2h
```

**Parte B:**
```dql
timeseries avg_cpu = avg(dt.host.cpu.usage),
    interval: 5m,
    from: now() - 2h,
    by: {dt.entity.host}
```

**Parte C:**
```dql
timeseries {
    avg_cpu = avg(dt.host.cpu.usage),
    p95_cpu = percentile(dt.host.cpu.usage, 95)
},
    interval: 5m,
    from: now() - 2h,
    by: {dt.entity.host}
```

---

## Exercício 9 — Desafio Final do Lab 01

```dql
fetch logs, from: now() - 3h
| filterOut contains(content, "/health")
    or contains(content, "/ping")
    or contains(content, "/readiness")
| filterOut loglevel in ["DEBUG", "INFO"]
| filter loglevel in ["ERROR", "FATAL"]
| filter isNotNull(dt.entity.service)
| sort timestamp, direction: "descending"
| limit 100
| fields timestamp, loglevel, content, dt.entity.service
```

> **Nota:** A ordem dos comandos importa para performance — coloque `filterOut` antes do `filter` quando estiver reduzindo muito volume, e `fields` no final para projetar apenas o que você precisa.
