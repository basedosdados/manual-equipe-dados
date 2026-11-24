---
tipo: adr
titulo: ADR-0001 — Regras de schema do upload em um módulo novo (`schema_check.py`)
status: aceito
data: 2026-11-24
pr: https://github.com/basedosdados/pipelines/pull/2228
---
# ADR-0001 — Regras de schema do upload em um módulo novo (`schema_check.py`)

## Contexto

O `upload_to_gcs` passou a comparar as colunas do arquivo com o schema da tabela externa de staging (ver a [explicação](../explicacao/checagem-de-schema-no-upload.md)). A comparação precisa de ordem de colunas, tipo, layout de partição e severidades diferentes (warning ou erro).

Já existia o `TableSchemaValidator` (`pipelines/utils/schema_validator.py`). Ele compara só conjuntos de nomes e levanta `ValueError` para qualquer diferença. Seu único uso é `crawler/bcb/utils.py`, com 4 chamadas.

## Decisão

Criar o módulo novo `pipelines/utils/schema_check.py` e deixar o `schema_validator.py` intacto.

## Consequências

### Positivas
- O `bcb` não entra no diff e não corre risco de regressão.
- Cada módulo tem um contrato próprio.

### Negativas
- Passam a existir dois validadores de schema no repositório.

### Neutras
- A unificação dos dois fica fora do escopo e pode ser feita depois.

## Alternativas consideradas

- **Usar o `TableSchemaValidator` como está e capturar o `ValueError`** — descartada porque não distingue coluna a mais de coluna faltando, não vê ordem nem tipo, e a exceção carrega só texto.
- **Estender o `TableSchemaValidator` no mesmo arquivo** — descartada. Foi a primeira tentativa, mas o código novo não reaproveitava nada do antigo e só misturava dois contratos no mesmo arquivo.

## Status

Aceito. Implementado no PR [#2228](https://github.com/basedosdados/pipelines/pull/2228).
