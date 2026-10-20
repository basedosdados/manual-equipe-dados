---
tipo: referencia
titulo: Table-approve
---
# Table-approve

## Resumo

| Campo | Valor |
|---|---|
| O que é | Workflow do GitHub Actions que materializa em prod as tabelas alteradas em um PR do repositório `pipelines` |
| Gatilho | Push na `main`, com o label `table-approve` no PR de origem |
| Workflow | `.github/workflows/table-approve.yaml` |
| Script | `.github/workflows/scripts/prefect_run_dbt.py` |
| Flow do Prefect | `BD template: Executa DBT model/run_dbt_model_flow`, em `pipelines/utils/execute_dbt_model/flows.py` |
| Exportação de CSV | `download_data_to_gcs`, em `pipelines/utils/tasks.py` |

## Seleção das tabelas

| Regra | Valor |
|---|---|
| Arquivos considerados | Arquivos do PR com status diferente de `removed`, listados com paginação de 100 por página |
| Caminho aceito | `models/<dataset>/<modelo>.sql`, com exatamente três níveis |
| Caminhos ignorados | Subdiretórios de `models/<dataset>/`, `tests-dbt/`, `macros/`, `analysis/` e qualquer arquivo que não seja `.sql` |
| Dataset | Nome do diretório pai do `.sql` |
| Tabela | Nome do arquivo sem extensão e sem o prefixo `<dataset>__` |
| `dbt_alias` | `True` quando o prefixo `<dataset>__` está presente; `False` caso contrário |
| Arquivo inexistente no checkout | Ignorado |

## Caminhos

| Recurso | Caminho |
|---|---|
| Staging de dev (origem) | `gs://basedosdados-dev/staging/<dataset>/<tabela>/` |
| Staging de prod (destino) | `gs://basedosdados/staging/<dataset>/<tabela>/` |
| Backup | `gs://basedosdados-backup/staging/<dataset>/<tabela>/`, uma única versão por tabela |
| Tabela externa de dev | `basedosdados-dev.<dataset>_staging.<tabela>` |
| Tabela externa de prod | `basedosdados-staging.<dataset>_staging.<tabela>` |
| Tabela final | `basedosdados.<dataset>.<tabela>` |
| Projeto de cobrança do GCS | `basedosdados` |

O macro `set_datalake_project` resolve a staging para `basedosdados-dev` no
target `dev` e para `basedosdados-staging` no target `prod`.

## Esquema da tabela externa de prod

| Formato do primeiro arquivo | Origem do esquema |
|---|---|
| CSV | `SELECT * FROM basedosdados-dev.<dataset>_staging.<tabela> LIMIT 1`, sem as colunas de partição Hive |
| Parquet | O próprio arquivo |

O primeiro arquivo é o primeiro `.csv` ou `.parquet` listado em
`gs://basedosdados-dev/staging/<dataset>/<tabela>/`, em ordem alfabética de
caminho. Todas as colunas são criadas como STRING. CSV é lido por posição, e
Parquet, por nome.

## Parâmetros do flow

Valores enviados pelo script a `run_dbt_model_flow` para cada tabela.

| Parâmetro | Valor no table-approve | Padrão do flow |
|---|---|---|
| `dataset_id` | Dataset do `.sql` | — |
| `table_id` | Tabela do `.sql` | `None` (dataset inteiro) |
| `dbt_command` | `run` | `run` |
| `dbt_alias` | Conforme a seleção | `True` |
| `target` | `prod` | `dev` |
| `download_csv_file` | `True` | `True` |
| `flags` | Não enviado | `None` |
| `dbt_vars` | Não enviado | `None` |

O comando executado é
`dbt run --select models/<dataset>/<modelo>.sql --target prod`. Os flows são
disparados um de cada vez, na ordem dos arquivos do PR.

## Exportação de CSV

Executada após o `dbt run`, conforme o tamanho da tabela em
`basedosdados.<dataset>.<tabela>`.

| Tamanho | Arquivos gerados |
|---|---|
| Até 100 MB | `<tabela>.csv.gz` (aberto) e, se houver a row access policy `bdpro_filter`, `<tabela>_bdpro.csv.gz` |
| Até 100 MB, com cobertura BD Pro no backend e sem `bdpro_filter` | Apenas `<tabela>_bdpro.csv.gz` |
| De 100 MB a 1 GB | Apenas `<tabela>_bdpro.csv.gz` |
| Acima de 1 GB | Nenhum |

Os prefixos de destino vêm do secret `url_download_data`
(`URL_DOWNLOAD_OPEN` e `URL_DOWNLOAD_CLOSED`), seguidos de
`<dataset>/<tabela>/`.

## Mensagens de log

| Mensagem | Significado |
|---|---|
| `table-approve label add: true` / `false` | Resultado da verificação do label |
| `No pull request found for this commit.` | O commit não está associado a um PR; o job falha |
| `Changed files: <lista>` | Arquivos do PR enviados ao script |
| `No objects found on the source bucket basedosdados-dev.staging/<dataset>/<tabela>/` | Staging de dev vazia |
| `DATA ERROR ON staging.<dataset>.<tabela>` | Exceção durante a cópia entre buckets |
| `TABLE CREATED: basedosdados-staging.<dataset>_staging.<tabela>` | Staging substituída e tabela externa recriada |
| `TABLE was NOT created: basedosdados-staging.<dataset>_staging.<tabela>` | Staging não substituída |
| `Launching materialization flow for <dataset>.<tabela> (alias=<bool>)...` | Início do flow da tabela |
| `Flow run <id> (<dataset>.<tabela>) finished successfully.` | Flow concluído |
| `Flow run <id> for <dataset>.<tabela> finished with state "<estado>"` | Flow falhou; o job é encerrado |

## Workflow promote-staging

Workflow manual (`.github/workflows/promote-staging.yaml`), executado a partir
da `main`. Usa a mesma função de cópia e recriação da staging do
table-approve.

| Input | Tipo | Padrão | Descrição |
|---|---|---|---|
| `dataset_id` | string | — (obrigatório) | Dataset a promover |
| `staging_tables` | string | vazio | Tabelas de staging a copiar de dev para prod, separadas por vírgula |
| `models` | string | vazio | Tabelas a materializar, uma execução do flow por tabela, com exportação de CSV |
| `all_models` | boolean | `false` | Materializa o dataset inteiro em uma única execução, em ordem de dependências, sem exportação de CSV |
| `force` | boolean | `false` | Copia a staging mesmo quando a quantidade de arquivos no destino já é igual à da origem |
| `dry_run` | boolean | `false` | Apenas informa o que seria feito |

```bash
gh workflow run promote-staging.yaml -R basedosdados/pipelines --ref main \
  -f dataset_id=<dataset> -f all_models=true
```

## Ver também

- [Table-approve](../explicacao/table-approve.md), sobre o funcionamento e as limitações do processo
- [Como materializar uma tabela em prod](../como-fazer/materializar-tabela-em-prod.md)
- [Runbook — table-approve falhou](../runbooks/table-approve-falhou.md)
