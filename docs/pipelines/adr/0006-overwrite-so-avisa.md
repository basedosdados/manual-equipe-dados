---
tipo: adr
titulo: ADR-0006 — No overwrite, a checagem só avisa e nunca bloqueia
status: aceito
data: 2026-11-24
pr: https://github.com/basedosdados/pipelines/pull/2228
---
# ADR-0006 — No overwrite, a checagem só avisa e nunca bloqueia

Na discussão do PR, esta decisão aparece como "D2".

## Contexto

Com `dump_mode="overwrite"`, a staging e o prefixo do GCS são recriados do zero a partir do arquivo novo. Não há histórico para desalinhar. Mesmo assim, uma mudança de schema pode quebrar o dbt (coluna renomeada ou removida) e, sem checagem, passaria em silêncio.

## Decisão

Ler o schema da staging atual **antes** de `tb.delete` e `st.delete_table`, comparar com o arquivo novo pelas mesmas regras do `append` e transformar **todos** os erros em warning. Nenhuma divergência levanta `SchemaMismatchError`.

A mensagem diz que é um `overwrite`, que a staging foi substituída e pede para revisar os modelos dbt dependentes.

## Consequências

### Positivas
- Um `overwrite` intencional nunca fica travado pela checagem.
- A equipe é avisada de mudanças que podem quebrar o dbt.

### Negativas
- Um `overwrite` com schema errado por engano não é barrado.

### Neutras
- A leitura do schema antigo precisa acontecer antes das deleções. Os testes com mocks verificam essa ordem.

## Alternativas consideradas

- **Não checar** — descartada porque a mudança passaria em silêncio.
- **Aplicar as regras do `append`, com erros** — descartada porque bloquearia um `overwrite` que, por definição, pode mudar o schema.

## Status

Aceito. Implementado no PR [#2228](https://github.com/basedosdados/pipelines/pull/2228).
