---
tipo: adr
titulo: ADR-0004 — Coluna nova no fim é acrescentada à staging e avisada
status: aceito
data: 2026-11-24
pr: https://github.com/basedosdados/pipelines/pull/2228
---
# ADR-0004 — Coluna nova no fim é acrescentada à staging e avisada

## Contexto

Em `dump_mode="append"`, a staging só é criada quando ainda não existe, então o schema fica congelado na criação. Quando a fonte ganha uma coluna, o arquivo novo a traz mas a tabela externa não, e o dbt quebra com `Unrecognized name` na primeira materialização seguinte.

O `_sync_staging_schema` já acrescentava as colunas novas ao fim do schema, mas sem avisar ninguém.

## Decisão

Acrescentar a coluna nova ao fim da staging e **avisar** (warning). A checagem roda **antes** do sync.

Em CSV, isso só vale se a coluna estiver no **fim** do arquivo. No meio, o arquivo desalinha da tabela e vira erro de ordem ([ADR-0002](0002-ordem-das-colunas.md)).

## Consequências

### Positivas
- O upload segue sem intervenção humana quando a fonte só ganha colunas.
- A equipe fica sabendo da mudança, em vez de descobrir no dbt.

### Negativas
- A staging é alterada no lugar. O sync é só aditivo: nunca remove nem reordena colunas e não toca no prefixo do GCS.

### Neutras
- Os modelos dbt não passam a usar a coluna nova sozinhos.

## Alternativas consideradas

- **Erro** — descartada porque bloquearia o upload por uma mudança que o pipeline já sabia tratar, acrescentando a coluna ao fim da staging.
- **Acrescentar sem avisar** — descartada porque era o comportamento anterior, que deixava a mudança invisível.

## Status

Aceito. Implementado no PR [#2228](https://github.com/basedosdados/pipelines/pull/2228).
