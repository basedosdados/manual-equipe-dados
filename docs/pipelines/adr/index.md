# ADRs — Pipelines

Architecture Decision Records sobre decisões que afetam as pipelines da Base dos Dados: a arquitetura dos flows, convenções dos modelos dbt e **deprecações** (quando descontinuamos uma pipeline, por quê, o que a substitui e quem é impactado).

Cada ADR é numerado sequencialmente dentro deste domínio (`0001-titulo.md`, `0002-titulo.md`, …) e segue o [template de ADR](https://github.com/basedosdados/manual-equipe-dados/blob/main/docs/_templates/adr.md).

> Para deprecações, use o título no formato `Deprecar pipeline <nome>` e registre na seção *Contexto* qual pipeline/tabela substitui (se houver) e o downstream impactado.
>
> **Obrigatório**: toda ADR de deprecação deve linkar o **PR de deprecação no repositório `basedosdados/pipelines`** (ex.: `https://github.com/basedosdados/pipelines/pull/<n>`). Inclua o link no front-matter como `pr:` e referencie-o na seção *Decisão* ou *Status*. Sem PR vinculado, a ADR não deve ser aceita.

## Índice

| Nº  | Título                            | Status | Data | PR  |
| --- | --------------------------------- | ------ | ---- | --- |
| [0001](0001-dividir-flows-em-estagios.md) | Dividir os flows de dataset em estágios encadeados | aceito | 2026-10-05 | [#1932](https://github.com/basedosdados/pipelines/pull/1932) |
| [0002](0002-nao-apagar-regras-de-acesso-no-pre-hook.md) | Não apagar as regras de acesso no pre_hook dos modelos dbt | aceito | 2026-11-17 | [#2298](https://github.com/basedosdados/pipelines/pull/2298) |
