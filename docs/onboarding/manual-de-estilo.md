---
tipo: tutorial
titulo: Manual de estilo
---

# Manual de estilo

Toda tabela publicada em `basedosdados` segue um conjunto de convenções compartilhadas — o **[Manual de estilo](https://basedosdados.org/docs/style_data)** da BD. Como novo membro da equipe, ler o manual **antes** de tocar em qualquer pipeline ou modelo dbt economiza retrabalho: a maior parte das mudanças pedidas em revisão de PR são, na prática, ajustes para o manual.

## Por que ele existe

Datasets na BD são mantidos por pessoas diferentes, em momentos diferentes, vindos de fontes muito heterogêneas (APIs, FTPs, planilhas, sites antigos). Sem um padrão único, dois datasets independentes não seriam cruzáveis — mesmo quando descrevem o mesmo município no mesmo ano. O manual existe para garantir que **qualquer tabela em `basedosdados` possa ser cruzada com qualquer outra** sem longas etapas de tratamento.

## O que ele cobre, de relance

- **Nomes** de datasets (`<pais>_<orgao>_<nome>`), tabelas e colunas (snake_case, sem acentos).
- **Tipos** de coluna no BigQuery (`STRING` vs `INT64`, `DATE` vs `STRING`, etc.).
- **Padronização de valores** — unidades, formatos de data, códigos de UF/município, chaves de cruzamento.
- **Estrutura de diretórios** dos datasets no repositório.
- **Tabelas auxiliares** (`dicionario`, `diretorio_*`) e quando criá-las.
- **Cobertura temporal e espacial** descritas nos metadados.

A versão completa, com exemplos, está no site público: <https://basedosdados.org/docs/style_data>.

## Onde o manual aparece no seu dia a dia

| Momento | O que o manual decide |
|---|---|
| Nomeando um dataset novo | Prefixo de país, nome do órgão, separadores. |
| Criando uma nova pipeline | nome do conjunto de dados e tabelas |
| Escrevendo o modelo dbt | Nome do arquivo (`<dataset>__<table>.sql`), tipos no `schema.yml`, sufixos. |
| Preenchendo metadados no backend | `coverage`, `column.description`, `directory_column`, `bdpro_filter`. |


## Próximos passos

1. Leia o [Manual de estilo no site da BD](https://basedosdados.org/docs/style_data) de ponta a ponta uma vez. Não precisa decorar — basta saber **onde achar** cada regra quando bater a dúvida.
2. Leia a [explicação interna do papel do manual na infraestrutura](../governanca/metadados/explicacao-manual-de-estilo.md) para entender como ele se conecta às zonas dev/prod, ao dbt e ao backend.
3. Olhe um dataset existente (ex.: `br_bcb_agencia` ou `br_bcb_sicor`) e tente identificar quais regras do manual ele aplica — é a forma mais rápida de internalizar.

## Ver também

- [Manual de estilo (site público)](https://basedosdados.org/docs/style_data) — fonte canônica.
- [Manual de estilo — papel na infraestrutura](../governanca/metadados/explicacao-manual-de-estilo.md) — explicação interna.
- [Fluxo de dados e infraestrutura](fluxo-de-dados-e-infraestrutura.md) — onde o manual se aplica em cada etapa do pipeline.
- [Glossário — Manual de estilo](../glossario.md#m)
