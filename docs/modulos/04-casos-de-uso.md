# Módulo 04 — Casos de Uso Práticos para Parceiros

> **Duração:** ~15 minutos  
> **Objetivo:** Ver DQL aplicado em cenários reais que os parceiros vão encontrar em clientes

---

## Introdução

DQL não é só uma ferramenta de debugging ad-hoc — é a base de **dashboards**, **alertas** e **relatórios** que você vai construir para seus clientes. Os casos abaixo representam os mais solicitados.

---

## Caso 1 — Análise de Saúde de Aplicação

**Cenário:** Cliente quer um painel com health geral dos microserviços — quantos erros, qual o ritmo, quais os top problemas.

### 1.1 — Error rate por serviço (últimas 4h)

```dql
fetch logs, from: now() - 4h
| filter isNotNull(dt.entity.service)
| summarize
    total   = count(),
    errors  = countIf(loglevel == "ERROR"),
    by: {service_id = dt.entity.service}
| lookup [
    fetch dt.entity.service
    | fields entity.id, entity.name
  ],
  sourceField: service_id,
  lookupField: entity.id
| fields
    service  = entity.name,
    total,
    errors,
    rate     = round(errors / total * 100, 2)
| sort rate, direction: "descending"
| limit 10
```

### 1.2 — Tendência de erros — detecção de spike

```dql
// Série temporal de erros de 1 em 1 minuto — boa para detecção de anomalias
fetch logs, from: now() - 2h
| filter loglevel == "ERROR"
| makeTimeseries error_count = count(), interval: 1m
```

### 1.3 — Top mensagens de erro

```dql
fetch logs, from: now() - 2h
| filter loglevel == "ERROR"
| filter isNotNull(exception.message)
| summarize occurrences = count(), by: {exception.message, exception.type}
| sort occurrences, direction: "descending"
| limit 15
```

---

## Caso 2 — Análise de Latência e Performance

**Cenário:** Cliente quer saber quais endpoints são mais lentos, e se a latência está piorando ao longo do tempo.

### 2.1 — P95 e P99 de latência por serviço (spans)

```dql
fetch spans, from: now() - 1h
| filter isNotNull(duration)
| summarize
    p50_ms = percentile(duration, 50) / 1000000,
    p95_ms = percentile(duration, 95) / 1000000,
    p99_ms = percentile(duration, 99) / 1000000,
    count  = count(),
    by: {dt.entity.service, span.name}
| fields
    service_id = dt.entity.service,
    endpoint   = span.name,
    p50_ms, p95_ms, p99_ms, count,
    service = entityAttr(service_id, "entity.name")
| sort p99_ms, direction: "descending"
| limit 20
```

### 2.2 — Degradação de latência — comparar ontem vs hoje

```dql
// Latência de hoje
fetch spans, from: now() - 1h
| filter isNotNull(duration)
| summarize today_p95 = percentile(duration, 95) / 1000000, by: {dt.entity.service}

| join [
    // Latência de ontem (mesmo período de 1h, mas 24h atrás)
    fetch spans, from: now() - 25h, to: now() - 24h
    | filter isNotNull(duration)
    | summarize yesterday_p95 = percentile(duration, 95) / 1000000, by: {dt.entity.service}
  ],
  on: {dt.entity.service},
  kind: left

| fields
    service     = entityAttr(dt.entity.service, "entity.name"),
    today_p95,
    yesterday_p95,
    delta_ms    = round(today_p95 - yesterday_p95, 2),
    delta_pct   = round((today_p95 - yesterday_p95) / yesterday_p95 * 100, 1)
| sort delta_pct, direction: "descending"
| limit 15
```

---

## Caso 3 — Business Intelligence com BizEvents

**Cenário:** Equipe de produto quer métricas de negócio — funil de conversão, receita, abandono de carrinho.

> **BizEvents** são eventos de negócio enviados via OneAgent SDK, API ou Rules Engine. Cada organização define seus próprios campos.

### 3.1 — Funil de conversão simples

```dql
fetch bizevents, from: now() - 24h
| summarize count = count(), by: {event.type}
| sort count, direction: "descending"
```

```dql
// Funil: checkout → payment → order confirmed
data record(step: 1, event: "checkout.initiated"),
     record(step: 2, event: "payment.submitted"),
     record(step: 3, event: "order.confirmed")
| lookup [
    fetch bizevents, from: now() - 24h
    | summarize users = countDistinct(user.id), by: {event_name = event.type}
  ],
  sourceField: event,
  lookupField: event_name
| fields step, event, users
| sort step
```

### 3.2 — Receita por hora (gráfico de área)

```dql
fetch bizevents, from: now() - 7d
| filter event.type == "order.confirmed"
| filter isNotNull(order.total)
| makeTimeseries {
    total_revenue = sum(order.total),
    order_count   = count()
  },
  interval: 1h
```

### 3.3 — Abandono de carrinho por categoria de produto

```dql
fetch bizevents, from: now() - 24h
| filter event.type in ["cart.add", "checkout.initiated"]
| summarize
    cart_adds     = countIf(event.type == "cart.add"),
    checkouts     = countIf(event.type == "checkout.initiated"),
    by: {product.category}
| fields
    category        = product.category,
    cart_adds,
    checkouts,
    abandonment_pct = round((cart_adds - checkouts) / cart_adds * 100, 1)
| sort abandonment_pct, direction: "descending"
```

---

## Caso 4 — Correlação Infraestrutura ↔ Aplicação

**Cenário:** Alguém reporta lentidão. Queremos cruzar métricas de CPU com logs de erro para identificar se é problema de infra.

```dql
// Hosts com CPU > 70% nas últimas 2h
timeseries avg_cpu = avg(dt.host.cpu.usage),
    by: {dt.entity.host},
    interval: 30m,
    from: now() - 2h
| filter avg_cpu > 70
| fields host_id = dt.entity.host, avg_cpu

| join [
    // Erros de aplicação nesses hosts
    fetch logs, from: now() - 2h
    | filter loglevel == "ERROR"
    | summarize error_count = count(), by: {host_id = dt.entity.host}
  ],
  on: {host_id},
  kind: left

| fields
    host        = entityAttr(host_id, "entity.name"),
    avg_cpu,
    error_count,
    // Se há correlação clara
    likely_cause = if(error_count > 100 and avg_cpu > 85, "HIGH_CPU_LIKELY_CAUSE", "INVESTIGATE_FURTHER")
| sort avg_cpu, direction: "descending"
```

---

## Caso 5 — Auditoria e Conformidade

**Cenário:** Cliente precisa de trilha de auditoria — quem fez o quê, quando, em qual ambiente.

```dql
// Eventos de mudança de configuração das últimas 48h
fetch events, from: now() - 48h
| filter event.type in ["CUSTOM_CONFIGURATION", "DEPLOYMENT", "MARKING"]
| fields
    timestamp,
    event_type     = event.type,
    event_name     = event.name,
    changed_by     = actor.email,
    service        = entityAttr(dt.entity.service, "entity.name"),
    environment    = dt.environment,
    details        = event.description
| sort timestamp, direction: "descending"
| limit 100
```

---

## Caso 6 — SLO Tracking com DQL

**Cenário:** Cliente quer medir disponibilidade e latência conforme SLAs acordados.

### 6.1 — Disponibilidade (% de requests com sucesso)

```dql
fetch logs, from: now() - 24h
| filter isNotNull(http.status_code)
| summarize
    total    = count(),
    success  = countIf(http.status_code < 500),
    failures = countIf(http.status_code >= 500)
| fields
    total, success, failures,
    availability_pct = round(success / total * 100, 3),
    slo_target       = 99.9,
    slo_met          = success / total * 100 >= 99.9
```

### 6.2 — Latência vs SLO (% de requests abaixo do threshold)

```dql
fetch spans, from: now() - 1h
| filter span.kind == "SERVER"
| summarize
    total         = count(),
    within_slo    = countIf(duration < 500000000),   // < 500ms em nanosegundos
    by: {dt.entity.service}
| fields
    service    = entityAttr(dt.entity.service, "entity.name"),
    total,
    within_slo,
    slo_pct    = round(within_slo / total * 100, 2)
| sort slo_pct, direction: "ascending"   // pior performance primeiro
| limit 10
```

---

## Caso 7 — Segurança e Detecção de Anomalias

**Cenário:** Detectar padrões suspeitos — logins com falha em sequência, scan de endpoints, IPs anômalos.

```dql
// IPs com muitas tentativas de login falho — possível brute force
fetch logs, from: now() - 1h
| filter matchesPhrase(content, "login failed")
    or matchesPhrase(content, "authentication failed")
    or matchesPhrase(content, "invalid credentials")
| parse content, "LD 'ip=' IPADDR:source_ip LD"
| filter isNotNull(source_ip)
| summarize
    failed_attempts = count(),
    distinct_users  = countDistinct(user.id),
    by: {source_ip}
| filter failed_attempts > 20    // threshold de alerta
| sort failed_attempts, direction: "descending"
```

```dql
// Scanning de endpoints — muitos 404 da mesma origem
fetch logs, from: now() - 30m
| filter http.status_code == 404
| parse content, "LD 'ip=' IPADDR:source_ip LD"
| filter isNotNull(source_ip)
| summarize
    not_found_count  = count(),
    distinct_paths   = countDistinct(http.url),
    by: {source_ip}
| filter not_found_count > 50 and distinct_paths > 10
| sort not_found_count, direction: "descending"
```

---

## Receituário — Melhores Práticas para Produção

| Situação | Recomendação |
|----------|-------------|
| Query exploratória (grande janela) | Use `scanLimitGBytes: 1` e `samplingRatio: 0.1` |
| Dashboard executivo | Fixe `from:` e `to:` para janelas previsíveis |
| Alerta/notificação | Nunca use `samplingRatio` — dados devem ser completos |
| Query de logs estruturados | Use `fieldsSummary` primeiro para entender o schema |
| Resolver entity IDs | Para 1-2 campos: `entityAttr`. Para mais: `lookup` |
| Cruzar dois datasets | `lookup` se N:1, `join` se precisar de M:N |
| Trend over time de logs | `makeTimeseries` ao invés de múltiplos `summarize` |
| Query lenta | Mova `filter` para antes do `summarize` ou `join` |

---

## Próximos Passos

Você aprendeu:
- ✅ Fundamentos: `fetch`, `filter`, `timeseries`
- ✅ Transformação: `summarize`, `makeTimeseries`, `fields`, `parse`
- ✅ Enriquecimento: `lookup`, `join`, `entityAttr`
- ✅ Casos de uso reais

**Agora pratique nos labs:**

| Lab | Conteúdo |
|-----|----------|
| [Lab 01 — Fundamentos](../labs/lab-01-fundamentos.md) | DQL 100 na prática |
| [Lab 02 — Agregação](../labs/lab-02-agregacao.md) | DQL 200 na prática |
| [Lab 03 — Avançado](../labs/lab-03-avancado.md) | DQL 300 na prática |

**Recursos para continuar:**
- 📖 [Documentação DQL](https://docs.dynatrace.com/docs/shortlink/dql-reference)
- 🎓 [DQL 301 — WWSE Enablement](https://dynatrace-wwse.github.io/enablement-dql-301/)
- 💬 [Dynatrace Community — DQL](https://community.dynatrace.com/t5/DQL/bd-p/dql)
- 🏫 [Dynatrace University](https://university.dynatrace.com)
