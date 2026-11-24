---
tipo: adr
titulo: ADR-0005 — Coluna faltando no arquivo é erro no append
status: aceito
data: 2026-11-24
pr: https://github.com/basedosdados/pipelines/pull/2228
---
# ADR-0005 — Coluna faltando no arquivo é erro no append

## Contexto

Com `allow_jagged_rows=False` (o padrão de `Table.create`), um CSV com menos colunas que o schema pode ser recusado pelo BigQuery ("too few columns"). Uma coluna faltando no meio desloca os dados seguintes sem erro. Em Parquet, a coluna faltando seria lida como `NULL`.

## Decisão

Coluna faltando é **erro** em `dump_mode="append"`, em CSV e em Parquet, no fim ou no meio. No `overwrite`, é só warning ([ADR-0006](0006-overwrite-so-avisa.md)).

Uma renomeação aparece como coluna a mais e coluna faltando ao mesmo tempo, então também é erro no `append` e warning no `overwrite`.

## Consequências

### Positivas
- Nenhum arquivo com colunas faltando entra na staging sem alguém ter decidido isso.

### Negativas
- A fonte que deixar de trazer uma coluna precisa de um `overwrite` ou de um ajuste deliberado da staging.

### Neutras
- Habilitar `allow_jagged_rows=True` nas staging de CSV fica como evolução, se o erro de coluna faltando no fim ficar frequente.

## Alternativas consideradas

- **Warning sempre** — não adotada. Em CSV, uma coluna faltando no meio desloca os dados seguintes sem erro.
- **Warning, exceto CSV com coluna faltando no meio, que seria erro de ordem** — não adotada.
- **Warning e habilitar `allow_jagged_rows=True` nas staging de CSV** — não adotada nesta versão. Fica como evolução possível.

## Status

Aceito. Implementado no PR [#2228](https://github.com/basedosdados/pipelines/pull/2228).
