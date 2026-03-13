---
marp: true
theme: default
class: lead
paginate: true
backgroundColor: #0d1117
color: #e6edf3
style: |
  section {
    font-family: 'Roboto', 'Segoe UI', sans-serif;
    background-color: #0d1117;
    color: #e6edf3;
  }
  section.lead {
    background: linear-gradient(135deg, #00b4d8 0%, #0077b6 50%, #023e8a 100%);
    color: #ffffff;
  }
  section.module {
    background: linear-gradient(135deg, #1a1a2e 0%, #16213e 100%);
  }
  h1 { color: #00d4ff; font-size: 2.2rem; }
  h2 { color: #00b4d8; font-size: 1.6rem; border-bottom: 2px solid #00b4d8; padding-bottom: 0.3rem; }
  h3 { color: #48cae4; font-size: 1.2rem; }
  code { background: #161b22; color: #7ee787; border-radius: 4px; padding: 0 4px; }
  pre { background: #161b22 !important; border: 1px solid #30363d; border-radius: 8px; }
  pre code { color: #7ee787; font-size: 0.8rem; }
  table { border-collapse: collapse; width: 100%; }
  th { background: #00b4d8; color: #000; padding: 8px; }
  td { border: 1px solid #30363d; padding: 8px; font-size: 0.85rem; }
  tr:nth-child(even) { background: #161b22; }
  .highlight { color: #00d4ff; font-weight: bold; }
  blockquote { border-left: 4px solid #00b4d8; color: #8b949e; font-style: italic; }
  footer { font-size: 0.7rem; color: #8b949e; }
---

<!-- _class: lead -->

# DQL para Parceiros Dynatrace

## Dynatrace Query Language — Do Básico ao Avançado

**Duração:** 1h45 – 2h &nbsp;|&nbsp; **Formato:** Demo ao vivo + Labs para depois

---

<!-- _class: module -->

## Agenda

| # | Módulo | Tempo |
|---|--------|-------|
| 0 | Introdução — Grail e DQL | 10 min |
| 1 | **DQL 100** — Fundamentos | 25 min |
| 2 | **DQL 200** — Agregação e Parsing | 25 min |
| 3 | **DQL 300** — Enriquecimento e Joins | 25 min |
| 4 | Casos de Uso Práticos | 15 min |
| — | Q&A e próximos passos | 5 min |

> **Labs hands-on** são entregues como material complementar — vocês executam no próprio ritmo

---

<!-- _class: lead -->

# Módulo 0
## O que é Grail e por que DQL?

---

## O Grail — Data Lakehouse nativo

```
   Logs  ──────►  ┌─────────────────────────┐
   Métricas ────► │          GRAIL           │
   Spans  ──────► │  sem pré-agregação       │
   BizEvents ───► │  sem schema fixo         │
   Entidades ───► │  auto-correlacionado     │
                  └──────────┬──────────────┘
                             │  queryable via DQL
                    ┌────────▼───────────┐
                    │  Notebook / Alert  │
                    │  Dashboard / API   │
                    └────────────────────┘
```

**Sem index tuning. Sem partitions. Sem pré-computação.**

---

## Por que DQL unifica tudo?

| Antes | Com DQL |
|-------|---------|
| Log Viewer (busca limitada) | `fetch logs \| filter ...` |
| Metrics Explorer (só clique) | `timeseries avg(cpu)` |
| USQL (só sessões) | `fetch bizevents \| summarize ...` |
| Cruzar logs + traces = impossível | `fetch spans \| join [fetch logs]` |

**Uma linguagem. Todas as fontes. Um resultado.**

---

## Fontes de dados (Record Sources)

| Record Source | O que contém |
|---------------|-------------|
| `logs` | Registros de log |
| `metrics` | Métricas OOTB e custom |
| `spans` | Traces distribuídos |
| `bizevents` | Eventos de negócio |
| `events` | Deploys, mudanças, alertas |
| `dt.entity.*` | Modelo de entidades Dynatrace |

---

## Conceito de Pipeline

```dql
fetch logs                            // 1. busca dados
| filter loglevel == "ERROR"          // 2. filtra
| summarize count = count(),          // 3. agrega
    by: {dt.entity.service}
| sort count, direction: "descending" // 4. ordena
| limit 10                            // 5. limita
```

> Pense como um cano: dados entram brutos, cada `|` os transforma, o resultado sai refinado.

**Demos ao vivo no Notebook →**

---

<!-- _class: lead -->

# Módulo 1
## DQL 100 — Fundamentos

`fetch` · `filter` · `filterOut` · `timeseries`

---

## `fetch` — A Porta de Entrada

```dql
// Syntaxe básica
fetch logs

// Com janela de tempo
fetch logs, from: now() - 2h

// Com limite de scan (boa prática!)
fetch logs, from: now() - 7d, scanLimitGBytes: 1
```

**Unidades de tempo:** `s` `m` `h` `d` `w`

> 💡 Sempre defina `from:` em queries de produção para resultados previsíveis

---

## `filter` — Mantém o que passa

```dql
fetch logs, from: now() - 1h
| filter loglevel == "ERROR"           // igual
| filter loglevel in ["ERROR","FATAL"] // lista
| filter duration > 1000               // comparação
| filter isNotNull(dt.entity.service)  // não nulo
```

**Operadores lógicos:**
```dql
| filter loglevel == "ERROR" and dt.entity.service != null
| filter loglevel == "ERROR" or loglevel == "WARN"
| filter not(loglevel in ["DEBUG","TRACE"])
```

---

## `filterOut` — Remove o que combina

```dql
fetch logs, from: now() - 1h
| filterOut contains(content, "/health")
    or contains(content, "/ping")
| filterOut loglevel in ["DEBUG","TRACE"]
```

> Use `filterOut` quando é mais natural descrever o que você **não quer**

---

## Funções de Match em Strings

| Função | Wildcards | Uso |
|--------|-----------|-----|
| `contains(campo, "val")` | Não | Substring simples |
| `matchesValue(campo, "val*")` | `*` e `?` | Match com padrão |
| `matchesPhrase(campo, "frase")` | Não | Frase em texto livre |
| `startsWith` / `endsWith` | Não | Prefixo / sufixo |

```dql
| filter matchesValue(dt.entity.service_name, "payment*")
| filter matchesPhrase(content, "connection refused")
```

---

## Controles de Volume

```dql
// Limita a saída (não o scan!)
| limit 100

// Amostra aleatória — só para exploração
| samplingRatio 0.1         // analisa 10%

// Limita o scan real — usar sempre em janelas grandes
fetch logs, from: now() - 7d, scanLimitGBytes: 1
```

> ⚠️ `samplingRatio` retorna **estimativas** — nunca use em alertas

---

## `timeseries` — Métricas no Tempo

```dql
// Média de CPU dos hosts de produção, a cada 5 min
timeseries {
    avg_cpu = avg(dt.host.cpu.usage),
    p95_cpu = percentile(dt.host.cpu.usage, 95)
},
    filter: {tags contains "production"},
    interval: 5m,
    from: now() - 2h,
    by: {dt.entity.host}
```

Funções: `avg` · `sum` · `min` · `max` · `percentile` · `count`

---

## Demo 1 — DQL 100 ao vivo

```dql
// Top 50 erros da última hora — excluindo healthchecks
fetch logs, from: now() - 1h
| filterOut contains(content, "/health") or contains(content, "/ping")
| filter loglevel in ["ERROR", "FATAL"]
| filter isNotNull(dt.entity.service)
| sort timestamp, direction: "descending"
| limit 50
```

**→ Vamos executar agora no Notebook**

---

<!-- _class: lead -->

# Módulo 2
## DQL 200 — Agregação, Seleção e Parsing

`summarize` · `makeTimeseries` · `fields` · `parse`

---

## `summarize` — Agregar Dados

```dql
fetch logs, from: now() - 1h
| summarize
    total    = count(),
    errors   = countIf(loglevel == "ERROR"),
    avg_dur  = avg(duration),
    p95_dur  = percentile(duration, 95),
    by: {dt.entity.service}
| sort errors, direction: "descending"
```

**Funções:** `count` · `countIf` · `countDistinct` · `avg` · `sum` · `min` · `max` · `percentile` · `collectDistinct`

---

## Error Rate por serviço

```dql
fetch logs, from: now() - 1h
| filter isNotNull(dt.entity.service)
| summarize
    total  = count(),
    errors = countIf(loglevel == "ERROR"),
    by: {dt.entity.service}
| fields dt.entity.service, total, errors,
         rate = round(errors / total * 100, 2)
| sort rate, direction: "descending"
```

---

## `makeTimeseries` — Tendência de Qualquer Dado

```dql
// Erros por minuto — pronto para gráfico de linha
fetch logs, from: now() - 2h
| filter loglevel == "ERROR"
| makeTimeseries {
    errors   = countIf(loglevel == "ERROR"),
    warnings = countIf(loglevel == "WARN")
  },
  interval: 5m,
  by: {dt.entity.service}
```

> **`timeseries`** — só métricas  
> **`makeTimeseries`** — qualquer fonte (logs, spans, bizevents)

---

## `fields` — Selecionar e Calcular

```dql
fetch logs, from: now() - 1h
| fields
    timestamp,
    service_id = dt.entity.service,
    duration_s = duration / 1000.0,       // cálculo
    is_slow    = duration > 5000,         // booleano
    health     = if(duration > 5000, "SLOW",
                 if(duration > 1000, "OK",
                 "FAST"))                 // condicional
```

**Remover campos:** `| fields -trace_id, -span_id`

**Explorar campos:** `| fieldsSummary`

---

## `parse` — Extrair Campos de Logs

Log original:
```
level=ERROR service=payment status=500 duration=2345ms
```

Query:
```dql
fetch logs, from: now() - 1h
| parse content,
    "'level=' WORD:level ' service=' WORD:svc
     ' status=' INT:status ' duration=' LONG:dur_ms 'ms'"
| filter isNotNull(status)
| fields level, svc, status, dur_ms
| sort dur_ms, direction: "descending"
```

---

## Padrões GROK mais usados

| Padrão | Captura | Exemplo |
|--------|---------|---------|
| `WORD` | Sem espaço | `ERROR`, `payment-api` |
| `INT` | Inteiro | `200`, `500` |
| `LONG` | Inteiro longo | timestamps |
| `FLOAT` | Decimal | `3.14` |
| `IPADDR` | IP | `192.168.1.1` |
| `LD` | Até próximo padrão | qualquer texto |
| `DATA` | Qualquer coisa | texto com espaços |
| `TIMESTAMP` | Data/hora ISO | `2024-03-12T10:45` |

---

## Demo 2 — DQL 200 ao vivo

```dql
// Análise de performance de endpoints
fetch logs, from: now() - 1h
| filter contains(content, "method=")
| parse content,
    "'method=' WORD:method ' path=' LD:path ' status=' INT:status ' duration=' LONG:ms 'ms'"
| filter isNotNull(status)
| summarize
    requests = count(),
    errors   = countIf(status >= 500),
    p95_ms   = percentile(ms, 95),
    by: {method, path}
| sort p95_ms, direction: "descending"
| limit 15
```

---

<!-- _class: lead -->

# Módulo 3
## DQL 300 — Enriquecimento e Conexão de Dados

`entityAttr` · `lookup` · `join` · `data` · `append`

---

## O Problema dos Entity IDs

Logs contêm IDs opacos:
```
dt.entity.service: "SERVICE-ABCD1234"  ← inútil em dashboard
```

Queremos:
```
service_name: "payment-service-prod"   ← legível pelo cliente
```

**3 formas de resolver:**

| Abordagem | Quando usar |
|-----------|-------------|
| `entityAttr(id, "attr")` | 1–2 atributos, inline |
| `lookup [fetch dt.entity.*]` | múltiplos atributos |
| `join` | datasets independentes, M:N |

---

## `entityAttr` — O Mais Simples

```dql
fetch logs, from: now() - 1h
| filter loglevel == "ERROR"
| fields
    timestamp,
    content,
    service = entityAttr(dt.entity.service, "entity.name"),
    host    = entityAttr(dt.entity.host,    "entity.name")
| sort timestamp, direction: "descending"
| limit 30
```

**Rápido. Sem subquery. Para 1–2 campos.**

---

## `lookup` — Enriquecimento N:1

```dql
fetch logs, from: now() - 1h
| filter loglevel == "ERROR"
| summarize errors = count(), by: {service_id = dt.entity.service}

| lookup [
    fetch dt.entity.service
    | fields entity.id, entity.name, lifecycleState, tags
  ],
  sourceField: service_id,
  lookupField: entity.id

| fields entity.name, errors, lifecycleState
| sort errors, direction: "descending"
```

---

## `data` + `lookup` — Tabela de Referência

```dql
fetch logs, from: now() - 2h
| parse content, "LD 'status=' INT:code LD"
| lookup [
    data record(code: 200, label: "Success"),
         record(code: 400, label: "Bad Request"),
         record(code: 401, label: "Unauthorized"),
         record(code: 500, label: "Server Error"),
         record(code: 503, label: "Unavailable")
  ],
  sourceField: code,
  lookupField: code
| summarize count = count(), by: {label}
```

---

## `append` — União de Datasets

```dql
// Logs de ERROR
fetch logs, from: now() - 7d
| filter loglevel == "ERROR"
| fields timestamp, type = "LOG_ERROR", desc = content

| append [
    fetch events, from: now() - 7d
    | filter event.type == "DEPLOYMENT"
    | fields timestamp, type = "DEPLOY", desc = event.name
  ]

| sort timestamp, direction: "descending"
| limit 100
```

> `append` = **UNION ALL** &nbsp;·&nbsp; `join` = **JOIN**

---

## `join` — Cruzar Dois Datasets

```dql
// Erros por serviço
fetch logs, from: now() - 1h
| filter loglevel == "ERROR"
| summarize errors = count(), by: {dt.entity.service}

| join [
    fetch logs, from: now() - 1h
    | summarize total = count(), by: {dt.entity.service}
  ],
  on: {dt.entity.service},
  kind: left

| fields
    service    = entityAttr(dt.entity.service, "entity.name"),
    errors, total,
    error_pct  = round(errors / total * 100, 2)
| sort error_pct, direction: "descending"
```

---

## Demo 3 — Pipeline Mestre DQL 300

```dql
// Service Health Dashboard — última 1 hora
fetch logs, from: now() - 1h
| filter isNotNull(dt.entity.service)
| summarize total = count(), errors = countIf(loglevel == "ERROR"),
    by: {service_id = dt.entity.service}
| lookup [fetch dt.entity.service | fields entity.id, entity.name, lifecycleState],
    sourceField: service_id, lookupField: entity.id
| filter isNotNull(entity.name)
| fields service_name = entity.name, lifecycleState, total, errors,
    error_rate = round(errors / total * 100, 2),
    health = if(errors/total > 0.05, "CRITICAL",
             if(errors/total > 0.01, "DEGRADED", "HEALTHY"))
| sort error_rate, direction: "descending"
```

---

<!-- _class: lead -->

# Módulo 4
## Casos de Uso Práticos para Parceiros

---

## Caso 1 — Saúde de Aplicação

```dql
// Top 10 serviços com mais erros — últimas 4h
fetch logs, from: now() - 4h
| filter loglevel in ["ERROR","FATAL"]
| summarize errors = count(), dist_exceptions = countDistinct(exception.type),
    by: {service_id = dt.entity.service}
| lookup [fetch dt.entity.service | fields entity.id, entity.name],
    sourceField: service_id, lookupField: entity.id
| fields entity.name, errors, dist_exceptions
| sort errors, direction: "descending"
| limit 10
```

---

## Caso 2 — Latência (SLO Tracking)

```dql
// Endpoints violando SLO de 500ms
fetch spans, from: now() - 1h
| filter isNotNull(duration)
| summarize
    requests   = count(),
    within_slo = countIf(duration < 500000000),  // < 500ms (ns)
    p99_ms     = percentile(duration, 99) / 1000000,
    by: {dt.entity.service, span.name}
| fields
    service    = entityAttr(dt.entity.service, "entity.name"),
    endpoint   = span.name,
    requests,
    slo_pct    = round(within_slo / requests * 100, 2),
    p99_ms,
    slo_ok     = within_slo / requests >= 0.999
| filter not(slo_ok)
| sort slo_pct
```

---

## Caso 3 — BizEvents (Negócio)

```dql
// Tendência de receita por hora — últimos 7 dias
fetch bizevents, from: now() - 7d
| filter event.type == "order.confirmed"
| filter isNotNull(order.total)
| makeTimeseries {
    revenue     = sum(order.total),
    order_count = count()
  },
  interval: 1h
```

```dql
// Abandono de carrinho por categoria
fetch bizevents, from: now() - 24h
| filter event.type in ["cart.add","checkout.initiated"]
| summarize adds = countIf(event.type == "cart.add"),
    checkouts = countIf(event.type == "checkout.initiated"),
    by: {product.category}
| fields category = product.category,
    abandonment = round((adds - checkouts) / adds * 100, 1)
| sort abandonment, direction: "descending"
```

---

## Caso 4 — Correlação CPU × Erros

```dql
// Hosts com CPU alta E muitos erros ao mesmo tempo
timeseries avg_cpu = avg(dt.host.cpu.usage),
    by: {dt.entity.host}, interval: 30m, from: now() - 2h
| filter avg_cpu > 70
| fields host_id = dt.entity.host, avg_cpu

| join [
    fetch logs, from: now() - 2h
    | filter loglevel == "ERROR"
    | summarize errors = count(), by: {host_id = dt.entity.host}
  ],
  on: {host_id}, kind: left

| fields host = entityAttr(host_id, "entity.name"),
    avg_cpu, errors,
    likely_cause = if(errors > 100 and avg_cpu > 85, "CPU_CORRELATED", "INVESTIGATE")
| sort avg_cpu, direction: "descending"
```

---

## Caso 5 — Segurança (Brute Force)

```dql
// IPs com muitas tentativas de login falho
fetch logs, from: now() - 1h
| filter matchesPhrase(content, "login failed")
    or matchesPhrase(content, "authentication failed")
| parse content, "LD 'ip=' IPADDR:src_ip LD"
| filter isNotNull(src_ip)
| summarize
    attempts       = count(),
    distinct_users = countDistinct(user.id),
    by: {src_ip}
| filter attempts > 20
| sort attempts, direction: "descending"
```

---

## Receituário de Boas Práticas

| Situação | Recomendação |
|----------|-------------|
| Query exploratória (janela grande) | `scanLimitGBytes: 1` + `samplingRatio: 0.1` |
| Dashboard executivo | Fixe `from:` e `to:` explícitos |
| Alerta / notificação | Nunca use `samplingRatio` |
| Query lenta | Mova `filter` para antes do `summarize` |
| Resolver 1–2 atributos | Use `entityAttr` |
| Resolver múltiplos atributos | Use `lookup` |
| Combinar linhas de fontes diferentes | `append` (UNION) |
| Cruzar duas métricas/contagens | `join` (JOIN) |

---

<!-- _class: lead -->

# Próximos Passos

---

## Material Entregue

| Material | O que é |
|----------|---------|
| 📄 **Módulos** | Referência completa de cada comando com exemplos |
| 🧪 **Lab 01** | Exercícios de fundamentos (fetch, filter, timeseries) |
| 🧪 **Lab 02** | Exercícios de agregação e parsing |
| 🧪 **Lab 03** | Exercícios avançados (lookup, join, entityAttr) |
| ✅ **Gabaritos** | Soluções explicadas para os 3 labs |

Execute os labs no **Dynatrace Notebook** do seu tenant.

---

## Para Continuar Aprendendo

- 📖 [Documentação Oficial DQL](https://docs.dynatrace.com/docs/shortlink/dql-reference)
- 🚀 [DQL 301 — Advanced Concepts (WWSE)](https://dynatrace-wwse.github.io/enablement-dql-301/)
- 🎓 [Dynatrace University](https://university.dynatrace.com)
- 💬 [Community — DQL](https://community.dynatrace.com/t5/DQL/bd-p/dql)

**Próximos temas sugeridos:**
- Workflow Automations com DQL
- Davis Copilot e DQL
- Criação de Alertas e SLOs baseados em DQL
- DQL em APIs (Grail V2)

---

<!-- _class: lead -->

# Obrigado! 🙏

## Dúvidas?

**Workshop material:** [github.com/alan-delcaro/workshop-dql-parceiros](https://github.com/alan-delcaro/workshop-dql-parceiros)

---

<!-- Notas do apresentador — não aparecem nos slides exportados -->

<!--
SLIDE 1 — Título
- Introduza-se brevemente
- Mencione o formato: demos ao vivo + labs para depois
- Pergunte: "Alguém já usou DQL antes?"

SLIDE 4 — Pipeline
- Demonstre o conceito de pipe no terminal/terminal do Notebook
- Mostre que cada comando pode ser executado com Ctrl+Enter para ver o resultado intermediário

SLIDE 7 — filter
- Execute ao vivo: fetch logs, from: now() - 1h | filter loglevel == "ERROR" | limit 10
- Mostre o autocompletar com Ctrl+Space

SLIDE 13 — timeseries
- Visualize como gráfico de linha no Notebook (mude para modo de gráfico)

SLIDE 15 — Demo 1
- Momente: execute linha a linha, comentando cada pipe

SLIDE 22 — Demo 2
- Se não tiver logs no formato key=value, use spans: fetch spans | fields span.name, duration | limit 20

SLIDE 30 — Demo 3
- Este é o mais impressionante — pipeline completo. Reserve 5 minutos para ele.

SLIDE 36 — Caso 5 Segurança
- Bom momento para mencionar que DQL é base de Workflows e Alertas automáticos

-->
