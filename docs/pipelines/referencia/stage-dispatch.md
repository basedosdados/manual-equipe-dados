---
tipo: referencia
titulo: stage_dispatch.py
---
# stage_dispatch.py — building blocks do pipeline orientado a eventos

## Resumo

| Campo | Valor |
|---|---|
| O que é | Building blocks compartilhados pra encadear os flows de um dataset migrado (`check_update` → `extract_and_load` → `build_and_promote`) via `run_deployment()` |
| Fonte | [`pipelines/utils/stage_dispatch.py`](https://github.com/basedosdados/pipelines/blob/main/pipelines/utils/stage_dispatch.py) (repo `basedosdados/pipelines`) |
| Atualização | Acompanha mudanças de código, sem cadência fixa — ler o arquivo direto é a fonte de verdade mais atual |
| Dono | Equipe de dados — pipelines |

## Definição

`Etapa`, `SourceInspection`, `ExtractAndLoad`: tipos compartilhados entre as
etapas. `deploy_tags`/`deployment_name`/`_flow_name`: convenção de nome de
deployment. `check_update_and_dispatch`/`dispatch_build_and_promote`: dispatch
entre etapas via `run_deployment()`. `pipeline_factory`/
`CheckThenExtractLoadPipeline`: interface recomendada pra um `flows.py` de
dataset na variante padrão (`check_update` e `extract_and_load` como estágios
separados). Ver [explicação](../explicacao/pipeline-orientado-a-eventos.md) pro
racional de design.

### `Etapa` (enum, `StrEnum`)

| Valor | Significado |
|---|---|
| `CHECK_UPDATE` = `"check_update"` | Verifica se há dado novo na fonte |
| `EXTRACT_AND_LOAD` = `"extract_and_load"` | Baixa o dado e sobe pro staging |
| `BUILD_AND_PROMOTE` = `"build_and_promote"` | Materializa, testa e promove pra prod |

## Schema / Parâmetros

### `SourceInspection` (dataclass)

O que `get_latest_update` de um dataset devolve pro `check_update`.

| Nome | Tipo | Descrição |
|---|---|---|
| `reference_date` | `datetime.date` | Data de referência encontrada na fonte |
| `extra_download_params` | `dict` (default `{}`) | Campos extra que `extract_load_data` vai precisar |
| `compare_against` | `str` (default `"coverage"`) | `"coverage"` lê `Coverage.DateTimeRange`; `"table_update"` lê `Table.Update.latest` — necessário pra tabelas `NonHistorical`, sem baseline de coverage confiável |

### `ExtractAndLoad` (dataclass)

O que `extract_load_data` de um dataset devolve pro `extract_and_load`.

| Nome | Tipo | Descrição |
|---|---|---|
| `coverage` | `dict` | `CoverageSpec.model_dump()` (`AllFree`/`AllBdpro`/`PartBdpro`/`NonHistorical`, ver `pipelines.utils.metadata.domain`) |
| `data_path` | `str` | Caminho local escrito por `extract_load_data`. Tabela particionada: diretório já na estrutura Hive que será promovida (ex. `data_path/ano=2026/mes=09/dados.csv`) |
| `prefect_mode` | `str` (default `"prod"`) | `"dev"`/`"prod"` — resolve o projeto BigQuery de destino automaticamente (`MODE_PROJECT`) |
| `dump_mode` | `str` (default `"append"`) | Modo de escrita no BigQuery |
| `source_format` | `str` (default `"csv"`) | Formato do arquivo em `data_path` |
| `partition_folders` | `list[str] \| None` (`init=False`) | Pastas Hive atualizadas nesta execução — só elas são promovidas pra prod. Preenchido automaticamente por `discover_partition_folders(data_path)`, nunca pelo dataset; `None` se não há partição |

### `discover_partition_folders(data_path: str) -> list[str] | None`

Acha as pastas-folha Hive (`chave=valor`) escritas em `data_path`. Genérico pra
qualquer profundidade de partição — não precisa saber de antemão quais partições
existem, só que `data_path` segue a convenção Hive. Devolve `None` se `data_path`
não existe ou não tem subpasta (tabela sem partição).

### `deploy_tags(dataset_id, etapa, table_id=None) -> list[str]`

Tags de deploy pra achar deployments relacionados no Prefect UI/CI. Usar em
`<flow>.deploy_tags = deploy_tags(...)`.

| Parâmetro | Tipo | Descrição |
|---|---|---|
| `dataset_id` | `str` | ID do dataset |
| `etapa` | `Etapa` | Etapa do flow |
| `table_id` | `str \| None` | ID da tabela, quando o dataset tem mais de uma — adiciona a tag `<dataset_id>__<table_id>` |

Retorna `["staged-pipeline", str(etapa), dataset_id]`, mais
`f"{dataset_id}__{table_id}"` quando `table_id` é passado.

### `deployment_name(dataset_id, etapa, deployment=None) -> str`

Resolve o identificador `"<flow name>/<deployment name>"` — o formato aceito por
`run_deployment(name=...)`.

- `etapa == BUILD_AND_PROMOTE`: sempre `"build_and_promote/build_and_promote"`
  (deployment único, genérico, sempre no pool de prod — nunca leva o prefixo
  `dev-`, mesmo quando quem despacha rodou em dev).
- Caso contrário: `f"{_flow_name(dataset_id, etapa)}/{prefix}{deployment or str(etapa)}"`,
  onde `prefix` é `"dev-"` quando a tag `env:dev` está presente no flow run atual
  (`_dev_prefix()`), `""` caso contrário.

### `check_update_and_dispatch(...) -> bool`

Encapsula o padrão de `check_update`: `poll_source_for_update_task` decide se há
dado novo; se houver, `commit_source_update_task` comita o `Update` e dispara o
deployment da próxima etapa via `run_deployment()` (`timeout=0`,
`as_subflow=True`).

| Parâmetro | Tipo | Descrição |
|---|---|---|
| `prefect_dataset_id` | `str` | Convenção usada por `deployment_name()` — normalmente `f"{dataset_id}__{table_id}"` |
| `dataset_id` / `table_id` | `str` | IDs no backend/BigQuery |
| `reference_date` | `datetime.date` | Data de referência encontrada na fonte |
| `next_etapa` | `Etapa` (default `EXTRACT_AND_LOAD`) | Etapa a disparar em seguida |
| `next_deployment` | `str \| None` | Sobrescreve o nome do deployment, se a variável do flow não se chamar `<etapa>` |
| `compare_against` | `str` (default `"coverage"`) | Ver `SourceInspection.compare_against` |

Retorna `True` se havia dado novo e o próximo estágio foi disparado.

### `dispatch_build_and_promote(dataset_id, table_id, result: ExtractAndLoad, env="prod") -> None`

Dispara o `build_and_promote` genérico via `run_deployment()`. `promote_to_prod`
**não vem de `result`**: é `not _dev_prefix()` — derivado de onde quem despachou
está rodando, nunca configurado pelo dataset.

### `pipeline_factory(dataset_id, get_latest_update_factory, extract_load_data_factory, **shared_kwargs)`

Fábrica de `CheckThenExtractLoadPipeline` pra datasets multi-tabela. Fixa
`dataset_id` e kwargs comuns a todas as tabelas; devolve uma função
`table_id -> CheckThenExtractLoadPipeline`.

### `CheckThenExtractLoadPipeline`

Interface recomendada pra um `flows.py` de dataset com `check_update` e
`extract_and_load` como estágios separados.

| Atributo/método | Descrição |
|---|---|
| `dataset_id`, `table_id` | IDs no backend/BigQuery |
| `get_latest_update` | Callable `() -> SourceInspection` com a lógica específica do dataset |
| `extract_load_data` | Callable `(dict) -> ExtractAndLoad` com a lógica específica do dataset |
| `prefect_dataset_id` | Default `f"{dataset_id}__{table_id}"` — convenção universal do repo, mesmo com 1 tabela só |
| `extract_load_deployment` | Nome do deployment de `extract_and_load` a disparar — setar **depois** que o flow existir, a partir de `<flow>.fn.__name__` |
| `check_update_flow_name` (property) | Nome formatado pro `@flow(name=...)` do `check_update` |
| `extract_and_load_flow_name` (property) | Nome formatado pro `@flow(name=...)` do `extract_and_load` |
| `run_check_update()` | Corpo completo do estágio `check_update` — renomeia o flow run, chama `get_latest_update`, e `check_update_and_dispatch` |
| `run_extract_and_load(download_params)` | Corpo completo do estágio `extract_and_load` — renomeia o flow run, chama `extract_load_data`, descobre partições, `upload_to_gcs`, e `dispatch_build_and_promote` |

## Exemplo

Dataset de tabela única, sem `pipeline_factory`:

```python
from pipelines.utils.flow import flow
from pipelines.utils.stage_dispatch import (
    CheckThenExtractLoadPipeline,
    Etapa,
    SourceInspection,
    ExtractAndLoad,
    deploy_tags,
)

def get_latest_update() -> SourceInspection:
    return SourceInspection(reference_date=data_encontrada)

def extract_load_data(download_params: dict) -> ExtractAndLoad:
    return ExtractAndLoad(coverage=COVERAGE.model_dump(), data_path=caminho_local)

_pipeline = CheckThenExtractLoadPipeline(
    dataset_id=DATASET_ID,
    table_id=TABLE_ID,
    get_latest_update=get_latest_update,
    extract_load_data=extract_load_data,
)

@flow(name=_pipeline.check_update_flow_name, log_prints=True)
def check_update() -> None:
    _pipeline.run_check_update()
check_update.deploy_tags = deploy_tags(DATASET_ID, Etapa.CHECK_UPDATE)

@flow(name=_pipeline.extract_and_load_flow_name, log_prints=True)
def extract_and_load(download_params: dict) -> None:
    _pipeline.run_extract_and_load(download_params)
extract_and_load.deploy_tags = deploy_tags(DATASET_ID, Etapa.EXTRACT_AND_LOAD)
_pipeline.extract_load_deployment = extract_and_load.fn.__name__
```

Exemplo real multi-tabela, com `pipeline_factory`: ver
[como estruturar um flow de dataset](../como-fazer/estruturar-um-flow-de-dataset.md).

## Ver também

- [Pipeline orientado a eventos](../explicacao/pipeline-orientado-a-eventos.md) — por que a arquitetura é assim
- [Como estruturar um flow de dataset](../como-fazer/estruturar-um-flow-de-dataset.md)
- [issue #1867](https://github.com/basedosdados/pipelines/issues/1867) e [PR #1932](https://github.com/basedosdados/pipelines/pull/1932)
