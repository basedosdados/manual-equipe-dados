# ADRs — Pipelines

Architecture Decision Records sobre decisões que afetam pipelines da Base dos Dados, como as regras da checagem de schema do upload e as **deprecações** (quando descontinuamos uma pipeline, por quê, o que a substitui e quem é impactado).

Cada ADR é numerado sequencialmente dentro deste domínio (`0001-titulo.md`, `0002-titulo.md`, …) e segue o [template de ADR](https://github.com/basedosdados/manual-equipe-dados/blob/main/docs/_templates/adr.md).

> Para deprecações, use o título no formato `Deprecar pipeline <nome>` e registre na seção *Contexto* qual pipeline/tabela substitui (se houver) e o downstream impactado.
>
> **Obrigatório**: toda ADR de deprecação deve linkar o **PR de deprecação no repositório `basedosdados/pipelines`** (ex.: `https://github.com/basedosdados/pipelines/pull/<n>`). Inclua o link no front-matter como `pr:` e referencie-o na seção *Decisão* ou *Status*. Sem PR vinculado, a ADR não deve ser aceita.

## Índice

| Nº  | Título | Status | Data | PR |
| --- | ------ | ------ | ---- | -- |
| [0001](0001-regras-de-schema-em-modulo-novo.md) | Regras de schema do upload em um módulo novo (`schema_check.py`) | aceito | 2026-11-24 | [#2228](https://github.com/basedosdados/pipelines/pull/2228) |
| [0002](0002-ordem-das-colunas.md) | Ordem das colunas diferente é erro em CSV e warning em Parquet | aceito | 2026-11-24 | [#2228](https://github.com/basedosdados/pipelines/pull/2228) |
| [0003](0003-tipo-das-colunas.md) | Checar tipo só em Parquet, com igualdade estrita | aceito | 2026-11-24 | [#2228](https://github.com/basedosdados/pipelines/pull/2228) |
| [0004](0004-coluna-a-mais.md) | Coluna nova no fim é acrescentada à staging e avisada | aceito | 2026-11-24 | [#2228](https://github.com/basedosdados/pipelines/pull/2228) |
| [0005](0005-coluna-faltando.md) | Coluna faltando no arquivo é erro no append | aceito | 2026-11-24 | [#2228](https://github.com/basedosdados/pipelines/pull/2228) |
| [0006](0006-overwrite-so-avisa.md) | No overwrite, a checagem só avisa e nunca bloqueia | aceito | 2026-11-24 | [#2228](https://github.com/basedosdados/pipelines/pull/2228) |
| [0007](0007-erro-de-schema-fora-do-retry.md) | `SchemaMismatchError` próprio e fora do retry | aceito | 2026-11-24 | [#2228](https://github.com/basedosdados/pipelines/pull/2228) |
| [0008](0008-falha-da-checagem-fail-closed.md) | Se a checagem não consegue rodar, o upload é abortado (fail-closed) | aceito | 2026-11-24 | [#2228](https://github.com/basedosdados/pipelines/pull/2228) |
| [0009](0009-tabelas-particionadas.md) | Chaves de partição lidas de `partitionDefinition` e fora da comparação de colunas | aceito | 2026-11-24 | [#2228](https://github.com/basedosdados/pipelines/pull/2228) |

As ADRs 0001 a 0009 registram as decisões da [checagem de schema no upload](../explicacao/checagem-de-schema-no-upload.md). Para consultar as regras resultantes, veja a [referência de `schema_check.py`](../referencia/schema-check.md).
