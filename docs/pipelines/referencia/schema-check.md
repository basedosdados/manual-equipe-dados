---
tipo: referencia
titulo: schema_check.py
---
# `schema_check.py`: checagem de schema no `upload_to_gcs`

## Resumo

### O que é

O `schema_check.py` compara as colunas do arquivo que vai subir para o GCS com o schema da tabela externa de staging no BigQuery e classifica cada diferença como **warning** (o upload segue e avisa) ou **erro** (o upload é abortado antes de alterar a staging ou o bucket).

### Fonte

- [`pipelines/utils/schema_check.py`](https://github.com/basedosdados/pipelines/blob/main/pipelines/utils/schema_check.py): regras e modelos.
- [`pipelines/utils/tasks.py`](https://github.com/basedosdados/pipelines/blob/main/pipelines/utils/tasks.py): integração na task `upload_to_gcs`.
- PR [#2228](https://github.com/basedosdados/pipelines/pull/2228) (repositório `basedosdados/pipelines`).

Esta página descreve o que existe na branch `main`. Para o racional, leia a [explicação](../explicacao/checagem-de-schema-no-upload.md).

## Regras de classificação

O schema de referência é a staging atual. O **schema final esperado** é a staging mais as colunas novas, no fim.

| Situação | CSV | Parquet |
|---|---|---|
| Sem mudanças | segue | segue |
| a1: coluna a mais | **warning** se estiver no fim do arquivo; no meio, **erro** (desalinha) | **warning** |
| a2: coluna faltando | **erro**, no fim ou no meio | **erro** |
| a3: ordem diferente | **erro** | warning informativo (casa por nome) |
| a4: tipo incompatível | não se aplica (tudo `STRING`) | **erro** |
| a5: layout de partição diferente | **erro** | **erro** |
| Chave de partição repetida no header | **erro** | warning |

No `dump_mode="overwrite"`, todos os erros viram warning, com a nota de que a staging foi substituída e de que é preciso revisar os modelos dbt dependentes.

Detalhes das regras:

- **CSV (a1 + a2 + a3):** o header tem que ser prefixo do schema final, posição por posição. Só a coluna nova **no fim** vira warning.
- **Renomeação:** aparece como a1 + a2 (uma coluna some, outra surge). É erro no `append` e warning no `overwrite`.
- **Parquet, a4:** os aliases são normalizados (`INTEGER`=`INT64`, `FLOAT`=`FLOAT64`, `BOOLEAN`=`BOOL`) e a comparação entre o tipo do arquivo (pyarrow) e o da staging é de igualdade estrita. Tipo desconhecido (`data_type=None`) fica fora da checagem.
- **Nomes:** comparados normalizados com `bq_safe_column_name`, dos dois lados.
- **Chaves de partição:** ficam fora da comparação de colunas, dos dois lados. As do upload vêm das pastas `chave=valor` relativas a `data_path`; as da staging, de `table._properties["partitionDefinition"]`. Se não for possível ler as chaves (`None`), o layout não é verificado e a mensagem traz um warning.

## Mapa do módulo

| Grupo | Itens | Para que servem |
|---|---|---|
| Enums e resultado de regra | `SourceFormat`, `Severity`, `Finding` | Formato (`csv` ou `parquet`), gravidade e uma mensagem classificada |
| Modelo de schema | `Column`, `TableSchema`, `UploadSchema`, `StagingSchema` | Representar o schema do arquivo e o da staging |
| Fatos | `SchemaDiff`, `TypeMismatch` | Calcular as diferenças, sem juízo de gravidade |
| Regras | `_csv_findings`, `_parquet_findings`, `_layout_findings`, `_for_overwrite` | Transformar fatos em `Finding` |
| Resultado | `SchemaCheckResult` | Reunir avisos, erros e colunas novas |
| Entrada única | `check`, `schema_title` | O que a task chama |
| Exceção | `SchemaMismatchError` | Divergência de schema, fora do retry |

### Modelo

- `Column`: nome e tipo normalizados. `data_type=None` quando o tipo é desconhecido (caso do CSV).
- `TableSchema(columns, partition_keys)`: `columns` **não inclui** as chaves de partição. `partition_keys` é `()` sem partição e `None` quando não foi possível ler.

### Leitura dos schemas

| Classe | Método | O que faz |
|---|---|---|
| `UploadSchema` | `from_files(data_path, source_format)` | Lê o **primeiro** arquivo. CSV: header, com `pd.read_csv(nrows=0)`. Parquet: nomes e tipos, com pyarrow mapeado para BigQuery. As chaves de partição vêm das pastas `chave=valor`. Detecta chave de partição repetida no header e a registra em `UploadSchema.findings`. |
| `StagingSchema` | `from_table(table)` | Lê `table.schema`, tira as chaves de partição (que o BigQuery inclui no fim) e as lê de `partitionDefinition`. |

### Resultado: `SchemaCheckResult`

| Campo ou método | Descrição |
|---|---|
| `findings` | Lista de `Finding(severity, message)` |
| `added_columns` | Colunas novas, com nome normalizado e tipo do arquivo (em CSV, `STRING`) |
| `warnings`, `errors` | Mensagens por gravidade |
| `has_changes` | Se há algum aviso ou erro |
| `render(title)` | Texto para log e alerta, truncado |
| `raise_if_errors(title)` | Levanta `SchemaMismatchError` se houver erro |

### Entrada única

```python
check(staging, upload, source_format, overwrite)
```

Recebe um `StagingSchema`, um `UploadSchema`, o formato e se é `overwrite`. Devolve um `SchemaCheckResult`.

## Fluxo na task

```text
_upload_to_gcs
 ├─ append + staging existe
 │    ├─ _check_staging: StagingSchema.from_table + UploadSchema.from_files + check
 │    ├─ erros ──► raise SchemaMismatchError            (nada é alterado)
 │    ├─ _sync_staging_schema(result.added_columns)     # só se não houve erro
 │    └─ st.upload
 ├─ append + staging não existe → cria, sem comparação (primeiro upload)
 └─ overwrite                   → lê o schema antes de tb.delete / st.delete_table; só avisa
```

| Comportamento | Detalhe |
|---|---|
| Retorno de `_upload_to_gcs` | O `SchemaCheckResult`, ou `None` |
| Com avisos | A task `upload_to_gcs` faz `log(..., "warning")` e retorna `Completed(name="SchemaWarning", message=<texto>)`. O estado continua `COMPLETED`. |
| Retry | `SchemaMismatchError` fica fora do retry (`retry_condition_fn=_retry_unless_schema_mismatch`). Qualquer outra falha, inclusive da checagem, é retentada. |
| Falha da checagem | Fail-closed: o upload é abortado. |
| `_sync_staging_schema` | Recebe `added_columns` e altera a tabela no lugar com `update_table(["schema"])`. É só aditivo: nunca remove nem reordena, e não toca no prefixo do GCS. |
| `dump_mode` inválido | `ValueError`, sem efeito colateral |

## Alertas (Automation do Prefect e Discord)

Dois triggers, um para cada resultado:

1. **Warning:** evento de estado customizado.

   ```json
   {
     "type": "event",
     "match": { "prefect.resource.id": "prefect.task-run.*" },
     "expect": ["prefect.task-run.SchemaWarning"],
     "posture": "Reactive",
     "threshold": 1,
     "within": 0
   }
   ```

2. **Erro:** evento `prefect.task-run.Failed`, filtrado pela task `upload_to_gcs` e pelo erro `SchemaMismatchError`, para não alertar qualquer falha do upload.

Ação: *Send a notification* com um `CustomWebhookNotificationBlock` (`method=POST`, `json_data={"content": "{{body}}"}`, formato do Discord). Corpo em Jinja:

```text
{{ event.payload.validated_state.message }}
Flow run: {{ flow_run|ui_url }}
```

Para criar uma automação pela API REST do Prefect 3, veja o exemplo de [Flow Failed Webhook](../../infraestrutura/prefect/como-fazer/flow-failed-webhook.md).

## Testes

Para rodar: `uv run pytest pipelines/utils/tests/<arquivo>`.

### Unitários: `test_schema_check.py`

Sem rede. Usam schemas montados à mão e arquivos em `tmp_path`.

| Grupo | O que cobre |
|---|---|
| CSV (8) | sem mudança; coluna nova no fim (warning) e no meio (erro); faltando no fim e no meio (erro); ordem trocada; renomeação; nome normalizado |
| Parquet (7) | coluna nova em qualquer posição (warning, tipo em `added_columns`); faltando; ordem (warning); tipo incompatível; aliases; tipo desconhecido fora da checagem; formato inválido |
| Partição (6 casos) | layout igual; 4 variações de layout diferente (erro); chaves desconhecidas (warning) |
| Overwrite (3) | erro vira warning e `raise_if_errors` não levanta; sem mudança não avisa; nota do dbt |
| Resultado (2) | `raise_if_errors` com título; `render` truncado |
| Leitura (8) | `UploadSchema` em CSV (com e sem partição), Parquet (mapeamento de tipos), pasta vazia; `StagingSchema.from_table` com mock (3 casos) |
| Retry (2) | `SchemaMismatchError` não é retentado; outras falhas são |

### Mocks nas bordas: `test_upload_to_gcs.py`

Só `bd.Table`, `bd.Storage` e o client do BigQuery são falsos (`create_autospec`), anexados a um mock pai que registra a **ordem** das chamadas. Arquivos, schemas e regras são reais.

| # | Garantia | Origem |
|---|---|---|
| 1 | `append` com erro de schema levanta `SchemaMismatchError` antes de `update_table`, `st.upload` e qualquer deleção | [ADR-0007](../adr/0007-erro-de-schema-fora-do-retry.md) |
| 2 | `append` com coluna nova: `update_table` roda antes de `st.upload`, com as colunas e tipos de `added_columns` | Fluxo na task |
| 3 | `append` sem staging: cria a tabela e não chama `check` | Fluxo na task |
| 4 | `overwrite`: o schema é lido antes de `st.delete_table` e `tb.delete`; divergência não bloqueia | [ADR-0006](../adr/0006-overwrite-so-avisa.md) |
| 5 | Falha da checagem (erro de API) propaga e não é `SchemaMismatchError` | [ADR-0008](../adr/0008-falha-da-checagem-fail-closed.md) |
| 6 | Staging some entre `table_exists` e `get_table`: `RuntimeError` | Fluxo na task |
| 7 | `dump_mode` inválido: `ValueError`, sem efeito colateral | Fluxo na task |

Também cobre partição (chaves ignoradas na comparação; layout diferente é erro), Parquet (tipo da coluna nova; tipo incompatível), `_sync_staging_schema` chamado sem `incoming` (acrescenta só as colunas novas do header, e exige amostra e formato), a tradução do resultado na task (`SchemaWarning` ou `None`) e o **retry com o engine do Prefect** (`prefect_test_harness`): `SchemaMismatchError` roda 1 vez, outras falhas rodam 4.

Não há teste automatizado contra o BigQuery: o client é sempre mockado. Os comportamentos do BigQuery em que as regras se apoiam (formato de `partitionDefinition`, recusa a ler Parquet tipado numa staging `STRING`) foram verificados em staging reais e em dev.

## Ver também

- [Explicação: checagem de schema no upload](../explicacao/checagem-de-schema-no-upload.md): por que as regras são assim.
- [ADRs de Pipelines](../adr/index.md): as nove decisões que moldaram as regras.
