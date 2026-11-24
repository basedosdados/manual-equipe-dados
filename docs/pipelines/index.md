# Pipelines

Padrões de design, módulos reutilizáveis, validações e tratamentos específicos das pipelines da equipe dados.

## Explicação

- [Pipeline orientado a eventos](explicacao/pipeline-orientado-a-eventos.md) — por que um flow de dataset é dividido em estágios encadeados e como eles se conectam
- [Checagem de schema no upload para o GCS](explicacao/checagem-de-schema-no-upload.md) — por que o `upload_to_gcs` compara o arquivo com a staging e por que as regras são as que são

## Como fazer

- [Estruturar um flow de dataset](como-fazer/estruturar-um-flow-de-dataset.md) — escrever ou migrar `constants.py`, `tasks.py` e `flows.py` para o modelo em estágios
- [Abrir uma issue](como-fazer/abrir-issue.md)
- [Abrir um PR](como-fazer/abrir-pr.md)
- [Atualizar dependências](como-fazer/atualizar-dependencias.md) — Python (`uv`) e dbt packages

## Referência

- [`stage_dispatch.py`](referencia/stage-dispatch.md) — tipos e funções usados para encadear os estágios
- [`schema_check.py`](referencia/schema-check.md) — regras de classificação, módulo, fluxo na task e alertas da checagem de schema

## Runbooks

- [Erro ou aviso de schema no upload](runbooks/erro-de-schema-no-upload.md) — o que fazer quando o `upload_to_gcs` falha com `SchemaMismatchError` ou termina com `SchemaWarning`

## ADRs

- [Índice de ADRs de Pipelines](adr/index.md) — decisões sobre a checagem de schema (0001 a 0009) e deprecações
