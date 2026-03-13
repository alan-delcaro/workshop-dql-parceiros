# Soluções — Lab 03: Avançado (Enriquecimento e Conexão de Dados)

> Use apenas após tentar resolver os exercícios por conta própria.

---

## Exercício 1 — Explorando o Modelo de Entidades

**Parte A — Hosts:**
```dql
fetch dt.entity.host
| fields
    entity.id,
    entity.name,
    os_type      = entity.osType,
    state        = lifecycleState
| sort entity.name
| limit 30
```

**Parte B — Serviços ativos:**
```dql
fetch dt.entity.service
| filter lifecycleState == "ENABLED"
| fields entity.id, entity.name, entity.type, tags, lifecycleState
| sort entity.name
| limit 50
```

**Parte C — fieldsSummary:**
```dql
fetch dt.entity.service
| limit 50
| fieldsSummary
```

---

## Exercício 2 — `entityAttr` para Resolver Nomes

```dql
fetch logs, from: now() - 1h
| filter loglevel == "ERROR"
| filter isNotNull(dt.entity.service)
| fields
    timestamp,
    loglevel,
    content,
    service_name = entityAttr(dt.entity.service, "entity.name"),
    host_name    = entityAttr(dt.entity.host, "entity.name")
| sort timestamp, direction: "descending"
| limit 30
```

> **Comparação:** Sem `entityAttr`, o campo `dt.entity.service` mostra `SERVICE-ABCD1234` — inútil para um dashboard executivo. Com `entityAttr`, você vê `payment-service-prod`.

---

## Exercício 3 — `lookup` para Enriquecer com Metadados de Serviço

```dql
fetch logs, from: now() - 1h
| filter isNotNull(dt.entity.service)
| summarize
    total   = count(),
    errors  = countIf(loglevel == "ERROR"),
    by: {service_id = dt.entity.service}

| lookup [
    fetch dt.entity.service
    | fields entity.id, entity.name, lifecycleState, tags
  ],
  sourceField: service_id,
  lookupField: entity.id

| fields
    entity.name,
    lifecycleState,
    total,
    errors,
    error_rate = round(errors / total * 100, 2)

| sort error_rate, direction: "descending"
```

> **Por que `lookup` ao invés de `entityAttr`?**  
> `entityAttr` é ideal para 1 campo. `lookup` permite trazer múltiplos campos (`name`, `lifecycleState`, `tags`) em uma única operação e é mais flexível — você pode filtrar e transformar a subquery de lookup antes de unir.

---

## Exercício 4 — `lookup` com `data`

```dql
// Erros por serviço
fetch logs, from: now() - 2h
| filter loglevel == "ERROR"
| filter isNotNull(dt.entity.service)
| summarize errors = count(), by: {service_id = dt.entity.service}

// Enriquece com nome real
| lookup [
    fetch dt.entity.service
    | fields entity.id, entity.name
  ],
  sourceField: service_id,
  lookupField: entity.id

// Enriquece com criticidade definida pelo time
| lookup [
    data record(service: "payment-service-prod",  criticality: "HIGH"),
         record(service: "order-service-prod",    criticality: "HIGH"),
         record(service: "inventory-service",     criticality: "MEDIUM"),
         record(service: "notification-service",  criticality: "LOW")
         // adicione quantos serviços fizer sentido no seu ambiente
  ],
  sourceField: entity.name,
  lookupField: service

| fields entity.name, errors, criticality
| sort errors, direction: "descending"
```

> **Nota:** Se o `entity.name` não bater exatamente com os valores no `data`, o campo `criticality` virá como `null`. Use `isNotNull(criticality)` para filtrar apenas os mapeados, ou adicione um valor padrão com `if(isNull(criticality), "UNKNOWN", criticality)`.

---

## Exercício 5 — `append` para Unir Fontes de Dados

```dql
// Logs de ERROR
fetch logs, from: now() - 7d
| filter loglevel == "ERROR"
| fields
    timestamp,
    event_type  = "LOG_ERROR",
    description = content,
    service_id  = dt.entity.service

| append [
    // Events de deployment
    fetch events, from: now() - 7d
    | filter event.type == "DEPLOYMENT"
    | fields
        timestamp,
        event_type  = "DEPLOYMENT",
        description = event.name,
        service_id  = dt.entity.service
  ]

| fields
    timestamp,
    event_type,
    description,
    service = entityAttr(service_id, "entity.name")
| sort timestamp, direction: "descending"
| limit 100
```

> **Quando usar `append`:** quando você quer **combinar linhas** de fontes diferentes na mesma tabela (UNION). Use `join` quando você quer **combinar colunas** cruzando chaves (JOIN).

---

## Exercício 6 — `join` para Calcular Métricas Comparativas

```dql
// Dataset A: contagem de erros por serviço
fetch logs, from: now() - 1h
| filter loglevel == "ERROR"
| summarize errors = count(), by: {dt.entity.service}

| join [
    // Dataset B: total de logs por serviço
    fetch logs, from: now() - 1h
    | summarize total = count(), by: {dt.entity.service}
  ],
  on: {dt.entity.service},
  kind: left

| fields
    service_id  = dt.entity.service,
    errors,
    total,
    error_pct   = round(errors / total * 100, 2),
    // Bônus: nome real
    service     = entityAttr(dt.entity.service, "entity.name")

| sort error_pct, direction: "descending"
| limit 15
```

---

## Exercício 7 — `join` com `timeseries`

```dql
// Timeseries de CPU por host (últimas 2h, buckets de 30m)
timeseries avg_cpu = avg(dt.host.cpu.usage),
    by: {dt.entity.host},
    interval: 30m,
    from: now() - 2h
| filter avg_cpu > 60
| fields host_id = dt.entity.host, avg_cpu

| join [
    // Erros desses hosts no mesmo período
    fetch logs, from: now() - 2h
    | filter loglevel == "ERROR"
    | filter isNotNull(dt.entity.host)
    | summarize error_count = count(), by: {host_id = dt.entity.host}
  ],
  on: {host_id},
  kind: left

| fields
    host        = entityAttr(host_id, "entity.name"),
    avg_cpu,
    error_count,
    correlation = if(error_count > 50 and avg_cpu > 80, "LIKELY_CORRELATED",
                  if(isNull(error_count), "NO_ERRORS_FOUND",
                  "POSSIBLE_CORRELATION"))

| sort avg_cpu, direction: "descending"
```

---

## Exercício 8 — Pipeline Mestre (Desafio Final)

```dql
// Service Health Dashboard — última 1 hora
fetch logs, from: now() - 1h
| filter isNotNull(dt.entity.service)
| summarize
    total_logs  = count(),
    errors      = countIf(loglevel == "ERROR"),
    warnings    = countIf(loglevel == "WARN"),
    by: {service_id = dt.entity.service}

| lookup [
    fetch dt.entity.service
    | fields entity.id, entity.name, lifecycleState
  ],
  sourceField: service_id,
  lookupField: entity.id

| filter isNotNull(entity.name)

| fields
    service_name  = entity.name,
    lifecycle     = lifecycleState,
    total_logs,
    errors,
    warnings,
    error_rate    = round(errors / total_logs * 100, 2),
    health        = if(errors / total_logs > 0.05, "CRITICAL",
                    if(errors / total_logs > 0.01, "DEGRADED",
                    "HEALTHY")),
    // campo numérico para ordenação por severidade
    sort_order    = if(errors / total_logs > 0.05, 1,
                    if(errors / total_logs > 0.01, 2, 3))

| sort sort_order, direction: "ascending"
| sort error_rate, direction: "descending"
| fields -sort_order    // remove campo auxiliar de ordenação
| limit 20
```

---

## Exercício Bônus — BizEvents

```dql
// Parte 1 — Tipos de BizEvent e contagens
fetch bizevents, from: now() - 24h
| summarize count = count(), by: {event.type}
| sort count, direction: "descending"
```

```dql
// Parte 2 — Tendência dos top 3 eventos mais frequentes
fetch bizevents, from: now() - 24h
| filter event.type in ["checkout.completed", "payment.processed", "order.created"]
| makeTimeseries event_count = count(),
    interval: 1h,
    by: {event.type}
```

```dql
// Parte 3 — Enriquecido com nome do serviço
fetch bizevents, from: now() - 24h
| filter isNotNull(dt.entity.service)
| summarize count = count(), by: {event.type, service_id = dt.entity.service}
| lookup [
    fetch dt.entity.service
    | fields entity.id, entity.name
  ],
  sourceField: service_id,
  lookupField: entity.id
| fields entity.name, event.type, count
| sort count, direction: "descending"
| limit 20
```
