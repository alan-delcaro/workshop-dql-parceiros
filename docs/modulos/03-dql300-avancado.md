# Módulo 03 — DQL 300: Enriquecimento e Conexão de Dados

> **Duração:** ~25 minutos  
> **Objetivo:** Conectar datasets, enriquecer com metadados de entidades e montar pipelines completos

---

## 1. O Modelo de Entidades Dynatrace

Antes de falar em `lookup` e `join` com entidades, vale entender como o Dynatrace organiza seu modelo topológico — o **Smartscape**.

### O que são entidades?

Entidades representam componentes reais monitorados: hosts, serviços, aplicações, processos, k8s pods, etc.

Cada entidade tem:
- Um **ID único** (`dt.entity.host`, `dt.entity.service`, etc.)
- **Atributos** — propriedades como nome, IP, tipo de OS, versão
- **Tags** — labels definidos por usuário ou auto-descobertos
- **Relacionamentos** — "esse serviço chama aquele serviço"

### Tipos de entidade mais comuns

| Tipo | Descrição |
|------|-----------|
| `dt.entity.host` | Hosts físicos / VMs |
| `dt.entity.service` | Serviços detectados pelo OneAgent |
| `dt.entity.application` | Aplicações web (RUM) |
| `dt.entity.process_group` | Grupos de processos |
| `dt.entity.kubernetes_node` | Nodes Kubernetes |
| `dt.entity.kubernetes_pod` | Pods Kubernetes |
| `dt.entity.cloud_application` | App no cloud (Fargate, etc.) |

### Por que isso importa para DQL?

Logs e métricas contêm **entity IDs** (ex: `SERVICE-ABCD1234`), não nomes legíveis.  
Para criar dashboards e relatórios que humanos entendam, precisamos **resolver** esses IDs para nomes reais.

```
LOG record:
  content: "Payment failed"
  dt.entity.service: "SERVICE-ABCD1234"   ← ID opaco

  ↓ depois de lookup

  content: "Payment failed"
  entity.name: "payment-service-prod"     ← nome real
```

---

## 2. O Comando `fetch dt.entity.*`

Você pode consultar o modelo de entidades diretamente:

```dql
// Lista todos os serviços monitorados
fetch dt.entity.service
| fields entity.id, entity.name, entity.type, tags, lifecycleState
| limit 50
```

```dql
// Lista hosts com SO e IP
fetch dt.entity.host
| fields entity.id, entity.name,
         os_type        = entity.osType,
         ip_addresses   = entity.ipAddresses
| limit 30
```

```dql
// Filtra entidades por tag
fetch dt.entity.service
| filter arrayContains(tags, "production")
| fields entity.id, entity.name, tags
```

---

## 3. A Função `entityAttr`

A função `entityAttr` acessa **atributos de uma entidade** diretamente na query, sem precisar de um `lookup` separado.

### Sintaxe

```dql
entityAttr(entity_id_field, "nome.do.atributo")
```

### Exemplos

```dql
// Resolve o nome do serviço a partir do ID presente nos logs
fetch logs, from: now() - 1h
| filter loglevel == "ERROR"
| filter isNotNull(dt.entity.service)
| fields
    timestamp,
    content,
    service_name = entityAttr(dt.entity.service, "entity.name"),
    host_name    = entityAttr(dt.entity.host, "entity.name")
| limit 50
```

```dql
// Busca atributo de tags e lifecycleState do serviço associado ao span
fetch spans, from: now() - 1h
| filter duration > 2000000000    // mais de 2 segundos (em nanosegundos)
| fields
    trace_id,
    span_name,
    duration_ms = duration / 1000000,
    service     = entityAttr(dt.entity.service, "entity.name"),
    state       = entityAttr(dt.entity.service, "entity.lifecycleState")
| sort duration_ms, direction: "descending"
| limit 20
```

> 💡 `entityAttr` é a forma mais simples de resolver IDs para nomes. Use quando você só precisa de 1-2 atributos de uma entidade.

---

## 4. O Comando `data`

O `data` **injeta registros estáticos** no pipeline — útil para testar queries, criar tabelas de referência embutidas ou comparar com valores fixos.

```dql
// Cria uma tabela de ambiente → cor de alerta
data record(environment: "production",  threshold_ms: 500,  color: "red"),
     record(environment: "staging",     threshold_ms: 2000, color: "yellow"),
     record(environment: "development", threshold_ms: 5000, color: "green")
```

```dql
// Tabela de tradução de códigos HTTP
data record(status_code: 200, description: "OK"),
     record(status_code: 201, description: "Created"),
     record(status_code: 400, description: "Bad Request"),
     record(status_code: 401, description: "Unauthorized"),
     record(status_code: 500, description: "Internal Server Error")
```

> 💡 Útil em cenários onde você quer construir uma query de referência como protótipo, sem depender de dados ao vivo.

---

## 5. O Comando `append`

O `append` **concatena** o resultado de outra query ao dataset atual — análogo ao `UNION ALL` do SQL.

```dql
// Combina logs de dois serviços críticos em uma visão única
fetch logs, from: now() - 1h
| filter matchesValue(dt.entity.service_name, "payment-service*")
| append [
    fetch logs, from: now() - 1h
    | filter matchesValue(dt.entity.service_name, "order-service*")
  ]
| filter loglevel == "ERROR"
| sort timestamp, direction: "descending"
```

> ⚠️ Os schemas (campos) dos dois datasets não precisam ser idênticos, mas campos que não existem em um lado aparecerão como `null` no resultado combinado.

```dql
// Combina eventos de deployment com eventos de problema para correlação
fetch events, from: now() - 7d
| filter event.type == "DEPLOYMENT"
| fields timestamp, event.type, event.name, environment
| append [
    fetch events, from: now() - 7d
    | filter event.type == "AVAILABILITY"
    | fields timestamp, event.type, event.name, environment
  ]
| sort timestamp, direction: "descending"
```

---

## 6. O Comando `lookup`

O `lookup` **enriquece** os registros do dataset principal com campos de um **dataset de referência** — relação N:1. Análogo ao `LEFT JOIN` onde você resolve uma chave estrangeira.

### Sintaxe

```dql
| lookup [
    <subquery de referência>
  ],
  sourceField: <campo_chave_no_dataset_principal>,
  lookupField: <campo_chave_no_dataset_de_referencia>,
  prefix: "lookup."        // opcional — prefixo nos campos resolvidos
```

### Exemplo 1 — Resolver nome do serviço

```dql
fetch logs, from: now() - 1h
| filter loglevel == "ERROR"
| filter isNotNull(dt.entity.service)
| summarize error_count = count(), by: {dt.entity.service}
| lookup [
    fetch dt.entity.service
    | fields entity.id, entity.name, lifecycleState
  ],
  sourceField: dt.entity.service,
  lookupField: entity.id
| fields entity.name, error_count, lifecycleState
| sort error_count, direction: "descending"
```

### Exemplo 2 — Enriquecer com tabela `data`

```dql
fetch logs, from: now() - 1h
| parse content, "LD 'statusCode=' INT:status_code LD"
| filter isNotNull(status_code)
| lookup [
    data record(code: 200, label: "Success"),
         record(code: 201, label: "Created"),
         record(code: 400, label: "Bad Request"),
         record(code: 401, label: "Unauthorized"),
         record(code: 403, label: "Forbidden"),
         record(code: 404, label: "Not Found"),
         record(code: 500, label: "Server Error"),
         record(code: 503, label: "Unavailable")
  ],
  sourceField: status_code,
  lookupField: code
| fields timestamp, status_code, label, content
```

### Exemplo 3 — Com `prefix:` para evitar colisão de nomes

```dql
fetch logs, from: now() - 1h
| filter loglevel == "ERROR"
| summarize count = count(), by: {dt.entity.service}
| lookup [
    fetch dt.entity.service
    | fields entity.id, entity.name, entity.type, tags
  ],
  sourceField: dt.entity.service,
  lookupField: entity.id,
  prefix: "svc."
| fields svc.entity.name, count, svc.tags
| sort count, direction: "descending"
```

> 💡 **`lookup` vs `entityAttr`:**
> - `entityAttr`: resolve 1-2 atributos simples, mais conciso
> - `lookup`: quando você precisa de múltiplos campos, condicionar na subquery, ou enriquecer com `data`

---

## 7. O Comando `join`

O `join` realiza uma **junção completa** entre dois datasets — relação M:N. Mais poderoso que o `lookup`, mas mais custoso.

### Tipos de join

| Tipo | Comportamento | Equivalente SQL |
|------|--------------|-----------------|
| `inner` (padrão) | Apenas registros com match nos dois lados | `INNER JOIN` |
| `left` | Mantém todos do lado esquerdo, null se sem match | `LEFT JOIN` |
| `right` | Mantém todos do lado direito | `RIGHT JOIN` |

### Sintaxe

```dql
<dataset principal>
| join [
    <dataset secundário>
  ],
  on: {campo_comum},
  kind: left        // inner | left | right
```

### Exemplo 1 — Calcular error rate cruzando duas queries

```dql
// Esquerda: contagem de erros por serviço
fetch logs, from: now() - 1h
| filter loglevel == "ERROR"
| summarize errors = count(), by: {dt.entity.service}
| join [
    // Direita: contagem total por serviço
    fetch logs, from: now() - 1h
    | summarize total = count(), by: {dt.entity.service}
  ],
  on: {dt.entity.service},
  kind: left
| fields
    dt.entity.service,
    errors,
    total,
    error_pct = round(errors / total * 100, 2)
| sort error_pct, direction: "descending"
```

### Exemplo 2 — Correlacionar deploys com spikes de erro

```dql
// Serviços com deploy nas últimas 4h
fetch events, from: now() - 4h
| filter event.type == "DEPLOYMENT"
| fields service_id = dt.entity.service, deploy_time = timestamp, deploy_version = event.name

| join [
    // Erros antes e depois do deploy
    fetch logs, from: now() - 4h
    | filter loglevel == "ERROR"
    | summarize post_errors = count(), last_error = max(timestamp), by: {service_id = dt.entity.service}
  ],
  on: {service_id},
  kind: left

| fields service_id, deploy_time, deploy_version, post_errors, last_error
| sort post_errors, direction: "descending"
```

### Exemplo 3 — Cruzar métricas e logs para diagnóstico

```dql
// Hosts com CPU alta nas últimas 2h
timeseries avg_cpu = avg(dt.host.cpu.usage),
    interval: 30m,
    from: now() - 2h,
    by: {dt.entity.host}
| filter avg_cpu > 80
| fields host_id = dt.entity.host, avg_cpu

| join [
    // Logs de erro desses mesmos hosts nas últimas 2h
    fetch logs, from: now() - 2h
    | filter loglevel == "ERROR"
    | summarize error_count = count(), by: {host_id = dt.entity.host}
  ],
  on: {host_id},
  kind: left

| fields host_id,
         host_name  = entityAttr(host_id, "entity.name"),
         avg_cpu,
         error_count
| sort avg_cpu, direction: "descending"
```

---

## 8. Pipeline Completo — Juntando Tudo

Este é o padrão "DQL 300": um pipeline que busca dados brutos, filtra, agrega, enriquece com entidades e apresenta um resultado acionável.

```dql
// Pipeline completo: health report de serviços
// Última 1 hora | Error rate | Nome real | Status do ciclo de vida

fetch logs, from: now() - 1h
| filter isNotNull(dt.entity.service)
| summarize
    total    = count(),
    errors   = countIf(loglevel == "ERROR"),
    warnings = countIf(loglevel == "WARN"),
    by: {service_id = dt.entity.service}

// Enriquecer com nome e status da entidade
| lookup [
    fetch dt.entity.service
    | fields entity.id, entity.name, lifecycleState, tags
  ],
  sourceField: service_id,
  lookupField: entity.id

// Calcular métricas derivadas
| fields
    service_name  = entity.name,
    lifecycle     = lifecycleState,
    total,
    errors,
    warnings,
    error_rate    = round(errors / total * 100, 2),
    warn_rate     = round(warnings / total * 100, 2),
    // Classificar saúde
    health = if(errors / total > 0.05, "CRITICAL",
             if(errors / total > 0.01, "DEGRADED",
             "HEALTHY"))

| filter isNotNull(service_name)
| sort error_rate, direction: "descending"
| limit 20
```

---

## Resumo DQL 300

| Comando / Função | O que faz |
|-----------------|-----------|
| `fetch dt.entity.*` | Consulta o modelo de entidades |
| `entityAttr(id, "attr")` | Acessa atributo de entidade inline |
| `data record(f: v, ...)` | Injeta dados estáticos no pipeline |
| `append [subquery]` | Concatena dois datasets (UNION ALL) |
| `lookup [subquery], sourceField:, lookupField:` | Enriquecimento N:1 |
| `join [subquery], on: {campo}, kind:` | Junção M:N entre dois datasets |
| `if(cond, "val_true", "val_false")` | Condicional inline |
| `round(valor, casas)` | Arredonda número |

---

## Próximo Módulo

➡️ [Módulo 04 — Casos de Uso Práticos para Parceiros](04-casos-de-uso.md)
