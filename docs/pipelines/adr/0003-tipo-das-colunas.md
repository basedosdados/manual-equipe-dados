---
tipo: adr
titulo: ADR-0003 — Checar tipo só em Parquet, com igualdade estrita
status: aceito
data: 2026-11-24
pr: https://github.com/basedosdados/pipelines/pull/2228
---
# ADR-0003 — Checar tipo só em Parquet, com igualdade estrita

## Contexto

A staging Parquet é sempre all-`STRING`: o BigQuery deriva o schema do `header.parquet`, que o `dump_header` sobe com `.astype(str)`. Por convenção do pipeline, os arquivos de dados também são all-`STRING` (o dbt faz o `safe_cast`). Em CSV, a tabela também é toda `STRING`.

O BigQuery **não converte** tipos na leitura. Um arquivo com coluna `INT64` numa staging `STRING` falha com:

```text
Got non-null value in column n of parquet type INT64. Parquet column requires appropriate ConvertedType to load
```

Esse comportamento foi verificado em dev. 

## Decisão

Em Parquet, ler o tipo de cada coluna do arquivo (pyarrow) e comparar com `get_table().schema`, com **igualdade estrita** depois de normalizar aliases (`INTEGER`=`INT64`, `FLOAT`=`FLOAT64`, `BOOLEAN`=`BOOL`). Sem lista de conversões permitidas. Tipo incompatível é **erro**.

Em CSV não há checagem de tipo.

## Consequências

### Positivas
- Na prática, o erro dispara quando o arquivo viola a convenção all-`STRING`, e aparece antes do upload em vez de no dbt.

### Negativas
- CSV não tem checagem de tipo.
- O primeiro upload (sem staging) não é checado: um Parquet tipado passa e a leitura falha depois.

### Neutras
- Tipo desconhecido (`data_type=None`) fica fora da checagem.

## Alternativas consideradas

- **Comparar o schema inferido pela lib `basedosdados`** — descartada: ele é sempre `STRING`, então não serve para checar tipo.
- **Checar tipo também em CSV** — descartada pelo mesmo motivo.
- **Parquet com lista de conversões permitidas** — descartada: o BigQuery não converte tipos na leitura, então uma lista não teria o que permitir.
- **Deixar tipo fora da v1** — descartada porque o erro só apareceria no dbt.
- **Aceitar qualquer tipo de arquivo quando a staging é `STRING`** — descartada: esconderia um erro de leitura real.

## Status

Aceito. Implementado no PR [#2228](https://github.com/basedosdados/pipelines/pull/2228).
