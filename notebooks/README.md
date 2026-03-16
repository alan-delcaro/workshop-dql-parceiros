# Notebooks Dynatrace — Workshop DQL

Aqui estão os **4 notebooks Dynatrace** para importar e usar na plataforma.

## 📓 Notebooks Disponíveis

| Notebook | Arquivo | Duração | Foco |
|----------|---------|---------|------|
| **Módulo 00** — Introdução ao Grail e DQL | `00-Introducao-Grail-DQL.json` | ~10 min | Conceitos fundamentais |
| **Módulo 01** — DQL 100: Fundamentos | `01-DQL100-Fundamentos.json` | ~25 min | fetch, filter, timeseries |
| **Módulo 02** — DQL 200: Agregação | `02-DQL200-Agregacao.json` | ~25 min | summarize, fields, parse |
| **Módulo 03** — DQL 300: Avançado | `03-DQL300-Avancado.json` | ~25 min | lookup, join, append |

## 🚀 Como Importar no Dynatrace

### Opção 1: Upload Manual

1. **Abra a Dynatrace** e navegue para **Notebooks**
2. Clique em **+ New Notebook** → **Import**
3. Cole o conteúdo JSON do notebook desejado
4. Clique em **Import**

### Opção 2: Usar a API Dynatrace

```bash
# Substitua <TENANT_URL> e <API_TOKEN> pelos seus valores
TENANT_URL="https://seu-tenant.live.dynatrace.com"
API_TOKEN="seu-token-aqui"

curl -X POST "$TENANT_URL/api/v2/documents" \
  -H "Authorization: Api-Token $API_TOKEN" \
  -H "Content-Type: application/json" \
  --data @00-Introducao-Grail-DQL.json
```

## 💡 Como Usar

Cada notebook contém:

- 📝 **Explicações em Markdown** — conceitos e teoria
- 🔍 **Queries DQL Executáveis** — exemplos prontos para rodar
- 📊 **Desafios Práticos** — exercícios para consolidar aprendizado

**Para executar uma query:**
1. Selecione a célula de DQL
2. Pressione `Ctrl/Cmd + Enter`
3. Veja os resultados no painel lateral

## 🎯 Fluxo Recomendado

1. **Comece com o Módulo 00** — entenda o Grail e o conceito de pipeline
2. **Depois Módulo 01** — execute queries básicas de `fetch` e `filter`
3. **Prossiga para Módulo 02** — agregue dados com `summarize` e `parse`
4. **Finalize com Módulo 03** — conecte múltiplas fontes com `lookup` e `join`

Cada módulo se baseia no anterior, então siga a ordem!

## 🔧 Dependências

- ✅ **Ambiente Dynatrace** com acesso ao Notebook
- ✅ **Dados disponíveis** (logs, métricas, spans, etc.)
- ✅ **Permissões** para criar e editar notebooks

## 📚 Referências

- 📖 [Documentação Oficial DQL](https://docs.dynatrace.com/docs/shortlink/dql-reference)
- 🎓 [Dynatrace University](https://university.dynatrace.com)
- 💬 [Community Forum DQL](https://community.dynatrace.com/t5/DQL/bd-p/dql)

---

**Criado por:** Dynatrace Partner Enablement  
**Data:** Março de 2026  
**Status:** Pronto para uso em produção ✅
