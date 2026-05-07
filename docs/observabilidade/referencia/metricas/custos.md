---
tipo: referencia
titulo: Métricas — Custos
origem: notas internas (example-docs-observabilidade.md)
---
# Métricas — Custos

Para o contexto sobre por que algumas séries estão em USD e outras em BRL, e sobre a perda de dados detalhados entre fev/2025 e set/2025, ver [O dashboard Central da Equipe Dados](../../explicacao/dashboard-central-equipe-dados.md).

## Séries históricas de Custo com a GCP

### Custo Mensal com Todos Serviços da GCP (USD)

| Campo | Valor |
|---|---|
| O que é | Soma total dos custos com toda a infraestrutura GCP consumida pela BD, em USD |
| Fonte | Tabelas de custo exportadas pelo billing da GCP no conjunto `br_bd_indicadores` (projeto `basedosdados`) |

### Custo Mensal com Principais Serviços da GCP (USD)

| Campo | Valor |
|---|---|
| O que é | Soma dos custos com a infraestrutura GCP, agrupados pelos serviços de maior gasto, em USD |
| Definição | Os serviços com maior gasto são identificados; os demais são agrupados em **outros** |
| Fonte | Tabelas de billing no conjunto `br_bd_indicadores` |

## Métricas Mensais e Semestrais (a partir de 2026)

> Disclaimer: estas métricas são calculadas somente a partir de 2026.

### Gasto Mensal com Principais Serviços da GCP (BRL)

| Campo | Valor |
|---|---|
| O que é | Custo mensal com os principais serviços em BRL, no semestre atual |
| Fonte | `gcp_billing_export_resource_v1_01620F_C10520_CCE6DC` (custos da nova conta de faturamento, em BRL e USD) |

### Custo total de serviços da GCP por Semestre

| Campo | Valor |
|---|---|
| O que é | Soma do custo total de serviços em BRL por semestre, a partir de 2026 |
| Fonte | `gcp_billing_export_resource_v1_01620F_C10520_CCE6DC` |

### Custo por Serviço da GCP por Semestre

| Campo | Valor |
|---|---|
| O que é | Custo agregado por serviço GCP por semestre, a partir de 2026 |
| Fonte | `gcp_billing_export_resource_v1_01620F_C10520_CCE6DC` |

## Ver também

- [O dashboard Central da Equipe Dados](../../explicacao/dashboard-central-equipe-dados.md)
