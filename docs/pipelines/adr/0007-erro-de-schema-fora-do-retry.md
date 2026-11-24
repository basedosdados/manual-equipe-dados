---
tipo: adr
titulo: ADR-0007 — `SchemaMismatchError` próprio e fora do retry
status: aceito
data: 2026-11-24
pr: https://github.com/basedosdados/pipelines/pull/2228
---
# ADR-0007 — `SchemaMismatchError` próprio e fora do retry

## Contexto

A task `upload_to_gcs` tem `retries=3`. Repetir o upload não corrige uma divergência de schema, então as tentativas extras só atrasam o alerta.

## Decisão

Levantar uma exceção de tipo próprio, `SchemaMismatchError`, e excluí-la do retry com `retry_condition_fn` (`_retry_unless_schema_mismatch`). Qualquer outra falha continua sendo retentada.

## Consequências

### Positivas
- O erro aborta antes de alterar a staging ou o bucket, e o alerta chega na primeira tentativa.

### Negativas
- Mais um tipo de exceção para quem trata falhas do upload conhecer.

### Neutras
- A task `upload_to_gcs` continua com `retries=3` para as demais falhas.

## Alternativas consideradas

- **Exceção comum, com 3 tentativas inúteis** — descartada porque atrasa o alerta sem chance de sucesso.

## Status

Aceito. Implementado no PR [#2228](https://github.com/basedosdados/pipelines/pull/2228).
