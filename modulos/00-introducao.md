# Módulo 00 — Introdução ao Grail e ao DQL

> **Duração:** ~10 minutos  
> **Objetivo:** Entender onde o DQL se encaixa na plataforma Dynatrace e por que ele importa

---

## O que é o Grail?

**Grail** é o data lakehouse nativo da Dynatrace. Diferente dos bancos de dados tradicionais de monitoramento, o Grail foi projetado para:

- **Ingerir e reter** volumes massivos de dados de observabilidade sem pré-agregação
- **Indexar e buscar** logs, traces, métricas e eventos sem necessidade de schema fixo
- **Correlacionar** automaticamente dados via o **Smartscape** (modelo de entidades topológicas)
- **Escalar** elasticamente sem tuning de índices ou partições manuais

```
                  ┌─────────────────────────────────────┐
                  │              GRAIL                   │
   Logs  ──────►  │  ┌─────────┐  ┌─────────┐           │
   Métricas ────► │  │  logs   │  │ metrics │  ...       │
   Spans  ──────► │  └─────────┘  └─────────┘           │
   BizEvents ───► │         ▲ queryable via DQL          │
   Entidades ───► │         │                            │
                  └─────────┼────────────────────────────┘
                            │
                         DQL query
                            │
                     ┌──────▼──────┐
                     │  Notebook   │
                     │  Dashboard  │
                     │  Alert      │
                     └─────────────┘
```

---

## Por que DQL?

Antes do Grail, a Dynatrace tinha múltiplas interfaces de query:
- **USQL** — para métricas de sessão e conversões
- **Log Viewer** — busca simples baseada em texto
- **Metrics Explorer** — visualização de métricas (sem query language)

O DQL **unifica** tudo isso em uma única linguagem expressiva:

| O que você quer fazer | Antes | Com DQL |
|-----------------------|-------|---------|
| Analisar logs | Log Viewer (limitado) | `fetch logs | filter ...` |
| Agregar métricas | Metrics Explorer | `timeseries avg(...)` |
| Cruzar dados de trace com logs | Muito difícil | `fetch spans | join [fetch logs]` |
| Enriquecer com metadados de entidade | Impossível ad-hoc | `lookup` + `entityAttr` |
| Dashboards baseados em dados brutos | USQL restrito | Query DQL em tile |

---

## Tabelas e Fontes de Dados

O DQL trabalha com **record sources** — análogos a tabelas em SQL. Os principais:

| Record Source | O que contém | Exemplo de uso |
|---------------|-------------|----------------|
| `logs` | Registros de log ingeridos via OneAgent, OpenTelemetry, API | Análise de erros, debug |
| `metrics` | Métricas ingeridas (host CPU, custom metrics) | Tendências, capacity planning |
| `events` | Eventos de deployment, availability, config change | Correlação com incidentes |
| `spans` | Spans de distributed tracing | Latência, dependency mapping |
| `bizevents` | Eventos de negócio customizados | KPIs de negócio |
| `dt.entity.*` | Modelo de entidades (hosts, serviços, apps...) | Lookup de metadados |

---

## Conceito de Pipeline

O DQL usa um modelo de **pipeline** — cada comando recebe os dados do anterior e os transforma:

```dql
fetch logs                          // 1. busca os dados brutos
| filter loglevel == "ERROR"        // 2. filtra apenas erros
| summarize count = count(),        // 3. agrega por serviço
    by: {dt.entity.service}
| sort count, direction: "descending" // 4. ordena do maior pro menor
| limit 10                          // 5. retorna apenas os top 10
```

> Pense como um cano: a água (dados) entra de um lado, cada seção do cano a transforma, e o resultado sai do outro.

---

## O Dynatrace Notebook

O **Notebook** é o ambiente de query interativo do DQL.

**Como acessar:**
1. No menu lateral da Dynatrace, busque **Notebooks**
2. Clique em **+ New Notebook**
3. Adicione uma seção de **DQL query**

**Atalhos úteis:**
| Atalho | Ação |
|--------|------|
| `Ctrl/Cmd + Enter` | Executar query |
| `Ctrl/Cmd + Space` | Autocompletar |
| `Ctrl/Cmd + /` | Comentar linha |
| `Shift + Alt + F` | Formatar query |

**Janela de tempo:**
- O Notebook tem um seletor de tempo global (canto superior direito)
- Queries com `from:` explícito ignoram o seletor global
- Para exploração, use o seletor; para queries de produção, especifique `from:` no código

---

## Estrutura Básica de uma Query DQL

```dql
fetch <record_source>,              // sempre começa com fetch ou timeseries
    from: now() - 1h,              // janela de tempo (opcional, padrão: 2h)
    scanLimitGBytes: 1             // limite de scan (boas práticas)
| comando2 ...
| comando3 ...
```

**Sintaxe geral:**
- Comandos separados por `|` (pipe)
- Case-sensitive para nomes de campos
- Strings entre aspas duplas: `"valor"`
- Comentários com `//`
- Campos especiais começam com `dt.` (namespaced fields)

---

## Próximo Módulo

➡️ [DQL 100 — Fundamentos](01-dql100-fundamentos.md)
