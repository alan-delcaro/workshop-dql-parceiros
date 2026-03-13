# Agenda — DQL para Parceiros Dynatrace

> **Duração total:** 1h45m – 2h  
> **Formato:** Tela compartilhada com demos ao vivo no Dynatrace Notebook  
> **Ambiente:** Tenant demo com logs, métricas, spans e BizEvents

---

## Linha do Tempo

```
00:00 ─── Abertura e Contexto (10 min)
           - O que é o Grail
           - Por que DQL substitui o USQL e o Log Viewer legado
           - Tour rápido no Notebook
           - Conceito de pipeline: cada comando transforma os dados

00:10 ─── DQL 100: Fundamentos (25 min)
           - fetch — a porta de entrada dos dados
           - filter / filterOut
           - Funções de match: contains, matchesValue, matchesPhrase
           - limit, samplingRatio, scanLimitGBytes
           - timeseries — métricas ao longo do tempo
           - Demo: analisar logs de erro em tempo real

00:35 ─── DQL 200: Agregação, Seleção e Parsing (25 min)
           - summarize: count, avg, sum, percentile
           - makeTimeseries: série temporal a partir de logs
           - fields: selecionar, renomear, calcular campos
           - fieldsSummary: descobrir a estrutura dos dados
           - expand: explodir arrays
           - parse: extrair campos de log com GROK e JSON
           - Demo: error rate por serviço, extrair HTTP status de logs

01:00 ─── DQL 300: Enriquecimento e Conexão de Dados (25 min)
           - Modelo de entidades Dynatrace — rápida introdução
           - entityAttr: acessar atributos do modelo
           - data e append: combinar fontes de dados
           - lookup: enriquecer N:1 (resolver entity ID → nome)
           - join: cruzar dois datasets (M:N)
           - Demo: pipeline completo — error rate por nome real do serviço

01:25 ─── Casos de Uso Práticos (15 min)
           - Análise de saúde de aplicação
           - Dashboards de negócio com BizEvents
           - Correlação infraestrutura ↔ serviço
           - Conformidade e auditoria
           - Base para Alertas e SLOs com DQL

01:40 ─── Como Continuar Aprendendo (5 min)
           - Apresentação dos labs hands-on
           - Recursos oficiais Dynatrace
           - Q&A aberto

01:45 ─── Encerramento
```

---

## Objetivos de Aprendizado

Ao final deste workshop, os participantes serão capazes de:

1. **Entender** a arquitetura Grail e o papel do DQL nela
2. **Escrever queries básicas** com `fetch`, `filter` e `timeseries`
3. **Agregar e transformar dados** com `summarize`, `makeTimeseries`, `fields` e `parse`
4. **Enriquecer e conectar datasets** com `lookup`, `join` e `entityAttr`
5. **Identificar padrões de uso** aplicáveis a cenários reais de clientes

---

## Convenções Usadas no Material

| Convenção | Significado |
|-----------|-------------|
| `código em bloco` | Comando ou query DQL executável |
| **Negrito** | Comando ou função principal do tópico |
| > Nota | Observação importante ou gotcha |
| 💡 | Dica de boas práticas |
| ⚠️ | Cuidado com performance / custo |

---

## Sobre os Labs

Os labs **não** fazem parte da sessão ao vivo. São entregues como material complementar para os parceiros executarem em seu próprio ritmo, em seus ambientes ou no Dynatrace Playground.

Estrutura de cada exercício:

```
### Exercício X — Título

**Objetivo:** O que você vai praticar

**Tarefa:** O que você deve fazer (em português claro)

**Dica:** Pistas para quem travar

**Espaço para sua query:**
\`\`\`dql

\`\`\`
```
