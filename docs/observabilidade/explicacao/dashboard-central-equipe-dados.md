---
tipo: explicacao
titulo: O dashboard Central da Equipe Dados
origem: notas internas (example-docs-observabilidade.md)
---
# O dashboard Central da Equipe Dados

## Contexto

A central de acompanhamento da equipe dados é o dashboard **Central da Equipe Dados** no Metabase. Centraliza um conjunto de indicadores sobre a plataforma e estrutura essas informações em páginas temáticas, evitando que cada pessoa monte sua própria visão a partir de queries soltas no BQ.

## A ideia central

O dashboard se organiza em quatro grupos de informação:

- **Panorama de Dados** — estatísticas gerais do BigQuery de produção (`basedosdados`) e do banco de metadados de produção da API: volumes, contagens de datasets/tabelas, status de publicação, presença de pipelines.
- **Atualização** — proporção de tabelas, pipelines e tabelas semi-automatizadas por status de atualização (atualizada, em observação, desatualizada). A lógica de cálculo está descrita em [Cálculo do status de atualização](calculo-status-atualizacao.md).
- **Prefect** — quantidade de flows com schedule ativa. Construída a partir de `br_bd_metadados.prefect_flows`, extraída diariamente da API do Prefect.
- **Custos** — séries históricas de gasto com a GCP em USD e BRL, agregadas e por serviço.

A página de **Qualidade de Metadados** está prevista (% de preenchimento de metadados de conjuntos e tabelas, validação de compatibilidade com o BigQuery), mas ainda não foi implementada.

### Pontos ausentes conhecidos

- Os metadados do BigQuery hoje cobrem apenas o projeto `basedosdados`. Faltam `basedosdados-dev` e `basedosdados-staging`.
- Não há estatísticas de buckets (`basedosdados-dev`, `basedosdados`).
- Não há custos detalhados a nível de pipeline — apenas a nível de serviço da GCP.

### Disclaimer sobre custos (USD vs. BRL)

Em janeiro de 2025 a conta de faturamento foi modificada para ser cobrada em BRL ao invés de USD. Após a troca, a exportação das tabelas de custo no GCP não foi reativada — como resultado, perdemos as tabelas detalhadas entre fevereiro e setembro de 2025.

Para preservar as séries históricas agregadas, foi feito um compilado a partir do painel de billing da GCP e gravado no BigQuery na tabela `gcp_billing_export_cost_panel_01620F_C10520_CCE6DC`. Isso cobre custos agregados anteriores a 2025 e o período jan–set/2025.

- Pós set/2025: as séries têm USD e BRL exatos (a nova conta inclui a taxa de câmbio que a GCP usou).
- Pré jan/2025: cobranças em USD sem informação de câmbio. As séries foram mantidas em USD; nenhum ajuste foi aplicado.

## Trade-offs e alternativas consideradas

- **Centralizar no Metabase** em vez de Looker/Grafana: alinha com o resto da operação de BI da BD e permite que pessoas fora da equipe técnica consigam abrir uma query.
- **Agrupar tabelas semi-automatizadas separadamente** das pipelines automatizadas: mantém visibilidade sobre dívida operacional (tabelas que dependem de trabalho humano para atualizar).

## Ver também

- [Cálculo do status de atualização](calculo-status-atualizacao.md)
- [Catálogo de métricas](../referencia/metricas/index.md)
