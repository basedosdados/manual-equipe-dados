---
tipo: explicacao
titulo: Cálculo do status de atualização
origem: notas internas (example-docs-observabilidade.md)
---
# Cálculo do status de atualização

## Contexto

As métricas da seção **Atualização** do dashboard Central da Equipe Dados classificam cada tabela em três estados: **Atualizada**, **Em observação** ou **Desatualizado**. Esta página explica a lógica do cálculo e por que existe a categoria intermediária.

## A ideia central

Para cada tabela, o cálculo combina três metadados:

- `latest` — timestamp da última atualização bem-sucedida na BD.
- `frequency` — multiplicador da frequência de atualização (ex.: `1`, `2`).
- slug da entidade temporal (`day`, `week`, `month`, `quarter`, `year`).

Derivam-se duas variáveis:

- `date_diff` — idade real dos dados em dias, em relação à data atual.
- `frequency_days` — produto do multiplicador pela base de dias da unidade (ex.: 1 mês = 30 dias).

### A régua de triagem

O `status_atencao` funciona como uma régua com **zona de escape**, desenhada para acomodar a natureza volátil de fontes de dados públicas:

| Condição | Status |
|---|---|
| `date_diff <= frequency_days` | Atualizada |
| `frequency_days < date_diff <= frequency_days + tolerance_days` | Em observação |
| `date_diff > frequency_days + tolerance_days` | Desatualizado |

O `tolerance_days` depende da periodicidade — por exemplo, 7 dias para tabelas mensais e 90 dias para anuais.

### Por que existe "Em observação"

É comum que tabelas não tenham cronogramas fixos de atualização, o que gera **falsos positivos** para "desatualizado" sempre que a fonte atrasa um pouco. A categoria intermediária sinaliza um atraso existente, porém ainda aceitável, e evita que o painel grite todos os dias por flutuações naturais da fonte original.

### Pipelines vs. tabelas semi-automatizadas

A identificação usa a coluna `pipeline_id` da tabela `"public"."table"` do banco de metadados:

- **Pipelines** — `pipeline_id IS NOT NULL`. Tabelas com automação no Prefect.
- **Tabelas semi-automatizadas** — `pipeline_id IS NULL`. Têm código de ELT/ETL no repositório, mas dependem de trabalho humano para serem atualizadas.

## Trade-offs e alternativas consideradas

- **Tolerância dependente da frequência** em vez de um número fixo: tabelas anuais não devem ser tratadas com a mesma vara das diárias.
- **Manter "Em observação" visível no painel** em vez de só "Atualizada / Desatualizada": permite distinguir atrasos crônicos toleráveis de quebras reais que demandam ação.

## Ver também

- [Catálogo de métricas — Atualização](../referencia/metricas/atualizacao.md)
