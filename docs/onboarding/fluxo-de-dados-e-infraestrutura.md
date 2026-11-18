---
tipo: tutorial
titulo: Fluxo de dados e infraestrutura
---

# Fluxo de dados e infraestrutura

Guia de onboarding com o retrato dos componentes de infraestrutura usados pela equipe dados e de como o dado se move entre eles, da fonte original até a publicação no projeto `basedosdados`.

> Retrato da infraestrutura em novembro de 2026, já no Prefect 3. A [página de infraestrutura no site da BD](https://basedosdados.org/docs/colab_infrastructure#sistema-de-ingestão-de-dados) descreve a arquitetura antiga.

## Componentes da infraestrutura

| Componente | Tecnologia | Papel |
|---|---|---|
| Orquestrador | [Prefect 3](https://docs.prefect.io/v3/) — `prefect3.basedosdados.org` | Agenda e executa as [pipelines](../glossario.md#p). Os flows rodam em pods no Kubernetes, disparados pelos workers. Ver [Arquitetura do Prefect 3](../infraestrutura/prefect/explicacao/arquitetura-prefect-3.md). |
| Repositório de código | [`basedosdados/pipelines`](https://github.com/basedosdados/pipelines) | Hospeda os flows, os modelos dbt e as GitHub Actions de CI, deploy e `table-approve`. |
| Storage — dev | GCS, bucket **`basedosdados-dev`** | Recebe os arquivos tratados (parquet/csv) que alimentam as tabelas externas de dev. |
| Storage — prod | GCS, bucket **`basedosdados`** | Recebe os mesmos arquivos depois da validação em dev. A versão anterior vai para o bucket `basedosdados-backup`. |
| Data warehouse — dev | BigQuery, projeto **`basedosdados-dev`** | Guarda **as duas camadas**: `<dataset>_staging` (tabelas externas sobre o bucket `basedosdados-dev`) e `<dataset>` (modelos materializados pelo dbt). Usado pela equipe para validação. |
| Data warehouse — prod (staging) | BigQuery, projeto **`basedosdados-staging`** | Tabelas externas `<dataset>_staging` sobre o bucket `basedosdados`. **Não é exposto ao público.** |
| Data warehouse — prod (público) | BigQuery, projeto **`basedosdados`** | Modelos materializados pelo dbt a partir de `basedosdados-staging`. É o projeto consultado pelos usuários no site e no pacote Python. |
| Transformação e testes | [dbt](https://www.getdbt.com/) | Em dev (`target=dev`): lê `basedosdados-dev.<dataset>_staging` e materializa em `basedosdados-dev.<dataset>`. Em prod (`target=prod`): lê `basedosdados-staging.<dataset>_staging` e materializa em `basedosdados.<dataset>`. |
| Metadados | API GraphQL Django — `backend.basedosdados.org` (prod) / `staging.backend.basedosdados.org` (staging) | Fonte da verdade dos metadados expostos no site e no pacote Python. |
| CI/CD | GitHub Actions | `ci.yaml` (testes do PR e labels `test-dev-model` / `check-metadata`), `cd-prefect3.yaml` (deploy dos flows no Prefect 3) e `table-approve.yaml` (promoção para prod no merge). |

## Visão geral do fluxo

```mermaid
flowchart TD
    SRC[Fonte original<br/>API / site / FTP]
    EXT[Extração + tratamento<br/>Python]
    GCSDEV[(GCS<br/>basedosdados-dev)]
    BQDEVSTG[(BigQuery<br/>basedosdados-dev.dataset_staging)]
    BQDEV[(BigQuery<br/>basedosdados-dev.dataset)]
    GATE{Aprovado?}
    GCSPROD[(GCS<br/>basedosdados)]
    BQPRODSTG[(BigQuery<br/>basedosdados-staging.dataset_staging<br/>tabelas externas)]
    BQPROD[(BigQuery<br/>basedosdados.dataset<br/>público)]
    API[(Backend GraphQL<br/>backend.basedosdados.org)]
    SITE[Site / pacote Python]

    SRC --> EXT
    EXT --> GCSDEV
    GCSDEV -. tabela externa .-> BQDEVSTG
    BQDEVSTG -- dbt run/test<br/>target=dev --> BQDEV
    BQDEV -- testes dbt --> GATE
    GATE -- sim --> GCSPROD
    GCSPROD -. tabela externa .-> BQPRODSTG
    BQPRODSTG -- dbt run<br/>target=prod --> BQPROD
    BQPROD -. cobertura e atualização .-> API
    BQPROD --> SITE
    API --> SITE

    subgraph DEV[Zona de desenvolvimento]
        GCSDEV
        BQDEVSTG
        BQDEV
    end
    subgraph PROD[Zona de produção]
        GCSPROD
        BQPRODSTG
        BQPROD
    end
```

A diferença entre [pipeline automatizada](../glossario.md#p) e [semi-automatizada](../glossario.md#p) está em **como** o portão "Aprovado?" é cruzado e **quem** dispara a promoção para prod.

## 1. Pipelines automatizadas (Prefect 3)

A pipeline roda no schedule declarado no próprio `flows.py` do dataset (`deploy_schedules`, com `Cron`). Hoje convivem dois desenhos de flow.

### Pipeline em estágios (desenho atual)

Os datasets migrados dividem o trabalho em três flows encadeados, cada um no seu pod. Exemplo canônico: [`br_bcb_agencia`](https://github.com/basedosdados/pipelines/blob/main/pipelines/datasets/br_bcb_agencia/flows.py), que usa `CheckThenExtractLoadPipeline` de [`stage_dispatch.py`](../pipelines/referencia/stage-dispatch.md).

```mermaid
sequenceDiagram
    autonumber
    participant SCH as Schedule (cron)
    participant CU as check_update<br/>(pod pequeno)
    participant SRC as Fonte original
    participant API as Backend GraphQL
    participant EL as extract_and_load
    participant GDEV as GCS<br/>basedosdados-dev
    participant BP as build_and_promote<br/>(pool de prod)
    participant GPRD as GCS<br/>basedosdados
    participant BQP as BQ basedosdados<br/>(público)

    SCH->>CU: dispara run
    CU->>SRC: qual a data mais recente publicada?
    CU->>API: qual a data mais recente na BD?
    Note over CU: sem dado novo, o flow termina aqui
    CU->>EL: run_deployment
    EL->>SRC: baixa e trata
    EL->>GDEV: upload_to_gcs
    EL->>BP: run_deployment
    BP->>BP: dbt run/test target=dev
    Note over BP: teste falhou: nada vai para prod
    BP->>GPRD: upload_to_gcs
    BP->>BQP: dbt run/test target=prod
    BP->>API: register_table_materialization_task
```

**Pontos importantes:**

- `check_update` só compara datas. Os pods de download e de dbt sobem apenas quando há dado novo.
- `build_and_promote` é um deployment genérico, compartilhado por todos os datasets migrados, e roda sempre no pool de prod.
- O `dbt run/test` em `target=dev` é o **gate de qualidade**: se um teste falhar, o passo de prod não acontece. Em prod, o dbt também roda os testes.
- `register_table_materialization_task` lê o BigQuery e atualiza no backend a cobertura temporal (aberta e BD Pro), as row access policies, quando a tabela tem, e a data da última atualização.

O desenho completo, com o porquê de cada etapa, está em [Pipeline orientado a eventos](../pipelines/explicacao/pipeline-orientado-a-eventos.md). Para escrever um flow nesse formato, veja [Estruturar um flow de dataset](../pipelines/como-fazer/estruturar-um-flow-de-dataset.md).

### Flow único (datasets ainda não migrados)

Os datasets ainda não migrados fazem tudo num flow só, num único pod: baixam, sobem para `basedosdados-dev` (`upload_to_gcs`), rodam `run_dbt` em `target=dev`, sobem para `basedosdados`, rodam `run_dbt` em `target=prod` e chamam `register_table_materialization_task`. Os passos e o gate são os mesmos, mas o pod tem o mesmo tamanho do começo ao fim, mesmo nas execuções em que não há dado novo.

## 2. Códigos semi-automatizados

[Pipelines semi-automatizadas](../glossario.md#p) são modelos dbt + script de carga executados manualmente pelo dev, com a materialização em produção disparada pelo merge de um PR com a label `table-approve`.

### Passo a passo do colaborador

1. Baixar a pasta template e os dados originais.
2. Preencher as **tabelas de arquitetura** e marcar a equipe de dados na issue ao finalizar.
3. Escrever pipeline de carregamento dos dados (script Python que sobe arquivos para `basedosdados-dev`).
4. Subir as tabelas em `basedosdados-dev.<dataset>_staging` no BigQuery (tabela externa sobre o GCS).
5. Escrever modelos **dbt** para transformação em `models/<dataset>/`.
6. Escrever testes **dbt**.
7. Organizar arquivos auxiliares, se necessário.
8. Criar tabela `dicionario`, se necessário.
9. Preencher os metadados na API do backend, deixando a tabela como `under_review`.
10. Abrir o PR.
11. Aplicar as labels **`test-dev-model`** e **`check-metadata`** no PR.
12. Enviar para revisão.

### Deploy via `table-approve`

Quando a staging de dev está completa e final, a revisão aplica a label **`table-approve`**. A label sozinha não dispara nada: a Action [`table-approve.yaml`](https://github.com/basedosdados/pipelines/blob/main/.github/workflows/table-approve.yaml) roda em todo push na `main` e, se o PR mergeado tiver a label, chama o script `prefect_run_dbt.py`. Para cada modelo `.sql` alterado no PR, o script dispara no Prefect 3 o flow `transfer_files_to_prod_flow`, que:

1. Faz backup da staging de prod no bucket `basedosdados-backup` e copia os arquivos de `basedosdados-dev` para `basedosdados`.
2. Recria a tabela externa em `basedosdados-staging.<dataset>_staging`.
3. Roda `dbt run` com `target=prod` e exporta o CSV de download.
4. Registra a materialização no backend (cobertura e atualização).

```mermaid
flowchart LR
    DEV[Desenvolvedor] -->|sobe dados em dev| GDEV[(GCS<br/>basedosdados-dev)]
    GDEV -. externa .-> BQDS[(BQ<br/>basedosdados-dev.dataset_staging)]
    BQDS -- dbt run/test target=dev<br/>local ou CI --> BQD[(BQ basedosdados-dev)]
    DEV -->|labels test-dev-model<br/>+ check-metadata| PR[PR no GitHub]
    PR -->|revisão aplica table-approve<br/>e o PR é mergeado| ACT[Action table-approve.yaml]
    ACT -->|prefect_run_dbt.py| TF[transfer_files_to_prod_flow<br/>no Prefect 3]
    TF --> GPRD[(GCS<br/>basedosdados)]
    GPRD -. externa .-> BQPS[(BQ<br/>basedosdados-staging.dataset_staging)]
    BQPS -- dbt run target=prod --> BQP[(BQ basedosdados.dataset<br/>público)]
    DEV -->|metadados under_review| API[(Backend GraphQL)]
```

A diferença prática com a pipeline automatizada: a promoção é disparada pela **Action**, no merge, e não pelo schedule; e o gatilho é humano (a label). O flow que faz a cópia e a materialização é o mesmo que o `build_and_promote` usa. Os detalhes e os riscos (troca da staging antes da materialização, esquema da tabela externa) estão em [Table-approve](../pipelines/explicacao/table-approve.md).

## 3. Preenchimento de metadados na API do backend

> **Antes de preencher qualquer metadado, leia o [Manual de estilo](../governanca/metadados/explicacao-manual-de-estilo.md).** Ele é a referência canônica para nomes de datasets/tabelas/colunas, tipos no BigQuery, formatos de data, padronização de UF/município e estrutura de diretórios. Sem aderência ao manual, os metadados ficam inconsistentes com o que está materializado em `basedosdados`, e a tabela é barrada na revisão.

A API GraphQL Django é a fonte da verdade dos metadados que aparecem no site e no pacote `basedosdados`. Endpoints:

- **Produção** — `https://backend.basedosdados.org/api/v1/graphql`
- **Staging** — `https://staging.backend.basedosdados.org/api/v1/graphql`

Pontos de contato a partir do repositório `pipelines` (código em `pipelines/utils/metadata/`):

- **`check_update`** (pipeline em estágios): lê do backend a data mais recente da tabela (cobertura ou última atualização) para decidir se há dado novo.
- **`register_table_materialization_task`**: ao final da materialização em prod, pelo flow ou pelo `table-approve`, atualiza a cobertura temporal, as row access policies e a data da última atualização.
- **Fluxo semi-automatizado**: o colaborador cadastra a tabela no admin do backend, deixando-a `under_review` antes do PR.

Para o passo a passo, veja [Subir metadados](../governanca/metadados/como-fazer/subir-metadados.md) (interface do Django) e [Registrar metadados no backend](../infraestrutura/prefect/como-fazer/registrar-metadados.md) (pelo Prefect).

## Ver também

- [Setup do repositório de pipelines](setup-pipelines.md) — como preparar o ambiente antes de mexer em qualquer um destes fluxos.
- [Pipeline orientado a eventos](../pipelines/explicacao/pipeline-orientado-a-eventos.md) — por que o flow foi dividido em três etapas.
- [Table-approve](../pipelines/explicacao/table-approve.md) — como a promoção de dev para prod funciona no merge.
- [Arquitetura do Prefect 3](../infraestrutura/prefect/explicacao/arquitetura-prefect-3.md) — servidor, workers e pools.
- [Glossário](../glossario.md) — definições de **Pipeline**, **Pipeline semi-automatizada**, **BD Pro**.
- [`basedosdados/pipelines` — CONTRIBUTING.md](https://github.com/basedosdados/pipelines/blob/main/CONTRIBUTING.md) — referência operacional para o passo a passo do colaborador semi-automatizado.
