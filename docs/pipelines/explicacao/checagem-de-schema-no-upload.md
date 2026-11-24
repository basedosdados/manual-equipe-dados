---
tipo: explicacao
titulo: Checagem de schema no upload para o GCS
---
# Checagem de schema no upload para o GCS

Esta página explica por que a task `upload_to_gcs` compara o arquivo que vai subir com a tabela de staging, por que as regras são as que são e o que a checagem não cobre. Para as regras em formato de consulta, veja a [referência de `schema_check.py`](../referencia/schema-check.md). Para agir diante de um alerta, veja o [runbook](../runbooks/erro-de-schema-no-upload.md).

> **Estado:** implementado e mergeado no PR [#2228](https://github.com/basedosdados/pipelines/pull/2228) (24/11/2026), com testes unitários e com mocks. 

## Contexto

Todo dataset sobe seus arquivos para o GCS com `upload_to_gcs`, e o BigQuery os lê por uma **tabela externa de staging** (`<dataset>_staging`). O dbt materializa a tabela final a partir dessa staging.

O problema é que a staging é criada uma vez e **seu schema fica congelado** em `dump_mode="append"`. Quando a fonte muda, o arquivo novo e a staging deixam de combinar, e o erro costumava aparecer só depois, no dbt (por exemplo, `Unrecognized name`). Pior: algumas mudanças nem geram erro. Uma coluna trocada de posição num CSV põe os dados na coluna errada sem nenhum aviso.

A checagem de schema compara as colunas do arquivo com as da staging **antes** de subir e produz um de dois resultados:

- **warning:** o upload segue e a equipe é avisada;
- **erro:** o upload é abortado antes de alterar a staging ou o bucket.

## A ideia central

### Dois formatos, duas lógicas

O comportamento depende do formato porque a lib `basedosdados` monta a tabela externa de formas diferentes:

| | CSV | Parquet |
|---|---|---|
| Schema da staging | explícito, com `autodetect=False` | derivado do `header.parquet`, subido antes pelo `dump_header` |
| Como o BigQuery casa as colunas | **por posição** | **por nome** |
| Tipo das colunas | tudo `STRING` | tudo `STRING` (o header usa `.astype(str)`) |

Daí vêm as diferenças de regra. Em CSV, ordem trocada e coluna faltando no meio corrompem os dados sem erro, então são erro. Em Parquet, a ordem não importa, então é só um aviso.

O tipo inferido pela lib é sempre `STRING`, em CSV e em Parquet, então não serve para checar tipo. A checagem de tipo existe só em Parquet, e compara o tipo **real** do arquivo com o da staging. Por convenção do pipeline, os arquivos de dados também são all-`STRING` (o dbt faz o `safe_cast`). O BigQuery não converte tipos na leitura, então um arquivo `INT64` numa staging `STRING` falha na leitura.

### A staging atual é a referência

O **schema final esperado** é a staging atual mais as colunas novas, no fim. Em CSV, o header do arquivo precisa ser um prefixo desse schema final, posição por posição. Só uma coluna nova **no fim** vira warning; no meio, desalinha tudo o que vem depois e vira erro.

Quando há coluna nova, a staging é alterada no lugar, só de forma aditiva (nunca remove nem reordena), logo depois da checagem e antes do upload.

### `overwrite` só avisa

Em `dump_mode="overwrite"`, a staging e o prefixo do GCS são recriados do zero. Não há histórico para desalinhar, então a checagem não bloqueia. Mesmo assim ela avisa, porque uma coluna renomeada ou removida pode quebrar o dbt a jusante. O schema antigo é lido antes de qualquer deleção.

### A checagem é fail-closed

Se a checagem não consegue rodar (por exemplo, erro de API do BigQuery), o upload é abortado. Isso evita subir dado sem validação, ao custo de uma instabilidade do BigQuery derrubar flows. Como essa falha é retentada normalmente, uma instabilidade curta não chega a derrubar o flow.

Já a divergência de schema (`SchemaMismatchError`) **não é retentada**: repetir o upload não a corrige, então o erro e o alerta chegam na primeira tentativa.

### Tabelas particionadas

Em tabelas particionadas (Hive), as chaves de partição existem só como nome de pasta (`ano=2024/mes=01/`) e o BigQuery as inclui no fim de `table.schema`. A checagem as tira da comparação de colunas e verifica à parte se o **layout** (chaves e ordem) é o mesmo da staging. Valores novos de partição (`ano=2025`) não são mudança de schema.

## Trade-offs e alternativas consideradas

Cada decisão está registrada em uma ADR própria:

| Decisão | ADR |
|---|---|
| Módulo novo em vez de estender o validador existente | [ADR-0001](../adr/0001-regras-de-schema-em-modulo-novo.md) |
| Ordem das colunas: erro em CSV, warning em Parquet | [ADR-0002](../adr/0002-ordem-das-colunas.md) |
| Tipo: só Parquet, igualdade estrita | [ADR-0003](../adr/0003-tipo-das-colunas.md) |
| Coluna a mais: acrescentar à staging e avisar | [ADR-0004](../adr/0004-coluna-a-mais.md) |
| Coluna faltando: erro no `append` | [ADR-0005](../adr/0005-coluna-faltando.md) |
| `overwrite`: só avisa | [ADR-0006](../adr/0006-overwrite-so-avisa.md) |
| `SchemaMismatchError` fora do retry | [ADR-0007](../adr/0007-erro-de-schema-fora-do-retry.md) |
| Falha da própria checagem: aborta (fail-closed) | [ADR-0008](../adr/0008-falha-da-checagem-fail-closed.md) |
| Tabelas particionadas | [ADR-0009](../adr/0009-tabelas-particionadas.md) |

## Limitações e pontos em aberto

- **Primeiro upload sem checagem.** Sem staging, não há com o que comparar. Um Parquet tipado no primeiro upload passa, e a leitura falha depois.
- **Só o primeiro arquivo é lido.** O header e os tipos vêm do primeiro arquivo do `data_path`. As chaves de partição vêm do caminho, que é barato e permitiria exigir um único conjunto de tuplas. Em Parquet, `pq.read_schema` lê só o rodapé e também é barato. Em CSV, ler todos os arquivos exigiria abrir cada um, um custo a decidir contra o risco.
- **Fallback por blob para as chaves da staging.** Se `partitionDefinition` faltar mas `hive_partitioning` existir, as chaves ficam desconhecidas e o layout não é verificado, com um warning explícito. Uma alternativa seria ler um blob em `source_uri_prefix` (`list_blobs(max_results=1)`).
- **Chave de partição repetida no header.** Se uma chave Hive aparecer também dentro do arquivo, a tabela teria duas colunas com o mesmo nome. A checagem detecta isso: é **erro** em CSV (posicional: a coluna seria acrescentada à staging, em conflito com a chave) e **warning** em Parquet (casa por nome). No `overwrite`, o erro vira warning. **Em aberto:** confirmar na staging real se o BigQuery aceita a duplicata em Parquet. Se recusar, o warning vira erro. O caso conhecido é o `br_fnde_fundeb`, que escreve `ano` dentro do Parquet e na pasta.


## Ver também

- [Referência de `schema_check.py`](../referencia/schema-check.md): regras, módulo, fluxo na task, alertas e testes.
- [Pipeline orientado a eventos](pipeline-orientado-a-eventos.md): onde o `upload_to_gcs` entra na cadeia de estágios.
- [ADRs de Pipelines](../adr/index.md)
