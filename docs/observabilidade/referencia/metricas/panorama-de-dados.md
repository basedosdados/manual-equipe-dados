---
tipo: referencia
titulo: Métricas — Panorama de Dados
origem: notas internas (example-docs-observabilidade.md)
---
# Métricas — Panorama de Dados

Estatísticas gerais do BigQuery de produção (`basedosdados`) e do banco de metadados de produção da API.

## Estatísticas Gerais do BigQuery de Produção

Fonte comum: tabela `bigquery_tables`, construída pela pipeline `br_bd_metadados.bigquery_tables`, que compila dados do `INFORMATION_SCHEMA` (default do BQ).

### Volume de Dados Armazenados (TB)

| Campo | Valor |
|---|---|
| O que é | Soma da quantidade total de TB armazenados no BQ `basedosdados` |
| Fonte | `br_bd_metadados.bigquery_tables` |
| Atualização | Diária (pipeline) |

### Quantidade de Datasets no BQ

| Campo | Valor |
|---|---|
| O que é | Contagem de datasets no BQ `basedosdados` |
| Fonte | `br_bd_metadados.bigquery_tables` |
| Atualização | Diária (pipeline) |

### Quantidade de Tabelas no BQ

| Campo | Valor |
|---|---|
| O que é | Contagem de tabelas no BQ `basedosdados` |
| Fonte | `br_bd_metadados.bigquery_tables` |
| Atualização | Diária (pipeline) |

## Estatísticas Gerais do Banco de Metadados (API)

Fonte comum: tabela `tables` do banco de metadados de produção da API.

### Quantidade de Datasets

| Campo | Valor |
|---|---|
| O que é | Contagem de datasets com tabelas registradas no banco de metadados de prod |
| Definição | `COUNT(DISTINCT dataset_id)` em `tables` |

### Quantidade de Tabelas

| Campo | Valor |
|---|---|
| O que é | Contagem de tabelas registradas no banco de metadados de prod |
| Definição | `COUNT(DISTINCT table_id)` em `tables` |

### Quantidade de Tabelas com Pipelines

| Campo | Valor |
|---|---|
| O que é | Contagem de tabelas com pipeline associada |
| Definição | Linhas de `tables` com `pipeline_id IS NOT NULL` |

### Quantidade de Tabelas por Status de Publicação

| Campo | Valor |
|---|---|
| O que é | Distribuição das tabelas por status de publicação (ativa / não-ativa) |
| Para que serve | Visualizar quais tabelas estão de fato disponíveis no produto |

## Ver também

- [O dashboard Central da Equipe Dados](../../explicacao/dashboard-central-equipe-dados.md)
