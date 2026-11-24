---
tipo: adr
titulo: ADR-0008 — Se a checagem não consegue rodar, o upload é abortado (fail-closed)
status: aceito
data: 2026-11-24
pr: https://github.com/basedosdados/pipelines/pull/2228
---
# ADR-0008 — Se a checagem não consegue rodar, o upload é abortado (fail-closed)

## Contexto

A checagem depende do BigQuery (para ler o schema da staging) e da leitura do arquivo local. Os dois podem falhar, por exemplo com um erro de API do BigQuery.

## Decisão

**Fail-closed**: se a checagem não consegue rodar, o upload é abortado. A falha usa um erro **diferente** de `SchemaMismatchError`, então mantém os retries normais, já que pode ser transitória ([ADR-0007](0007-erro-de-schema-fora-do-retry.md)).

## Consequências

### Positivas
- Nada sobe para o bucket sem validação.

### Negativas
- Uma instabilidade do BigQuery passa a derrubar flows que antes subiriam normalmente.

### Neutras
- Como a falha é retentada, uma instabilidade curta não chega a derrubar o flow.

## Alternativas consideradas

- **Fail-open, com warning no log** — descartada porque permitiria subir dado sem validação justamente quando a checagem está indisponível.

## Status

Aceito. Implementado no PR [#2228](https://github.com/basedosdados/pipelines/pull/2228).
