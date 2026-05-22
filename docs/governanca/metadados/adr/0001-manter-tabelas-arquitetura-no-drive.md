---
tipo: adr
titulo: ADR-0001 — Manter tabelas de arquitetura no Google Drive
status: aceito
data: 2026-05-22
origem: basedosdados/pipelines#1115 — "[RFC]: Arquiteturas de tabelas dentro do Git Workflow"
---
# ADR-0001 — Manter tabelas de arquitetura no Google Drive

## Contexto

A **tabela de arquitetura** é o artefato em que descrevemos a estrutura de uma tabela antes de subi-la para o BigQuery: nome e descrição das colunas, tipo no BigQuery, cobertura temporal, unidade de medida, presença em diretórios, marcação de dados sensíveis, partições, etc. Hoje esse artefato vive como **planilha no Google Drive**, compartilhada com a equipe de dados para revisão.

Em [basedosdados/pipelines#1115](https://github.com/basedosdados/pipelines/issues/1115) foi proposto mover a tabela de arquitetura para um **arquivo YAML versionado no Git**, com revisão via pull request — replicando o fluxo do antigo repositório `mais` (atual `sdk`).

### Argumentos a favor da migração para YAML/Git

- Histórico versionado e revisão dentro do mesmo fluxo de PR do código.
- YAML tem tipos nativos (boolean, int, float, lista, dict) e pode ser validado por CI.
- A planilha hoje pode ser editada ou apagada por acidente; já houve casos de tabelas em produção sem arquitetura salva no Drive.
- Não há um fluxo claro para contribuidor externo adicionar a arquitetura no Drive da BD.
- Pipeline para gerar o `schema.yml` do dbt e popular o backend (`/admin/`) ficaria mais direto.
- Arquivos texto habilitam revisão por bot e por IA.

### Argumentos contra a migração

- **Edição visual** de tabelas com muitas colunas é significativamente mais difícil em YAML do que em planilha — proximidade visual entre variáveis do mesmo tema, padronização de nomes, verificar quais colunas têm/não têm unidade de medida, copiar valores entre várias linhas de uma vez.
- O **formato YAML é distante do resultado final** que o usuário enxerga no site da BD (que é tabular). Editar em formato muito diferente do output aumenta a chance de erros só percebidos depois do deploy.
- Contribuidores externos têm mais dificuldade com YAML do que com planilha.

## Decisão

**Manter** as tabelas de arquitetura como planilhas no Google Drive. **Não** migrar para arquivos YAML versionados no Git.

A decisão se baseia no peso da **ergonomia de edição e revisão visual** para tabelas com muitas colunas — que é o caso comum na BD — e na **proximidade do formato planilha com o formato tabular** em que os metadados aparecem para o usuário final no site.

## Consequências

### Positivas

- Mantemos a ergonomia de planilha para edição em massa (copiar valores entre linhas, multi-seleção visual, ordenação por tema).
- O artefato de revisão continua próximo, visualmente, do que o usuário vê no site.
- Barreira de entrada baixa para contribuidores externos: planilha é universal.
- Nenhum trabalho de migração ou retooling é necessário no curto prazo.

### Negativas

- Continuamos sem histórico versionado das arquiteturas — alterações acidentais ou exclusões na planilha não têm trilha de auditoria.
- Não há fluxo claro para contribuidor externo adicionar a arquitetura no Drive da BD (precisamos suprir isso com processo, não com ferramenta).
- Revisão da arquitetura não acontece no mesmo lugar do PR de código — exige troca de contexto.
- Não conseguimos rodar validações de CI (tipos, nomes, manual de estilo) sobre as arquiteturas.
- Não conseguimos usar bots ou IA de code review sobre o artefato de arquitetura.

### Neutras

- A decisão pode ser revisitada se: (i) o volume de tabelas novas crescer a ponto de justificar automação pesada, ou (ii) surgir uma ferramenta que combine edição tabular com versionamento em Git.
- Mitigações de governança sobre a planilha (template padronizado, owners, link obrigatório no PR de subida) continuam sendo recomendáveis independentemente desta decisão.

## Alternativas consideradas

- **Migrar a tabela de arquitetura para YAML versionado no Git** — descartada. Os ganhos de versionamento e automação não compensam a perda de ergonomia para tabelas com muitas colunas e o distanciamento do formato tabular final.
- **Fluxo híbrido (planilha como interface, YAML gerado automaticamente)** — não avaliada em profundidade; pode ser objeto de RFC futuro caso a decisão atual seja revisitada.

## Status

Aceito. Issue [basedosdados/pipelines#1115](https://github.com/basedosdados/pipelines/issues/1115) fechada com base nesta ADR.
