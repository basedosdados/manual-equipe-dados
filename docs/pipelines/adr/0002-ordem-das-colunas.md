---
tipo: adr
titulo: ADR-0002 — Ordem das colunas diferente é erro em CSV e warning em Parquet
status: aceito
data: 2026-11-24
pr: https://github.com/basedosdados/pipelines/pull/2228
---
# ADR-0002 — Ordem das colunas diferente é erro em CSV e warning em Parquet

## Contexto

A tabela externa de CSV é posicional: uma ordem trocada coloca os dados na coluna errada sem nenhum erro do BigQuery. Em Parquet, o BigQuery casa as colunas por nome, então a ordem não afeta os dados.

## Decisão

- **CSV:** ordem diferente é **erro**. A regra geral é que o header do arquivo seja igual ao schema final esperado (a staging mais as colunas novas, no fim), posição por posição.
- **Parquet:** ordem diferente é **warning** informativo.

## Consequências

### Positivas
- Evita o erro silencioso mais perigoso: dado carregado na coluna errada.
- A mesma regra do CSV também pega coluna nova ou faltando no meio, porque ela desloca as colunas seguintes.

### Negativas
- Uma fonte CSV que reordene colunas passa a ter o upload bloqueado até alguém ajustar o arquivo ou a staging.

### Neutras
- O pipeline não conserta a ordem sozinho.

## Alternativas consideradas

- **Ignorar a ordem** — descartada porque, em CSV, corrompe os dados sem aviso.
- **Warning sempre** — descartada pelo mesmo motivo, para CSV.
- **Erro sempre** — descartada porque, em Parquet, a ordem não tem impacto e bloquearia o upload sem necessidade.
- **Reordenar o arquivo automaticamente** — descartada porque altera o dado enviado ao bucket.

## Status

Aceito. Implementado no PR [#2228](https://github.com/basedosdados/pipelines/pull/2228).
