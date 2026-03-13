# Soluções — Lab 02: Agregação, Seleção e Parsing

> Use apenas após tentar resolver os exercícios por conta própria.

---

## Exercício 1 — Minha Primeira Agregação

```dql
fetch logs, from: now() - 1h
| summarize count = count(), by: {loglevel}
| sort count, direction: "descending"
```

---

## Exercício 2 — Múltiplas Métricas em Um `summarize`

```dql
fetch logs, from: now() - 1h
| filter isNotNull(duration)
| summarize
    total     = count(),
    avg_dur   = avg(duration),
    max_dur   = max(duration),
    p95_dur   = percentile(duration, 95),
    by: {dt.entity.service}
| sort p95_dur, direction: "descending"
```

> **Nota:** Se o campo `duration` não existir no seu ambiente, substitua pelo campo de duração encontrado com `fieldsSummary` (pode ser `response_time`, `latency`, etc.)

---

## Exercício 3 — Error Rate por Serviço

```dql
fetch logs, from: now() - 2h
| filter isNotNull(dt.entity.service)
| summarize
    total   = count(),
    errors  = countIf(loglevel == "ERROR"),
    by: {dt.entity.service}
| fields
    dt.entity.service,
    total,
    errors,
    error_rate = round(errors / total * 100, 2)
| filter error_rate > 0
| sort error_rate, direction: "descending"
```

---

## Exercício 4 — Seleção e Cálculo de Campos

**Parte A:**
```dql
fetch logs, from: now() - 1h
| filter loglevel == "ERROR"
| fields
    timestamp,
    loglevel,
    content,
    dt.entity.service
| sort timestamp, direction: "descending"
| limit 50
```

**Parte B:**
```dql
fetch logs, from: now() - 2h
| filter isNotNull(dt.entity.service)
| summarize
    total   = count(),
    errors  = countIf(loglevel == "ERROR"),
    by: {dt.entity.service}
| fields
    dt.entity.service,
    total,
    errors,
    error_rate    = round(errors / total * 100, 2),
    health_status = if(errors / total > 0.05, "CRITICAL",
                    if(errors / total > 0.01, "DEGRADED",
                    "HEALTHY"))
| sort error_rate, direction: "descending"
```

---

## Exercício 5 — `fieldsSummary`

Não há solução única — o resultado depende do seu ambiente. O que importa é **observar e anotar** os campos disponíveis para usar nos próximos exercícios.

Exemplo de execução:
```dql
fetch logs, from: now() - 1h
| filter loglevel == "ERROR"
| fieldsSummary
```

Campos que normalmente existem no ambiente Dynatrace:
- Serviço: `dt.entity.service`, `service.name`
- Exception: `exception.type`, `exception.message`, `exception.stacktrace`
- Host: `dt.entity.host`, `host.name`
- Duração: `duration`, `response_time` (depende do SDK)

---

## Exercício 6 — Série Temporal de Erros

**Parte A:**
```dql
fetch logs, from: now() - 2h
| filter loglevel == "ERROR"
| makeTimeseries error_count = count(), interval: 1m
```

**Parte B:**
```dql
fetch logs, from: now() - 2h
| filter loglevel == "ERROR"
| makeTimeseries error_count = count(),
    interval: 1m,
    by: {dt.entity.service}
```

**Parte C:**
```dql
fetch logs, from: now() - 2h
| filter loglevel in ["ERROR", "WARN"]
| makeTimeseries {
    errors   = countIf(loglevel == "ERROR"),
    warnings = countIf(loglevel == "WARN")
  },
  interval: 5m
```

---

## Exercício 7 — Parse de Log Estruturado

**Exemplo genérico para log com `key=value`:**
```dql
fetch logs, from: now() - 1h
| filter contains(content, "level=") and contains(content, "duration=")
| parse content, "LD 'level=' WORD:level ' service=' WORD:svc ' status=' INT:status_code ' duration=' LONG:dur_ms 'ms'"
| filter isNotNull(status_code)
| fields level, svc, status_code, dur_ms
| sort dur_ms, direction: "descending"
| limit 20
```

**Exemplo para log de access log Apache/Nginx:**
```dql
fetch logs, from: now() - 1h
| filter matchesPhrase(content, "HTTP/")
| parse content, "IPADDR:client_ip ' - - [' LD:req_time '] \"' WORD:method ' ' LD:path ' ' LD '\" ' INT:status ' ' LONG:bytes"
| filter isNotNull(status)
| summarize count = count(), by: {status, method}
| sort count, direction: "descending"
```

> O padrão de parse varia muito conforme o formato dos logs. O fundamental é: 1) text literal entre aspas simples, 2) tipo do dado (INT, LONG, WORD, IPADDR, LD, DATA), 3) nome do campo após `:`.

---

## Exercício 8 — Analisando Arrays com `expand`

```dql
// Contagem por tag
fetch dt.entity.service
| expand tags
| filter isNotNull(tags)
| summarize service_count = count(), by: {tags}
| sort service_count, direction: "descending"
| limit 20
```

**Bônus:**
```dql
// Filtrar com arrayContains
fetch dt.entity.service
| filter arrayContains(tags, "production")
| fields entity.id, entity.name, tags
| limit 20
```

---

## Exercício 9 — Pipeline Completo DQL 200

```dql
// Pipeline completo — adapte o padrão de parse ao seu ambiente
fetch logs, from: now() - 2h
| filter contains(content, "method=") and contains(content, "status=")
| parse content, "LD 'method=' WORD:http_method ' path=' LD:endpoint ' status=' INT:status_code ' duration=' LONG:dur_ms 'ms'"
| filter isNotNull(status_code) and isNotNull(dur_ms)
| summarize
    requests      = count(),
    errors        = countIf(status_code >= 500),
    p95_ms        = percentile(dur_ms, 95),
    by: {http_method, endpoint}
| fields
    http_method,
    endpoint,
    requests,
    errors,
    p95_ms,
    slo_violated = p95_ms > 1000
| sort p95_ms, direction: "descending"
| limit 15
```

> **Alternativa caso o parse não se aplique ao seu ambiente:** use spans ao invés de logs para análise de latência de endpoint:
> ```dql
> fetch spans, from: now() - 2h
> | filter isNotNull(duration)
> | summarize
>     requests = count(),
>     p95_ms   = percentile(duration, 95) / 1000000,
>     by: {span.name, dt.entity.service}
> | fields
>     dt.entity.service,
>     endpoint = span.name,
>     requests,
>     p95_ms,
>     slo_violated = p95_ms > 1000
> | sort p95_ms, direction: "descending"
> | limit 15
> ```
