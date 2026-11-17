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

Cada teste protege um dos [princípios de tratamento de dados](../../tratamento-de-dados/explicacao/principios-de-tratamento.md).
Uma tabela só é aprovada no PR com todos os testes obrigatórios que se aplicam a ela.

Regras para todos os testes:

- **Partição nova em tabelas grandes.** Em tabelas com alguma partição maior que 1 GB ou tamanho
  total maior que 5 GB, todo teste usa `config: where:`
  com um dos placeholders de partição (tabela abaixo). Tabelas menores podem rodar os testes na
  tabela toda.
- **Severidade `error`.** Um teste que falha bloqueia a publicação. `warn` só com justificativa no
  README do conjunto.
- **Sem repetição.** Uma coluna não passa duas vezes pelo mesmo tipo de checagem (por exemplo,
  `not_null` e `not_null_proportion_multiple_columns` na mesma coluna).
- **Limiares medidos e justificados.** Antes de fixar `at_least` ou `proportion_allowed_failures`,
  meça o valor real. Registre no README do conjunto a medição e o motivo pelo qual é aceitável
  tolerar aquela proporção de falhas no teste.

## Schema / Parâmetros

### Testes obrigatórios

| Teste | Aplica em | Parâmetros principais | Princípio que protege |
|---|---|---|---|
| `dbt_utils.unique_combination_of_columns` | Toda tabela, com a chave única que representa a linha ([exceções](#excecoes-a-chave-unica)) | `combination_of_columns` | Nenhuma linha da fonte se perde (duplicata criada pelo tratamento é bug); [entender antes de testar](../../tratamento-de-dados/referencia/praticas-de-tratamento.md) |
| `not_null_proportion_multiple_columns` | Toda tabela | `at_least: 0.99` (o padrão do teste ainda é `0.05`, então passe o valor sempre; mudança do padrão na issue [#2323](https://github.com/basedosdados/pipelines/issues/2323)); `ignore_values`: colunas de chave e colunas muito vazias por natureza | Correção de tipo e formato: um `safe_cast` que falha vira nulo sem erro, e uma coluna que esvazia de repente mostra mudança de layout na fonte |
| `relationships` ou `custom_relationships` | Toda coluna padronizada para cruzar bases (`id_municipio`, `sigla_uf`, `cnae`, `cbo`…) contra o diretório | `to`, `field`, `ignore_values`, `proportion_allowed_failures` (padrão `0`) | Padronização para cruzar bases |
| `custom_dictionary_coverage` | Toda tabela com colunas descritas no `dicionario` do conjunto | `dictionary_model`, `columns_covered_by_dictionary` | Padronização para cruzar bases: o dicionário cobre todas as chaves da tabela |

### Exceções à chave única

- **Tabela sem chave única** (por exemplo, `br_me_rais.microdados_vinculos`): não leva o teste, e a
  ausência de chave fica documentada no README do conjunto e na descrição da tabela.
- **Duplicatas ou chaves nulas que vêm da fonte** (linhas repetidas, códigos faltantes preenchidos
  com nulo): use `custom_unique_combinations_of_columns`, com `combination_of_columns` e um
  `proportion_allowed_failures` baixo, medido na fonte. Documente a tolerância no README.

> **Atenção:** o `custom_unique_combinations_of_columns` calcula a proporção **por grupo
> repetido** (linhas de um mesmo valor de chave ÷ total de linhas), não pela soma de todas as
> duplicatas. Com o padrão de `0.05`, ele quase nunca falha. Use um valor bem baixo e meça antes.

### Testes recomendados (não obrigatórios)

| Teste | Quando usar | Parâmetros principais | Princípio que protege |
|---|---|---|---|
| `custom_row_count_variation` | Tabelas com série temporal e volume estável entre cargas | `history` (a tabela publicada em `basedosdados`), `min_annual_change`, `max_annual_change`, `min_monthly_change`, `max_monthly_change`, `granularity` (`month` ou `year`) | Nenhuma linha da fonte se perde |
| `is_row_count_increasing` | Tabelas em que cada carga acumula a anterior (retratos da fonte, como o CNPJ) | `column_name` (a coluna de data da carga) | Nenhuma linha da fonte se perde |

### Placeholders de partição (`config: where:`)

São o filtro padrão dos testes, definidos em `macros/custom_get_where_subquery.sql`:

| Placeholder | Filtra por |
|---|---|
| `__most_recent_year__` | O maior `ano` da tabela |
| `__most_recent_year_month__` | O maior par `ano` e `mes` |
| `__most_recent_date__` | O maior valor da coluna `data` |
| `__most_recent_date_cnpj__` | A maior `data_referencia` (tabelas do `br_rf_cnpj`) |

Não use `where` escrito à mão. Se a tabela é particionada por outra coluna (por exemplo,
`data_extracao`), crie um placeholder novo no macro, no mesmo padrão dos existentes.

### Fora do dbt

| Checagem | Onde | Princípio que protege |
|---|---|---|
| `check_rows_written`: linhas lidas da fonte contra linhas gravadas, antes do upload | `pipelines/utils/data_tests/rows.py`, chamada no `extract_and_load` | Nenhuma linha da fonte se perde |
| Cobertura temporal da tabela contra a da fonte | Metadados, no painel do Metabase ([dashboard Central da Equipe Dados](../../../observabilidade/explicacao/dashboard-central-equipe-dados.md)) | Cobertura temporal igual à da fonte original |

### Ainda sem teste

| Princípio | Situação |
|---|---|
| Série sem buracos | Teste genérico `custom_temporal_gaps` proposto na issue [#2322](https://github.com/basedosdados/pipelines/issues/2322) |
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
          at_least: 0.99
          ignore_values: [id_municipio, id_estabelecimento, complemento]
          config:
            where: __most_recent_year_month__
      - custom_dictionary_coverage:
          dictionary_model: ref('br_exemplo__dicionario')
          columns_covered_by_dictionary: [tipo_estabelecimento]
          config:
            where: __most_recent_year_month__
    columns:
      - name: id_municipio
        tests:
          - relationships:
              to: ref('br_bd_diretorios_brasil__municipio')
              field: id_municipio
              config:
                where: __most_recent_year_month__
```

O exemplo é de uma tabela grande (partição maior que 1 GB ou total maior que 5 GB). Numa tabela menor, os blocos `config: where:` podem
sair.

## Ver também

- [Princípios de tratamento de dados](../../tratamento-de-dados/explicacao/principios-de-tratamento.md)
  — o porquê de cada teste.
- [Práticas de tratamento](../../tratamento-de-dados/referencia/praticas-de-tratamento.md) — as
  regras de engenharia que estes testes aplicam.
- [Regras de negócio do preenchimento de metadados](regras-metadados.md) — as checagens de
  metadados antes da publicação.
