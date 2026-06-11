# Guia de Onboarding de Metadados no Backend Data Basis

Este guia descreve, passo a passo, como popular o backend da Base dos Dados ao
publicar um novo conjunto de dados — desde a criação do `Dataset` até a tabela
no BigQuery (`CloudTable`), `RawDataSource`, `Coverage`, `DatetimeRange` e
`Update`. É a referência canônica para qualquer agente que registre metadados
em `development.backend.basedosdados.org` ou `backend.basedosdados.org`.

O exemplo usado é o dataset **CAGED — Cadastro Geral de Empregados e
Desempregados**:

- Slug: `caged`
- ID: `562b56a3-0b01-4735-a049-eeac5681f056`
- Admin (prod): https://backend.basedosdados.org/admin/v1/dataset/562b56a3-0b01-4735-a049-eeac5681f056/change/
- Organização: `me` (Ministério da Economia)
- Tema: `economics`
- 6 `RawDataSource` (microdados oficiais + power BI estatístico)

---

## Pré-requisitos

1. Arquitetura de colunas já definida em planilha Google Sheets, seguindo
   `.claude/rules/data-basis-style.md`.
2. Dados já carregados em `basedosdados-dev.<gcp_dataset_id>_staging.<table_slug>`.
3. Modelo dbt rodando com sucesso em `dev` (vide `.claude/rules/dbt-conventions.md`).
4. Credenciais do backend em `~/.basedosdados/credentials.json` ou em
   `EMAIL`/`PASSWORD` no ambiente; chame `mcp__databasis__auth` antes de
   qualquer operação de escrita.

> **Ambiente padrão:** `dev`. Promover para `prod` somente após aprovação
> humana explícita no checkpoint do workflow.

---

## Visão geral do modelo de dados

```
Dataset (1) ──┬── (N) Table ──┬── (N) Column
              │                ├── (N) ObservationLevel
              │                ├── (1) CloudTable          ← liga ao BigQuery
              │                ├── (N) Coverage ── DatetimeRange
              │                └── (N) Update
              │
              ├── (N) RawDataSource ── (N) Coverage
              ├── (N) Organization   (M2M)
              ├── (N) Theme          (M2M)
              ├── (N) Tag            (M2M)
              └── status_id          (FK)
```

A ordem de criação **importa**:

1. `Dataset` (slug, nomes, descrições, FKs de catálogo).
2. `RawDataSource` (vinculados ao dataset).
3. Para cada tabela:
   a. `Table` (sem `raw_data_source_ids` ainda).
   b. `ObservationLevel` (uma por entidade observada).
   c. Colunas via `upload_columns_from_sheet`.
   d. Ajustes coluna-a-coluna (`is_partition`, `is_primary_key`, EN/ES).
   e. `CloudTable` (ponteiro para BigQuery).
   f. `Coverage` + `DatetimeRange`.
   g. `Update` (frequência/lag).
   h. Re-`create_update_table` com `raw_data_source_ids` agora populado.
4. (Opcional) `reorder_tables` e `reorder_observation_levels`.

---

## Walkthrough — usando o CAGED como caso

### Passo 0 — Resolver IDs de referência

Tabelas auxiliares como `Status`, `License`, `Availability`, `Language`,
`Theme`, `Organization`, `Tag`, `Entity`, `MeasurementUnitCategory` têm IDs
distintos em dev e prod. **Nunca chumbar IDs no código.** Use:

```python
mcp__databasis__discover_ids(env="prod")
# ou, mais barato:
mcp__databasis__lookup_id(env="prod", category="organization", slug="me")
mcp__databasis__lookup_id(env="prod", category="theme",        slug="economics")
mcp__databasis__lookup_id(env="prod", category="status",       slug="published")
```

Exemplo de saída (`prod`, parcial):

| Categoria | Slug | UUID |
|---|---|---|
| status | published | `e16221de-ac30-4926-83d3-de219998dab3` |
| license | cc_by | `92211312-c1b7-4d21-80c5-fd6715b70e22` |
| availability | online | `dd396d7d-0264-4c1f-bf0d-6efe2dc89cbe` |
| language | pt | `fb034b5c-97e2-4678-8a89-6dc385d52658` |

### Passo 1 — Verificar se o dataset já existe

```python
ds = mcp__databasis__get_dataset(slug="caged", env="prod")
```

Se `ds.found == True`, capture `ds["id"]` para passar como `id=` nas chamadas
de atualização (operações são idempotentes).

### Passo 2 — Identificar o usuário autenticado

Necessário para preencher `published_by_ids` e `data_cleaned_by_ids` nas
tabelas:

```python
me = mcp__databasis__get_authenticated_account(env="prod")
account_id = me["id"]
```

### Passo 3 — Criar/atualizar o `Dataset`

```python
mcp__databasis__create_update_dataset(
    env="prod",
    slug="caged",
    name_pt="Cadastro Geral de Empregados e Desempregados (CAGED)",
    name_en="General Registry of Employed and Unemployed Workers (CAGED)",
    name_es="Registro General de Empleados y Desempleados (CAGED)",
    description_pt="O CAGED, instituído pela Lei nº 4.923/1965, é uma fonte mensal de informações sobre admissões e desligamentos de trabalhadores regidos pela CLT...",
    description_en="CAGED is a monthly administrative record of hires and dismissals of CLT-regulated workers in Brazil...",
    description_es="El CAGED es un registro administrativo mensual de altas y bajas de trabajadores regidos por la CLT en Brasil...",
    organization_ids=[<id_org_me>],
    theme_ids=[<id_tema_economics>],
    tag_ids=[],
    status_id=<id_status_published>,
)
```

### Passo 4 — Criar/atualizar `RawDataSource`s

O CAGED tem 6 fontes brutas (verificadas via `get_raw_data_sources`):

| # | Nome | URL |
|---|---|---|
| 1 | Microdados originais | http://pdet.mte.gov.br/microdados-rais-e-caged |
| 2 | Microdados Antigos | http://pdet.mte.gov.br/microdados-rais-e-caged |
| 3 | CAGED Estatístico | https://app.powerbi.com/view?... |
| 4 | Microdados Movimentação Fora Prazo | https://www.gov.br/.../microdados-rais-e-caged |
| 5 | Microdados Movimentação Excluida | https://www.gov.br/.../microdados-rais-e-caged |
| 6 | Microdados Movimentação | https://www.gov.br/.../microdados-rais-e-caged |

Para cada uma:

```python
mcp__databasis__create_update_raw_data_source(
    env="prod",
    dataset_id=<dataset_id>,
    name="CAGED - Microdados Movimentação",
    url="https://www.gov.br/trabalho-e-emprego/pt-br/assuntos/estatisticas-trabalho/microdados-rais-e-caged",
    availability_id=<id_avail_online>,
    language_ids=[<id_lang_pt>],
    license_id=<id_license_unknown>,
    contains_structured_data=True,
    contains_api=False,
    is_free=True,
    requires_registration=False,
    required_registration=False,
    required_ip_address=False,
)
```

Guarde os IDs retornados; serão necessários no passo 9.

### Passo 5 — Criar a `Table`

```python
table = mcp__databasis__create_update_table(
    env="prod",
    slug="microdados_movimentacao",
    dataset_id=<dataset_id>,
    name_pt="Microdados de Movimentação",
    name_en="Worker Movement Microdata",
    name_es="Microdatos de Movimiento",
    description_pt="Tabela de microdados...",
    description_en="...",
    description_es="...",
    status_id=<id_status_published>,
    published_by_ids=[account_id],
    data_cleaned_by_ids=[account_id],
    # NÃO passe raw_data_source_ids agora — deixe para o passo 9
)
table_id = table["id"]
```

### Passo 6 — `ObservationLevel`

Uma chamada por nível de observação (entidade observada). Para o CAGED a
unidade típica é o trabalhador no município por mês:

```python
ol_municipio = mcp__databasis__create_update_observation_level(
    env="prod",
    table_id=table_id,
    entity_id=<id_entity_municipio>,
)
ol_mes = mcp__databasis__create_update_observation_level(
    env="prod",
    table_id=table_id,
    entity_id=<id_entity_mes>,
)
```

### Passo 7 — Upload de colunas a partir da arquitetura

```python
mcp__databasis__upload_columns_from_sheet(
    env="prod",
    table_id=table_id,
    architecture_url="https://docs.google.com/spreadsheets/d/<sheet_id>/edit#gid=0",
    observation_levels=json.dumps({
        "id_municipio": ol_municipio["id"],
        "mes": ol_mes["id"],
    }),
)
```

Em seguida, ajustes por coluna (apenas campos não cobertos pelo upload em
lote):

```python
mcp__databasis__update_column(
    env="prod",
    column_id=<id>, table_id=table_id, column_name="ano",
    is_partition=True,
    description_en="Reference year",
    description_es="Año de referencia",
)
```

### Passo 8 — `CloudTable`

```python
mcp__databasis__create_update_cloud_table(
    env="prod",
    table_id=table_id,
    gcp_project_id="basedosdados",
    gcp_dataset_id="br_me_caged",
    gcp_table_id="microdados_movimentacao",
)
```

### Passo 9 — `Coverage`, `DatetimeRange` e `Update`

```python
cov = mcp__databasis__create_update_coverage(
    env="prod",
    table_id=table_id,
    area_id=<id_area_br>,
)

mcp__databasis__create_update_datetime_range(
    env="prod",
    coverage_id=cov["id"],
    start_year=2020, start_month=1,
    end_year=None,   end_month=None,
    interval=1,
    is_closed=False,
)

mcp__databasis__create_update_update(
    env="prod",
    table_id=table_id,
    frequency=1, entity_id=<id_entity_mes>,
    lag=45,
    last_updated="2026-05-25",   # SEMPRE a data de hoje, não a data do dado
)
```

### Passo 10 — Religar `raw_data_source_ids` (deferred update)

A M2M de raw sources só é populada num **segundo** `create_update_table`,
passando **todos** os campos obrigatórios de novo (a API não suporta update
parcial):

```python
mcp__databasis__create_update_table(
    env="prod",
    id=table_id,
    slug="microdados_movimentacao",
    dataset_id=<dataset_id>,
    name_pt="Microdados de Movimentação",
    name_en="Worker Movement Microdata",
    name_es="Microdatos de Movimiento",
    description_pt="...", description_en="...", description_es="...",
    status_id=<id_status_published>,
    published_by_ids=[account_id],
    data_cleaned_by_ids=[account_id],
    raw_data_source_ids=[<id1>, <id2>, ...],
)
```

### Passo 11 — Verificar

Verifique no admin:

- prod: https://backend.basedosdados.org/admin/v1/dataset/<id>/change/
- site:  https://basedosdados.org/dataset/<slug>

Ou via MCP:

```python
mcp__databasis__get_dataset(slug="caged", env="prod")
```

---

## Referência de campos

### `Dataset`

| Campo | Tipo | Origem | Notas |
|---|---|---|---|
| `slug` | string | manual | Identificador curto (`caged`, `cnuc`). **Não usar** `<org>_<slug>`. |
| `name_pt` / `name_en` / `name_es` | string | manual | Título exibido. Sempre os 3 idiomas. |
| `description_pt` / `_en` / `_es` | text | docs da fonte | 1–3 frases; técnico; sem listas. |
| `organization_ids` | list[UUID] | `lookup_id(category="organization")` | M2M. CAGED → `me`. |
| `theme_ids` | list[UUID] | `lookup_id(category="theme")` | M2M. CAGED → `economics`. |
| `tag_ids` | list[UUID] | `lookup_id(category="tag")` | M2M; pode ser vazio. |
| `status_id` | UUID | `discover_ids.status` | Use `published` no fim do onboarding. |
| `id` | UUID | `get_dataset` | Passar apenas em updates. |

### `RawDataSource`

| Campo | Tipo | Notas |
|---|---|---|
| `dataset_id` | UUID | obrigatório |
| `name` | string | título humano-legível da fonte |
| `url` | string | link público para a fonte |
| `availability_id` | UUID | `online` / `in_person` / `physical` / `unknown` |
| `language_ids` | list[UUID] | idioma do conteúdo da fonte |
| `license_id` | UUID | quando desconhecido, `unknown` |
| `contains_structured_data` | bool | `True` se a fonte fornece dados tabulares |
| `contains_api` | bool | há API pública? |
| `is_free` | bool | sem custo? |
| `requires_registration` / `required_registration` | bool | usar ambos por compatibilidade |
| `required_ip_address` | bool | precisa de IP institucional? |

> No CAGED, todas as 6 sources são `online`, `pt`, gratuitas e sem registro.

### `Table`

| Campo | Tipo | Notas |
|---|---|---|
| `slug` | string | snake_case sem prefixo de dataset (`microdados_movimentacao`) |
| `dataset_id` | UUID | obrigatório |
| `name_pt/en/es` | string | 3 idiomas |
| `description_pt/en/es` | text | descrição técnica da granularidade |
| `status_id` | UUID | `status.published` ao publicar |
| `published_by_ids` | list[UUID] | conta autenticada (`get_authenticated_account`) |
| `data_cleaned_by_ids` | list[UUID] | normalmente a mesma conta |
| `raw_data_source_ids` | list[UUID] | **adicionar apenas na chamada deferred** |
| `id` | UUID | passar em updates |

### `ObservationLevel`

| Campo | Tipo | Notas |
|---|---|---|
| `table_id` | UUID | tabela alvo |
| `entity_id` | UUID | `lookup_id(category="entity", slug="municipio"/"mes"/...)` |

A entidade descreve o que cada linha representa (município, mês, trabalhador,
estabelecimento). Use `reorder_observation_levels` se a ordem importar.

### `Column`

Carregadas em lote via `upload_columns_from_sheet`. A planilha de
arquitetura segue `.claude/rules/data-basis-style.md`:

| Coluna da planilha | Mapeia para | Notas |
|---|---|---|
| `name` | `Column.name` | snake_case |
| `bigquery_type` | `Column.bigquery_type_id` | resolve por slug |
| `description` | `Column.description_pt` | tradução para EN/ES via `update_column` |
| `temporal_coverage` | `Column.temporal_coverage` | notação `2020(1)2024` |
| `covered_by_dictionary` | bool | |
| `directory_column` | `Column.directory_primary_key` | `<dataset>.<table>:<col>` |
| `measurement_unit` | `Column.measurement_unit` | ex.: `BRL`, `year` |
| `has_sensitive_data` | bool | |
| `observations` | text | notas livres |
| `original_name` | string | nome na fonte bruta |

Pós-upload, complete com `update_column`:

- `is_partition=True` para colunas de partição (`ano`, `mes`, `sigla_uf`, `id_municipio`).
- `is_primary_key=True` para a PK lógica da granularidade.
- `description_en` / `description_es` se ainda não estiverem na planilha.

### `CloudTable`

| Campo | Tipo | Exemplo (CAGED dev) | Exemplo (CAGED prod) |
|---|---|---|---|
| `table_id` | UUID | id da `Table` | id da `Table` |
| `gcp_project_id` | string | `basedosdados-dev` | `basedosdados` |
| `gcp_dataset_id` | string | `br_me_caged` | `br_me_caged` |
| `gcp_table_id` | string | `microdados_movimentacao` | `microdados_movimentacao` |

Nota: o `gcp_dataset_id` segue o padrão `<org>_<slug>`, **não** o slug puro.

### `Coverage` + `DatetimeRange`

`Coverage` é a "casca" geográfica (área); `DatetimeRange` é o intervalo
temporal aninhado.

| Campo de Coverage | Notas |
|---|---|
| `table_id` ou `raw_data_source_id` | aninhar em uma das duas |
| `area_id` | `lookup_id(category="area", slug="br")` para datasets nacionais |

| Campo de DatetimeRange | Notas |
|---|---|
| `coverage_id` | UUID do Coverage |
| `start_year` / `start_month` / `start_day` | mínimo presente nos dados |
| `end_year` / `end_month` / `end_day` | `None` se em andamento |
| `interval` | `1` = anual; ajuste para mensal/diário |
| `is_closed` | `False` se série segue ativa |

### `Update`

Descreve frequência de atualização e *lag* (atraso típico até a fonte
publicar uma nova janela).

| Campo | Notas |
|---|---|
| `table_id` | obrigatório |
| `entity_id` | unidade do `frequency`. Mensal → entity `mes`. |
| `frequency` | inteiro; `1` + entity `mes` = mensal |
| `lag` | dias de atraso entre o período de referência e a publicação |
| `last_updated` | **data de hoje** — quando o BD subiu, não o max do dado |

---

## Checkpoint pós-dev (entre passos 9 e 10 do workflow)

Antes de promover para `prod`, imprima o checklist (formato em
`.claude/rules/onboarding-workflow.md`) e aguarde aprovação humana. **Nunca**
prossiga sem o "approved" explícito.

## Erros conhecidos

- **M2M aparentemente vazios** após salvar (organizations, themes, tags,
  raw_data_source_ids): chamadas idempotentes. Verifique com `get_dataset`
  após `create_update_*`. Se persistir vazio é problema de deploy do
  backend — registre e siga.
- **`upload_columns_from_sheet` em dev** silenciosamente ignora
  `directoryPrimaryKey` quando o dataset de diretórios não existe no
  ambiente. Comportamento esperado.
- **`publishedBy` permission denied** em `get_dataset`: o GraphQL retorna
  dados parciais com erro em `tables.edges[].node.publishedBy` para usuários
  sem permissão de admin. Funcional, mas a MCP atual eleva isso a falha;
  contornar via `search_datasets` + `get_raw_data_sources` quando necessário.

---

## Apêndice: ordem de execução resumida

```text
1.  discover_ids                       → IDs de status, theme, org, etc.
2.  get_dataset(slug)                  → existe? captura id
3.  get_authenticated_account          → account_id
4.  create_update_dataset              → dataset_id
5.  create_update_raw_data_source ×N   → raw_ids
loop por tabela:
6.  create_update_table                → table_id (sem raw_ids)
7.  create_update_observation_level ×N → ol_ids
8.  upload_columns_from_sheet          → colunas em lote
9.  update_column ×N                   → is_partition/is_pk/EN/ES
10. create_update_cloud_table          → ponteiro BQ
11. create_update_coverage             → cov_id
12. create_update_datetime_range       → intervalo temporal
13. create_update_update               → frequency/lag/last_updated=hoje
14. create_update_table(id=...)        → religa raw_data_source_ids
fim do loop
15. reorder_tables / reorder_obs_levels (opcional)
16. checkpoint → aprovação humana → repetir 4–15 em prod
```
