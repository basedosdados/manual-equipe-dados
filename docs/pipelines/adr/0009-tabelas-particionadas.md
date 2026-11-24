---
tipo: adr
titulo: ADR-0009 — Chaves de partição lidas de `partitionDefinition` e fora da comparação de colunas
status: aceito
data: 2026-11-24
pr: https://github.com/basedosdados/pipelines/pull/2228
---
# ADR-0009 — Chaves de partição lidas de `partitionDefinition` e fora da comparação de colunas

## Contexto

O `to_partitions` grava `ano=2024/mes=01/data.csv` e remove as chaves das colunas do arquivo. Verificado em staging reais e em dev:

- `table.schema` **inclui as chaves de partição**, no fim. Sem tratamento, a regra de coluna faltando ([ADR-0005](0005-coluna-faltando.md)) bloquearia todo upload particionado.
- `table._properties["partitionDefinition"]["partitionedColumn"]` traz as chaves, em ordem (por exemplo `[{'field': 'ano'}]`). Em tabela não particionada, o valor é `None`.
- `external_data_configuration.hive_partitioning` é `None` na tabela não particionada e um objeto na particionada.

## Decisão

Obter as chaves da staging por `partitionDefinition`, isolado em `StagingSchema.from_table`. Ele usa o `get_table` que já existe, sem chamada ao storage.

Regras:

1. As chaves ficam **fora** da comparação de colunas, dos dois lados.
2. **Layout (a5):** as chaves do upload (em ordem, lidas do caminho `chave=valor`) precisam ser iguais às da staging. Diferença é **erro** no `append` e **warning** no `overwrite`.
3. Se `partitionDefinition` faltar mas `hive_partitioning` existir, as chaves ficam **desconhecidas** (`None`): o layout não é verificado e a mensagem traz um warning explícito.

## Consequências

### Positivas
- Valores novos de partição (`ano=2025`) não são mudança de schema.
- Não adiciona chamadas ao storage.

### Negativas
- `_properties` é um atributo privado, porque o client do BigQuery não expõe `partitionDefinition`. O formato pode mudar com a versão do client.

### Neutras
- O formato de `partitionDefinition` foi verificado em staging reais e em dev. Os testes automatizados usam mocks do client do BigQuery, então uma mudança de formato numa versão nova do client não aparece como teste falhando.

## Alternativas consideradas

- **Listar um blob no GCS e ler as pastas `chave=valor`** — descartada nesta versão por exigir uma chamada extra ao storage. Fica como fallback possível quando `partitionDefinition` faltar (ver [Limitações e pontos em aberto](../explicacao/checagem-de-schema-no-upload.md#limitacoes-e-pontos-em-aberto)).

## Status

Aceito. Implementado no PR [#2228](https://github.com/basedosdados/pipelines/pull/2228).
