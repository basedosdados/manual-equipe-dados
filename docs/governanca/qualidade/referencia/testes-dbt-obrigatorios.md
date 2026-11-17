---
tipo: referencia
titulo: Testes dbt obrigatórios
---
# Testes dbt obrigatórios

## Resumo

| Campo | Valor |
|---|---|
| O que é | O conjunto mínimo de testes dbt que toda tabela publicada pela BD deve ter no `schema.yml` |
| Fonte | Testes genéricos em `tests-dbt/generic/` e no pacote `dbt_utils`, no repositório `basedosdados/pipelines` |
| Atualização | Quando um princípio de tratamento ou um teste genérico novo for aprovado |
| Dono | Equipe dados |

## Definição

Cada teste protege um dos [princípios de tratamento de dados](../../tratamento-de-dados/explicacao-principios-de-tratamento.md).
Uma tabela só é aprovada no PR com todos os testes que se aplicam a ela.

Regras para todos os testes:

- **Só a partição nova.** Todo teste usa `config: where:` com o placeholder da partição mais recente
  (tabela abaixo). Tabela sem partição (snapshot único) roda sem `where`.
- **Severidade `error`.** Um teste que falha bloqueia a publicação. `warn` só com justificativa no
  README do conjunto.
- **Sem repetição.** Uma coluna não passa duas vezes pelo mesmo tipo de checagem. Colunas de chave
  levam `not_null` estrito e entram no `ignore_values` do `not_null_proportion_multiple_columns`.
- **Limiares medidos.** Antes de fixar `at_least`, `proportion_allowed_failures` ou faixas de
  variação, meça o valor real na partição mais recente e registre a medição no README do conjunto.

## Schema / Parâmetros

### Testes obrigatórios

| Teste | Aplica em | Parâmetros principais | Princípio que protege |
|---|---|---|---|
| `dbt_utils.unique_combination_of_columns` | Toda tabela, com a chave única que representa a linha | `combination_of_columns` | Nenhuma linha da fonte se perde (duplicata criada pelo tratamento é bug); [entender antes de testar](../../tratamento-de-dados/referencia-praticas-de-tratamento.md) |
| `not_null` | Cada coluna da chave única | — | Nenhuma linha da fonte se perde |
| `not_null_proportion_multiple_columns` | Toda tabela | `at_least` (padrão `0.05`), `ignore_values` (colunas de chave e colunas vazias por natureza) | Correção de tipo e formato: um `safe_cast` que falha vira nulo sem erro, e uma coluna que esvazia de repente mostra mudança de layout na fonte |
| `relationships` ou `custom_relationships` | Toda coluna padronizada para cruzar bases (`id_municipio`, `sigla_uf`, `cnae`, `cbo`…) contra o diretório | `to`, `field`, `ignore_values`, `proportion_allowed_failures` (padrão `0`) | Padronização para cruzar bases |
| `custom_dictionary_coverage` | Toda tabela com colunas descritas no `dicionario` do conjunto | `dictionary_model`, `columns_covered_by_dictionary` | Padronização para cruzar bases: o dicionário cobre todas as chaves da tabela |
| `custom_row_count_variation` | Toda tabela com série temporal | `history` (a tabela publicada em `basedosdados`), `min_annual_change`, `max_annual_change`, `min_monthly_change`, `max_monthly_change`, `granularity` (`month` ou `year`) | Nenhuma linha da fonte se perde: uma carga com volume muito abaixo do esperado falha |
| `is_row_count_increasing` | Tabelas em que cada carga acumula a anterior (retratos da fonte, como o CNPJ) | `column_name` (a coluna de data da carga) | Nenhuma linha da fonte se perde |

### Placeholders de partição (`config: where:`)

Definidos em `macros/custom_get_where_subquery.sql`:

| Placeholder | Filtra por |
|---|---|
| `__most_recent_year__` | O maior `ano` da tabela |
| `__most_recent_year_month__` | O maior par `ano` e `mes` |
| `__most_recent_date__` | O maior valor da coluna `data` |
| `__most_recent_date_cnpj__` | A maior `data_referencia` (tabelas do `br_rf_cnpj`) |

Tabelas particionadas por outra coluna de data (por exemplo, `data_extracao`) ainda não têm
placeholder próprio: use um `where` explícito ou crie o placeholder no mesmo padrão.

### Fora do dbt

| Checagem | Onde | Princípio que protege |
|---|---|---|
| `check_rows_written`: linhas lidas da fonte contra linhas gravadas, antes do upload | `pipelines/utils/data_tests/rows.py`, chamada no `extract_and_load` | Nenhuma linha da fonte se perde |

### Ainda sem teste

| Princípio | Situação |
|---|---|
| Série sem buracos | Teste genérico `custom_temporal_gaps` proposto na issue [#2322](https://github.com/basedosdados/pipelines/issues/2322) |
| Cobertura temporal igual à da fonte original | Sem teste automático: a comparação com a fonte é feita na revisão de PR |
| Fidelidade à fonte e histórico consistente | Verificados na revisão de PR, com a decisão registrada no README do conjunto |

## Exemplo

```yaml
models:
  - name: br_exemplo__microdados
    tests:
      - dbt_utils.unique_combination_of_columns:
          combination_of_columns: [ano, mes, id_municipio, id_estabelecimento]
          config:
            where: __most_recent_year_month__
      - not_null_proportion_multiple_columns:
          ignore_values: [id_municipio, id_estabelecimento]
          config:
            where: __most_recent_year_month__
      - custom_dictionary_coverage:
          dictionary_model: ref('br_exemplo__dicionario')
          columns_covered_by_dictionary: [tipo_estabelecimento]
          config:
            where: __most_recent_year_month__
      - custom_row_count_variation:
          history: basedosdados.br_exemplo.microdados
          min_monthly_change: -0.20
          max_monthly_change: 0.25
          min_annual_change: -0.10
          max_annual_change: 0.35
          config:
            where: __most_recent_year_month__
    columns:
      - name: id_municipio
        tests:
          - not_null:
              config:
                where: __most_recent_year_month__
          - relationships:
              to: ref('br_bd_diretorios_brasil__municipio')
              field: id_municipio
              config:
                where: __most_recent_year_month__
      - name: id_estabelecimento
        tests:
          - not_null:
              config:
                where: __most_recent_year_month__
```

## Ver também

- [Princípios de tratamento de dados](../../tratamento-de-dados/explicacao-principios-de-tratamento.md)
  — o porquê de cada teste.
- [Práticas de tratamento](../../tratamento-de-dados/referencia-praticas-de-tratamento.md) — as
  regras de engenharia que estes testes aplicam.
- [Regras de negócio do preenchimento de metadados](regras-metadados.md) — as checagens de
  metadados antes da publicação.
