---
tipo: como-fazer
titulo: Como estruturar um flow de dataset
---
# Como estruturar um flow de dataset para o pipeline orientado a eventos

Passo a passo pra escrever (ou migrar) os arquivos de um dataset —
`constants.py`, `utils.py`, `tasks.py`, `flows.py` — na variante padrão do pipeline orientado
a eventos (`check_update` → `extract_and_load` → `build_and_promote`). 

Para entender por que a arquitetura é assim, ver
[Pipeline orientado a eventos](../explicacao/pipeline-orientado-a-eventos.md).

## Pré-requisitos

- Acesso ao repo `pipelines`, ambiente configurado (`uv sync`).
- Enquadrar o potencial flow em uma das duas categorias mapeadas:

   1. **Variante `check_update`**: O dataset permite checar a atualidade dos dados em um processo de **check leve**, ou seja, sem precisar baixar o arquivo (apenas via uma HEAD de request, ou uma API de metadata, ou listagem FTP/HTML);
   2. **Variante `check_and_download`**: Para checar a atualidade dos dados da BD em relação à fonte é preciso baixar e inspecionar o arquivo.

A variante `check_and_download` não tem código real (ver [explicação](../explicacao/pipeline-orientado-a-eventos.md), seção "Duas variantes")

- Abrir o código real do dataset (`pipelines/datasets/<dataset>/` e/ou `pipelines/crawler/<nome>/`) **antes de escrever qualquer linha** para saber quantas tabelas o dataset tem ou se o check é leve de verdade. 

## Passos

### 1. `pipelines/datasets/<dataset_id>/constants.py`
Povoar o arquivo com constantes tais como:
- `DATSET_ID`, 
- um `TABLE_ID` por tabela, 
- o tipo de `Coverage` de cada uma (`PartBdpro`/`AllFree`/`NonHistorical`, ver `pipelines/utils/metadata/domain.py`);
- valores imutáveis reaproveitados nas _tasks_ do _flow_.  

**Exemplo:**

```python
from pipelines.utils.metadata.domain import (
    DateFormat,
    DateOnly,
    NonHistorical,
    PartBdpro,
)

DATASET_ID = "br_rf_cnpj"
EMPRESAS_TABLE_ID = "empresas"
ESTABELECIMENTOS_TABLE_ID = "estabelecimentos"
SOCIOS_TABLE_ID = "socios"
SIMPLES_TABLE_ID = "simples"
DICIONARIO_TABLE_ID = "dicionario"

# Parâmetros de download
DOWNLOAD_CHUNK_SIZE = 15 * 1024 * 1024
DOWNLOAD_MAX_RETRIES = 5
...
```

### 2. `pipelines/datasets/<dataset_id>/tasks.py:get_latest_update`

Descobre a data de referência na fonte e devolve um `SourceInspection`. **Não
decide** se o dado é novo — isso é responsabilidade de
`poll_source_for_update_task`, chamado por dentro do encapsulamento.

```python
from pipelines.utils.stage_dispatch import SourceInspection

def make_get_latest_update(table_id):
    def get_latest_update() -> SourceInspection:
        # só informa a data mais recente encontrada na fonte — não decide
        # se é "nova", isso é papel do poll_source_for_update_task
        return SourceInspection(reference_date=data_encontrada)
    return get_latest_update

```

- Use fábrica, `make_get_latest_update`, se o dataset tem mais de uma tabela com
lógica de check distinta; 
- Se o check é uma função só, compartilhada por todas as
tabelas, passe `lambda _table_id: minha_funcao_unica` no lugar da fábrica.

### 3. `pipelines/datasets/<dataset_id>/tasks.py:extract_load_data`

Recebe o `download_params` da etapa anterior (sempre tem `reference_date`, mais o
que `SourceInspection.extra_download_params` tiver posto), baixa o dado de
verdade e devolve um `ExtractAndLoad`.

```python
from pipelines.utils.stage_dispatch import ExtractAndLoad


def make_extract_load_data(table_id):
    def extract_load_data(download_params: dict) -> ExtractAndLoad:
        ref = download_params["reference_date"]
        # baixa o dado de verdade, de forma independente do check_update
        return ExtractAndLoad(coverage=COVERAGE.model_dump(), data_path=caminho_local)
    return extract_load_data
```

**Nunca chama `upload_to_gcs` diretamente** — o encapsulamento
(`CheckThenExtractLoadPipeline.run_extract_and_load`) já faz isso depois, a
partir do `data_path` devolvido.

**Tabela particionada**: 
- `data_path` precisa ser um diretório já organizado na estrutura Hive final (`data_path/ano=2026/mes=09/arquivo.csv`)
- `data_path` não é um parâmetro que você passa pro `ExtractAndLoad`, é descoberto sozinho
(`discover_partition_folders`, ver [referência](../referencia/stage-dispatch.md)).

### 4. `pipelines/datasets/<dataset_id>/tasks.py:make_pipeline` 

Se o dataset tem mais de uma tabela, adicione aos passos acima `pipeline_factory`, **como última definição do arquivo**:

```python
from pipelines.utils.stage_dispatch import pipeline_factory

make_pipeline = pipeline_factory(DATASET_ID, make_get_latest_update, make_extract_load_data)
```

Dataset de tabela única pode pular isso e instanciar `CheckThenExtractLoadPipeline` direto em `flows.py`.

### 5. `pipelines/datasets/<dataset_id>/flows.py`

```python
from pipelines.utils.flow import flow
from pipelines.utils.stage_dispatch import Etapa, deploy_tags
from prefect.schedules import Cron

_pipeline = make_pipeline(MES_BRASIL_TABLE_ID)

@flow(name=_pipeline.check_update_flow_name, log_prints=True)
def br_ibge_ipca_mes_brasil_check_update() -> None:
    _pipeline.run_check_update()

br_ibge_ipca_mes_brasil_check_update.deploy_tags = deploy_tags(
    DATASET_ID, Etapa.CHECK_UPDATE, MES_BRASIL_TABLE_ID
)
br_ibge_ipca_mes_brasil_check_update.deploy_schedules = [
    Cron("40 14 8,9,10,11,12,13 * *", timezone="America/Sao_Paulo")
]

@flow(name=_pipeline.extract_and_load_flow_name, log_prints=True)
def br_ibge_ipca_mes_brasil_download(download_params: dict) -> None:
    _pipeline.run_extract_and_load(download_params)

br_ibge_ipca_mes_brasil_download.deploy_tags = deploy_tags(
    DATASET_ID, Etapa.EXTRACT_AND_LOAD, MES_BRASIL_TABLE_ID
)
_pipeline.extract_load_deployment = br_ibge_ipca_mes_brasil_download.fn.__name__
```

Pontos fixos, nessa ordem:

1. `_x_pipeline = make_pipeline(TABLE_ID)` (ou `CheckThenExtractLoadPipeline(...)`
   direto, se tabela única).
2. `@flow(name=_x_pipeline.check_update_flow_name, ...)` — nunca montar o nome à
   mão.
3. `deploy_tags(DATASET_ID, Etapa.CHECK_UPDATE, table_id=...)` logo depois —
   inclua `table_id` sempre que o dataset tiver mais de uma tabela.
4. Repita 2-3 pro flow de `extract_and_load`.
5. **Por último**, `_x_pipeline.extract_load_deployment = <flow>.fn.__name__` —
   sempre depois que a função existir, nunca uma string digitada à mão.

**Não existe `build_and_promote` no `flows.py` do dataset** — é um deployment
genérico único, já existente (`pipelines/utils/metadata/flows.py`); o
`extract_and_load` dispara ele sozinho.

Dataset multi-tabela: repita a seção (construção do pipeline + os 2 `@flow`) uma
vez por tabela, sempre explícita — nunca por fábrica de `@flow` compartilhada
(o `.fn.__name__` de uma closure reusada seria o mesmo literal pra todas as
instâncias).

### 6. Se está substituindo um flow monolítico antigo

Remova o código antigo do arquivo por completo antes de qualquer deploy pela sua
branch. `deploy_flows.py --files <arquivo>` redeploya todo `Flow` que encontrar
no arquivo, usando a branch informada como fonte git — deixar o flow antigo ali
sobrescreveria silenciosamente a fonte do deployment real (que deveria continuar
em `main`) pra apontar pra sua branch de trabalho.

### 7. Deploy e ativação

1. Push pra `main` dispara o deploy em produção — seletivo, só os flows que
   mudaram no PR.
2. Todo deployment novo nasce **pausado** (`is_schedule_active=False`). O CI
   registra isso no backend logo depois do deploy.
3. Ative o schedule pelo admin do backend (ou, via MCP databasis,
   `mcp__databasis__set_deployment_schedule_active`) — **nunca** despause direto
   no Prefect: o próximo sync reimpõe o estado salvo no backend e re-pausaria
   silenciosamente.

## Verificação

1. **Validação estática**: `uv run ruff check` + `uv run ruff format` e
   `uv run pyrefly check` nos 3 arquivos; confirme também que nenhum nome de
   `@flow`/deployment colide com outro dataset já migrado (import do módulo +
   checagem de `vars(module)` por objetos `Flow`).
2. **Teste em dev, sem tocar em produção**: deploy só do seu `flows.py`, pool
   `basedosdados-dev`, sem schedule (deploy em dev nunca ativa schedule
   automaticamente).
3. Dispare o `check_update` manualmente (`run_deployment`, ou pela UI do
   Prefect) e acompanhe os logs — se houver dado novo, ele deve disparar o
   `extract_and_load` sozinho (aparece linkado como subflow na árvore do Prefect
   UI), que por sua vez dispara o `build_and_promote`.
4. Se a coverage já estiver em dia (sem dado novo pra detectar), isso **não é um
   bug** — é o comportamento esperado. Forçar um teste completo exige rolar a
   coverage do backend pra trás, decisão que precisa de aprovação explícita
   (mexe em dado real do backend).
5. Confirme que `promote_to_prod` só se comporta como esperado quando o teste
   roda de fato no pool de prod — um `extract_and_load` disparado em dev nunca
   deve promover pra prod de verdade (ver
   [explicação](../explicacao/pipeline-orientado-a-eventos.md)).

## Problemas comuns

- **`get_latest_update` parece não disparar nada** — ele só informa a data
  encontrada, não decide se é "nova". Quem decide é `poll_source_for_update_task`,
  chamado por dentro de `check_update_and_dispatch`. Comparar a lógica de decisão
  contra o backend, não contra o seu código.
- **Dado sobe duplicado ou não sobe** — `extract_load_data` nunca deve chamar
  `upload_to_gcs` diretamente; isso já é feito pela cápsula depois que a função
  retorna.
- **Tabela particionada não promove a partição certa** — `data_path` precisa
  apontar pra um diretório já na estrutura Hive final; partições são descobertas
  automaticamente a partir dele, nunca passadas manualmente.
- **Flow antigo continua rodando em paralelo ao novo** — migrar um dataset não
  desliga o flow monolítico antigo sozinho; ele continua registrado (ativo ou
  pausado) no Prefect até ser removido manualmente, depois de confirmar que o
  novo está funcionando.
- **`poll_source_for_update` falha silenciosamente** — quebra pra tabela sem
  `RawDataSource` cadastrado no backend (o filtro correspondente é ignorado em
  vez de filtrar por nulo). Cadastre o `RawDataSource` antes de migrar.
- **Tabela nova não aparece em `basedosdados-dev`** — `bd.Table.create()` só
  registra o ponteiro externo do BigQuery em `basedosdados-staging`; pode exigir
  criação manual da tabela em dev antes do primeiro `dbt run` nesse ambiente.
- **Fonte compara por tamanho de arquivo, não por data** — `stage_dispatch.py`
  ainda não cobre esse caso (ex. `br_bcb_sicor`); precisa de desenho à parte
  antes de migrar.

## Ver também

- [Pipeline orientado a eventos](../explicacao/pipeline-orientado-a-eventos.md) — por que a arquitetura é assim
- [Referência — `stage_dispatch.py`](../referencia/stage-dispatch.md) — tipos e funções usados neste guia
- [issue #1867](https://github.com/basedosdados/pipelines/issues/1867) e [PR #1932](https://github.com/basedosdados/pipelines/pull/1932) — histórico completo e exemplos reais de migração
