# DQL para Parceiros Dynatrace — Material de Treinamento

> **Formato:** Apresentação guiada pelo instrutor (tela compartilhada) + material de labs para execução posterior  
> **Duração:** 1h45m – 2h  
> **Pré-requisito:** Noções básicas de Dynatrace; acesso a um tenant com dados

---

## Estrutura do Workshop

| # | Módulo | Duração |
|---|--------|---------|
| 0 | [Introdução — O que é Grail e por que DQL?](modulos/00-introducao.md) | 10 min |
| 1 | [DQL 100 — Fundamentos](modulos/01-dql100-fundamentos.md) | 25 min |
| 2 | [DQL 200 — Agregação, Seleção e Parsing](modulos/02-dql200-agregacao.md) | 25 min |
| 3 | [DQL 300 — Enriquecimento e Conexão de Dados](modulos/03-dql300-avancado.md) | 25 min |
| 4 | [Casos de Uso Práticos para Parceiros](modulos/04-casos-de-uso.md) | 15 min |
| 5 | Próximos Passos & Q&A | 5 min |

---

## Material Hands-on (Para Fazer Após o Workshop)

Os labs foram desenhados para execução autônoma no **Dynatrace Notebook**.  
Execute-os na ordem indicada — cada lab constrói sobre o anterior.

| Lab | Conteúdo | Pré-requisito |
|-----|----------|---------------|
| [Lab 01 — Fundamentos](labs/lab-01-fundamentos.md) | `fetch`, `filter`, `timeseries` | Acesso ao tenant |
| [Lab 02 — Agregação](labs/lab-02-agregacao.md) | `summarize`, `fields`, `parse` | Lab 01 concluído |
| [Lab 03 — Avançado](labs/lab-03-avancado.md) | `join`, `lookup`, `entityAttr` | Lab 02 concluído |

Gabarito completo disponível em [solucoes/](solucoes/).

---

## Pré-requisitos para os Labs

- Acesso a um tenant Dynatrace com **Grail** ativado
- Logs, métricas e spans sendo ingeridos (ambiente de produção ou demo)
- Permissão para criar e executar **Notebooks**
- Recomendado: acesso ao [Dynatrace DQL Reference](https://docs.dynatrace.com/docs/shortlink/dql-reference)

---

## Referências

- [Documentação oficial DQL](https://docs.dynatrace.com/docs/shortlink/dql-reference)
- [DQL 301 — Advanced Concepts (WWSE)](https://dynatrace-wwse.github.io/enablement-dql-301/)
- [Dynatrace University](https://university.dynatrace.com)
- [Dynatrace Community — DQL](https://community.dynatrace.com/t5/DQL/bd-p/dql)
