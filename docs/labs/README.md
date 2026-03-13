# Labs Hands-on — DQL para Parceiros Dynatrace

> Este material é para execução **após o workshop**, no seu próprio ritmo.  
> Você precisará de acesso a um tenant Dynatrace com Grail ativado.

---

## Pré-requisitos

- [ ] Acesso a um tenant Dynatrace (produção, trial ou Dynatrace Playground)
- [ ] Grail ativado no tenant (a maioria dos tenants SaaS já tem)
- [ ] Dados sendo ingeridos: logs, métricas e/ou spans (OneAgent ou OpenTelemetry)
- [ ] Permissão para criar **Notebooks** (role: `logs-viewer`, `metrics-reader`)

### Como verificar se o Grail está ativo

1. Acesse seu tenant Dynatrace
2. Navegue até **Notebooks** no menu lateral
3. Crie um novo Notebook e execute:
   ```dql
   fetch logs, from: now() - 15m
   | limit 5
   ```
4. Se retornar registros (mesmo que vazios), o Grail está funcionando

---

## Como Acessar o Notebook

1. Menu lateral → **Notebooks**
2. Clique em **+ New Notebook**
3. No editor, clique em **+ Add section** → **DQL query**
4. Digite sua query e pressione `Ctrl+Enter` (ou `Cmd+Enter` no macOS)

### Atalhos Úteis

| Atalho | Ação |
|--------|------|
| `Ctrl/Cmd + Enter` | Executar query |
| `Ctrl/Cmd + Space` | Autocompletar (campos, funções, tabelas) |
| `Ctrl/Cmd + /` | Comentar/descomentar linha |
| `Shift + Alt + F` | Formatar query automaticamente |
| `Ctrl/Cmd + Z` | Desfazer |

---

## Estrutura dos Labs

| Lab | Foco | Comandos Praticados | Duração Estimada |
|-----|------|---------------------|-----------------|
| [Lab 01](lab-01-fundamentos.md) | Fundamentos | `fetch`, `filter`, `filterOut`, `timeseries` | 30–40 min |
| [Lab 02](lab-02-agregacao.md) | Agregação | `summarize`, `makeTimeseries`, `fields`, `parse` | 35–45 min |
| [Lab 03](lab-03-avancado.md) | Avançado | `lookup`, `join`, `entityAttr`, `data`, `append` | 40–50 min |

---

## Formato de Cada Exercício

Cada exercício segue este formato:

```
### Exercício X — Título claro

**Objetivo:** O que você vai praticar neste exercício

**Tarefa:** Descrição em português do que a query deve fazer

> 💡 Dica: pista para quem travar

**Sua solução:**
\`\`\`dql
// Escreva sua query aqui
\`\`\`
```

- Tente resolver **sem olhar a solução** primeiro
- As soluções estão em [solucoes/](../solucoes/)
- Não há uma resposta única — se sua query retornar o resultado correto, está certa

---

## Nota sobre os Dados

Os exercícios usam Record Sources padrão (`logs`, `metrics`, `spans`, `bizevents`).  
O **nome dos campos** pode variar dependendo de como os dados são ingeridos no seu ambiente.

**Se um campo não existir**, ajuste o exercício para um campo equivalente que exista nos seus dados. Use `fieldsSummary` para descobrir campos disponíveis:

```dql
// Descubra quais campos existem nos seus logs
fetch logs, from: now() - 30m
| limit 100
| fieldsSummary
```

---

## Dicas Gerais

1. **Sempre comece pequeno** — use `from: now() - 15m` e `limit 10` para testar
2. **Leia os erros** — o DQL dá mensagens de erro claras que indicam exatamente o problema
3. **Use o autocompletar** — `Ctrl+Space` depois de `|` mostra todos os comandos disponíveis
4. **Teste por partes** — comente linhas com `//` para isolar problemas
5. **Documente** — adicione comentários `//` para explicar sua lógica (ajuda revisitar mais tarde)

---

👉 **Comece pelo** [Lab 01 — Fundamentos](lab-01-fundamentos.md)
