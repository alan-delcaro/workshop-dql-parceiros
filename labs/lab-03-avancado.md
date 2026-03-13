# Lab 03 — Avançado: Enriquecimento e Conexão de Dados (DQL 300)

> **Foco:** `lookup`, `join`, `entityAttr`, `data`, `append`  
> **Duração estimada:** 40–50 minutos  
> **Pré-requisito:** [Lab 02](lab-02-agregacao.md) concluído

---

## Checkpoint Inicial

Verifique se você tem entidades disponíveis no seu tenant:

```dql
fetch dt.entity.service
| fields entity.id, entity.name, entity.type, lifecycleState
| limit 10
```

E verifique se os logs têm o campo de entity ID:

```dql
fetch logs, from: now() - 1h
| filter isNotNull(dt.entity.service)
| limit 5
```

---

## Exercício 1 — Explorando o Modelo de Entidades

**Objetivo:** Familiarizar-se com `fetch dt.entity.*`.

**Parte A:** Liste todos os **hosts** monitorados, com nome, tipo de OS e estado.

```dql
// Parte A

```

**Parte B:** Liste todos os **serviços**, filtrando apenas os que têm `lifecycleState == "ENABLED"`. Quais campos estão disponíveis?

```dql
// Parte B

```

**Parte C:** Use `fieldsSummary` no `dt.entity.service` para descobrir todos os atributos disponíveis:

```dql
fetch dt.entity.service
| limit 50
| fieldsSummary
```

Anote 3 atributos interessantes que você encontrou.

---

## Exercício 2 — `entityAttr` para Resolver Nomes

**Objetivo:** Resolver entity IDs para nomes legíveis usando `entityAttr`.

**Tarefa:** Busque os últimos 30 logs de erro e adicione o **nome legível do serviço** e do **host** usando `entityAttr`.

> 💡 Dica: `entityAttr(dt.entity.service, "entity.name")` resolve o ID para nome.

**Sua solução:**
```dql
// Escreva aqui

```

**Compare:** Execute a mesma query **sem** o `entityAttr`. Qual versão é mais útil num dashboard para o cliente?

---

## Exercício 3 — `lookup` para Enriquecer com Metadados de Serviço

**Objetivo:** Usar `lookup` para adicionar múltiplos atributos de entidade.

**Tarefa:** Construa um **health report** de serviços:

1. Conte erros e total de logs por `dt.entity.service` (última hora)
2. Use `lookup` para adicionar:
   - `entity.name` do serviço
   - `lifecycleState` do serviço
   - `tags` do serviço
3. Calcule o `error_rate` em porcentagem
4. Ordene do maior error rate para o menor

```dql
// Escreva aqui

```

**Reflexão:** Por que usar `lookup` ao invés de `entityAttr` neste caso?

---

## Exercício 4 — `lookup` com `data` (Tabela de Referência Embutida)

**Objetivo:** Usar `data` como tabela de lookup para classificar valores.

**Contexto:** Durante um incident review, você quer classificar serviços por criticidade baseado em uma tabela definida pelo time:

**Tarefa:**

1. Crie uma tabela com `data` mapeando nomes de serviço para criticidade: `"HIGH"`, `"MEDIUM"` ou `"LOW"` (use nomes de serviços reais do seu ambiente)
2. Sete erros por serviço nas últimas 2 horas
3. Use `lookup` para enriquecer com a criticidade
4. Mostre: nome do serviço, erros, criticidade

> 💡 Dica: `data record(service: "meu-servico", criticality: "HIGH"), record(...)`.

```dql
// Escreva aqui — substitua os nomes pelos serviços do seu ambiente

```

---

## Exercício 5 — `append` para Unir Fontes de Dados

**Objetivo:** Combinar dados de fontes distintas com `append`.

**Tarefa:** Crie uma timeline unificada de eventos dos últimos 7 dias combinando:

- **Logs de ERROR** (com `event_type = "LOG_ERROR"`)
- **Events de deployment** (com `event_type = "DEPLOYMENT"`)

Mostre: timestamp, event_type, descrição (content/event.name), serviço. Ordene por timestamp decrescente.

```dql
// Escreva aqui

```

**Pergunta:** Em que situação `append` é mais apropriado que `join`?

---

## Exercício 6 — `join` para Calcular Métricas Comparativas

**Objetivo:** Usar `join` para cruzar dois subsets de dados.

**Tarefa:** Calcule o **error rate real** cruzando dois datasets independentes:

- **Dataset A (esquerda):** contagem de erros por serviço (última hora)
- **Dataset B (direita):** contagem total de logs por serviço (última hora)
- **Resultado:** serviço | erros | total | error_pct

> 💡 Use `kind: left` para manter serviços que têm total mas podem não ter erros.

```dql
// Escreva aqui

```

**Bônus:** Adicione o nome real do serviço com `entityAttr` após o `join`.

---

## Exercício 7 — `join` com `timeseries` (Avançado)

**Objetivo:** Cruzar métricas de infraestrutura com logs de aplicação.

**Tarefa:** Identifique hosts com possível problema de correlação CPU × erros:

1. Use `timeseries` para obter CPU média por host nas últimas 2 horas (intervalo: 30m)
2. Filtre apenas hosts com CPU média > 60% (ajuste o threshold ao seu ambiente)
3. Use `join` para adicionar a contagem de erros de log desses hosts no mesmo período

```dql
// Escreva aqui

```

**Interpretação:** Os hosts com CPU alta têm mais erros? Que conclusão você tiraria apresentando isso para um cliente?

---

## Exercício 8 — Pipeline Mestre (Desafio Final)

**Objetivo:** Construir um pipeline completo de análise de saúde que um parceiro apresentaria para um cliente.

**Tarefa:** Crie um **Service Health Dashboard** completo que, para a **última hora**, mostra:

| Campo | Descrição |
|-------|-----------|
| `service_name` | Nome legível do serviço (via lookup) |
| `lifecycle` | Estado do ciclo de vida |
| `total_logs` | Total de registros |
| `errors` | Contagem de ERRORs |
| `warnings` | Contagem de WARNINGs |
| `error_rate` | `errors / total * 100` (arredondado 2 casas) |
| `health` | `CRITICAL` se rate > 5%, `DEGRADED` se > 1%, `HEALTHY` caso contrário |

**Regras adicionais:**
- Inclua apenas serviços com `isNotNull(entity.name)`
- Ordene: primeiro os `CRITICAL`, depois `DEGRADED`, depois `HEALTHY`
- Limite a 20 serviços

> 💡 Dica: use `sort` com campo calculado, ou crie um campo `sort_order` numérico para ordenar por health.

```dql
// Pipeline mestre — escreva aqui

```

---

## Exercício Bônus — BizEvents (se disponível no seu ambiente)

**Objetivo:** Análise de dados de negócio.

Se você tiver BizEvents configurados, crie uma query que:

1. Liste os tipos de evento disponíveis e suas contagens
2. Calcule a tendência dos eventos mais importantes nas últimas 24h
3. Enriqueça com o nome do serviço que gerou cada evento

```dql
// Parte 1 — explorar tipos de bizevents
fetch bizevents, from: now() - 24h
| summarize count = count(), by: {event.type}
| sort count, direction: "descending"
```

```dql
// Parte 2 e 3 — escreva aqui

```

---

## Checklist de Conclusão

- [ ] Consultar o modelo de entidades com `fetch dt.entity.*`
- [ ] Usar `entityAttr` para resolver IDs para nomes
- [ ] Usar `lookup` para enriquecer N:1 com atributos de entidade
- [ ] Usar `lookup` com `data` para tabela de referência embutida
- [ ] Usar `append` para combinar dois datasets (union)
- [ ] Usar `join` para cruzar dois datasets independentes
- [ ] Construir um pipeline completo com múltiplos estágios

---

## Parabéns! 🎉

Você completou os três labs. Agora você tem as bases para:

- Construir **dashboards** completos de observabilidade com DQL
- Criar **queries de alerta** baseadas em métricas derivadas de logs
- **Enriquecer** qualquer dado com metadados do modelo de entidades
- Apresentar **insights** de forma clara para clientes

**Próximos passos:**
- 📖 [DQL 301 — Advanced Concepts](https://dynatrace-wwse.github.io/enablement-dql-301/)
- 🔀 Explore **Workflow Automations** e **Davis Copilot** com DQL
- 📊 Crie um **Dashboard** usando as queries deste lab

🔍 **Gabarito:** [Soluções do Lab 03](../solucoes/lab-03-solucoes.md)
