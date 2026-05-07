---
tipo: referencia
titulo: Métricas — Atualização
origem: notas internas (example-docs-observabilidade.md)
---
# Métricas — Atualização

Métricas que monitoram a frescor das tabelas em relação à frequência esperada de atualização. A lógica de classificação (Atualizada / Em observação / Desatualizado) está em [Cálculo do status de atualização](../../explicacao/calculo-status-atualizacao.md).

## Proporção de atualização das Tabelas por Status

| Campo | Valor |
|---|---|
| O que é | Proporção de **todas as tabelas** da BD por status de atualização, e listagem das tabelas em observação ou desatualizadas |
| Definição | Ver [Cálculo do status de atualização](../../explicacao/calculo-status-atualizacao.md) |
| Escopo | Todas as tabelas registradas |

## Proporção de atualização das Pipelines por Status

| Campo | Valor |
|---|---|
| O que é | Proporção das **tabelas com pipeline automatizada** por status de atualização |
| Escopo | `pipeline_id IS NOT NULL` |
| Definição | Ver [Cálculo do status de atualização](../../explicacao/calculo-status-atualizacao.md) |

## Proporção de atualização das Tabelas Semi-automatizadas por Status

| Campo | Valor |
|---|---|
| O que é | Proporção das **tabelas semi-automatizadas** por status de atualização |
| Escopo | `pipeline_id IS NULL` (têm código de ELT/ETL no repositório, mas dependem de trabalho humano) |
| Definição | Ver [Cálculo do status de atualização](../../explicacao/calculo-status-atualizacao.md) |

## Ver também

- [Cálculo do status de atualização](../../explicacao/calculo-status-atualizacao.md)
- [O dashboard Central da Equipe Dados](../../explicacao/dashboard-central-equipe-dados.md)
