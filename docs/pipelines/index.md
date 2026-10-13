# Pipelines

Padrões de design, módulos reutilizáveis, validações e tratamentos específicos das pipelines da equipe dados.

## Explicação

- [Pipeline orientado a eventos](explicacao/pipeline-orientado-a-eventos.md) — por que um flow de dataset é dividido em estágios encadeados e como eles se conectam

## Como fazer

- [Estruturar um flow de dataset](como-fazer/estruturar-um-flow-de-dataset.md) — escrever ou migrar `constants.py`, `tasks.py` e `flows.py` para o modelo em estágios
- [Abrir uma issue](como-fazer/abrir-issue.md)
- [Abrir um PR](como-fazer/abrir-pr.md)
- [Atualizar dependências](como-fazer/atualizar-dependencias.md) — Python (`uv`) e dbt packages

## Referência

- [`stage_dispatch.py`](referencia/stage-dispatch.md) — tipos e funções usados para encadear os estágios
