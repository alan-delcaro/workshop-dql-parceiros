# Lab 01 — Fundamentos DQL (DQL 100)

> **Foco:** `fetch`, `filter`, `filterOut`, funções de string, controles de volume, `timeseries`  
> **Duração estimada:** 30–40 minutos  
> **Pré-requisito:** Acesso ao tenant com logs e métricas disponíveis

---

## Aquecimento

Antes de começar, verifique o que você tem disponível no ambiente:

```dql
// Quantos logs existem nos últimos 30 minutos?
fetch logs, from: now() - 30m
| summarize count = count()
```

```dql
// Quais são os campos disponíveis nos logs?
fetch logs, from: now() - 30m
| limit 50
| fieldsSummary
```

Anote os nomes dos campos que você vai usar ao longo dos exercícios.

---

## Exercício 1 — Seu Primeiro Fetch

**Objetivo:** Familiarizar-se com o comando `fetch` e a navegação básica.

**Tarefa:** Buscar os **20 registros mais recentes** de logs. Ordene do mais novo para o mais antigo.

> 💡 Dica: use `sort timestamp, direction: "descending"` e `limit`.

**Sua solução:**
```dql
// Escreva sua query aqui

```

---

## Exercício 2 — Filtrar por Nível de Log

**Objetivo:** Usar `filter` com comparação simples.

**Tarefa:** Buscar apenas logs com nível `ERROR` ou `FATAL` das **últimas 2 horas**. Mostre os 50 registros mais recentes.

> 💡 Dica: use `filter loglevel in [...]` ou combine com `or`.

**Sua solução:**
```dql
// Escreva sua query aqui

```

**Pergunta para reflexão:** Como você faria para buscar apenas logs que **não** são DEBUG ou TRACE?

---

## Exercício 3 — Busca em Texto Livre

**Objetivo:** Usar funções de match em texto.

**Tarefa:** Buscar logs que **contenham** a palavra "exception" (qualquer capitalização) na última hora.

> 💡 Dica: `contains(campo, "valor")` é case-insensitive.

**Sua solução:**
```dql
// Escreva sua query aqui

```

**Bônus:** Modifique a query para buscar `"timeout"` **ou** `"exception"` usando um único `filter`.

---

## Exercício 4 — Combinando Filtros

**Objetivo:** Combinar múltiplas condições de filtro.

**Tarefa:** Buscar logs que satisfaçam **todas** as seguintes condições:
1. Nível de log é `ERROR`
2. Contém a palavra `"database"` ou `"connection"` no conteúdo
3. O campo `dt.entity.service` **não** é nulo

Mostre os 30 resultados mais recentes.

> 💡 Dica: use `and`, `or` e `isNotNull()`.

**Sua solução:**
```dql
// Escreva sua query aqui

```

---

## Exercício 5 — Usando `filterOut`

**Objetivo:** Excluir registros indesejados com `filterOut`.

**Tarefa:** Buscar todos os logs da última hora, mas **exclua**:
- Logs de healthcheck (que contém `/health`, `/ping` ou `/readiness` no conteúdo)
- Logs com nível DEBUG ou TRACE

> 💡 Dica: você pode usar múltiplos `filterOut` em sequência, ou combinar com `or`.

**Sua solução:**
```dql
// Escreva sua query aqui

```

**Reflexão:** Em que situação é melhor usar `filterOut` ao invés de `filter not(...)`?

---

## Exercício 6 — Match com Wildcard

**Objetivo:** Usar `matchesValue` com wildcard `*`.

**Tarefa:** Buscar logs cujo campo `dt.entity.service_name` **começa com** o prefixo do seu serviço principal (ajuste `"payment*"` para um nome real no seu ambiente).

> 💡 Dica: `matchesValue(campo, "prefixo*")` suporta `*` como wildcard.

**Sua solução:**
```dql
// Escreva sua query aqui — substitua "payment*" pelo padrão do seu ambiente

```

---

## Exercício 7 — Controles de Volume

**Objetivo:** Entender `limit`, `samplingRatio` e `scanLimitGBytes`.

**Parte A:** Buscar logs dos **últimos 7 dias**, mas limitar o scan a **0.5 GB**. Mostre a contagem total.

```dql
// Parte A — Escreva aqui

```

**Parte B:** Repetir a query acima usando `samplingRatio: 0.1` (10% dos dados). Como o resultado difere?

```dql
// Parte B — Escreva aqui

```

> ⚠️ Observe que com `samplingRatio`, o resultado é uma estimativa — adequado apenas para exploração, nunca para alertas.

---

## Exercício 8 — Timeseries Básico

**Objetivo:** Criar uma série temporal de métricas.

**Parte A:** Criar um `timeseries` com a **média de CPU** de todos os hosts nas últimas 2 horas, em intervalos de 5 minutos.

> 💡 Dica: use a métrica `dt.host.cpu.usage`.

```dql
// Parte A — Escreva aqui

```

**Parte B:** Modificar a query para mostrar a CPU **por host** (agrupado por `dt.entity.host`).

```dql
// Parte B — Escreva aqui

```

**Parte C (Desafio):** Adicionar o **P95** de CPU além da média.

```dql
// Parte C — Escreva aqui

```

---

## Exercício 9 — Desafio Final do Lab 01

**Objetivo:** Combinar tudo que você aprendeu neste lab.

**Tarefa:** Crie uma query que:
1. Busque logs das **últimas 3 horas**
2. Exclua healthchecks e logs de nível DEBUG/INFO
3. Filtre apenas logs com nível ERROR ou FATAL
4. Mostre apenas registros onde o campo de serviço (`dt.entity.service`) **não** é nulo
5. Ordene do mais recente para o mais antigo
6. Limite a **100 registros**
7. Mostre apenas os campos: `timestamp`, `loglevel`, `content`, `dt.entity.service`

```dql
// Desafio — Escreva sua query aqui

```

---

## Checklist de Conclusão

Antes de ir para o Lab 02, verifique se você consegue:

- [ ] Escrever uma query `fetch` com janela de tempo customizada
- [ ] Usar `filter` com `==`, `!=`, `in`, `and`, `or`
- [ ] Usar `filter` com funções: `contains`, `matchesValue`, `matchesPhrase`, `isNotNull`
- [ ] Usar `filterOut` para excluir registros indesejados
- [ ] Controlar volume com `limit` e `scanLimitGBytes`
- [ ] Criar um `timeseries` básico com agrupamento

✅ **Quando estiver pronto:** [Lab 02 — Agregação](lab-02-agregacao.md)

🔍 **Gabarito:** [Soluções do Lab 01](../solucoes/lab-01-solucoes.md)
