---
tipo: adr
titulo: ADR-0002 — Não apagar as regras de acesso no pre_hook dos modelos dbt
status: aceito
data: 2026-11-17
pr: https://github.com/basedosdados/pipelines/pull/2298
---
# ADR-0002 — Não apagar as regras de acesso no pre_hook dos modelos dbt

## Contexto

Tabelas `part_bdpro` têm duas regras de acesso (Row Access Policies) no BigQuery: `bdpro_filter`,
que libera tudo para os grupos `bd-pro` e `sudo`, e `allusers_filter`, que libera para todos só a
faixa gratuita. As duas são criadas por `update_row_access_policy`
(`pipelines/utils/metadata/utils.py`), chamada pelo `register_table_materialization_task` depois do
dbt, no `build_and_promote` e no `table-approve`.

Em novembro de 2026, 68 modelos do repositório `pipelines` (45 incrementais e 21 `table`) tinham um
`pre_hook` com `DROP ALL ROW ACCESS POLICIES`. A justificativa registrada nos commits era que o
BigQuery bloqueia DML em tabela com regra de acesso, e o `insert_overwrite` do modelo incremental é
DML.

Um teste em `basedosdados-dev`, em 17/11/2026, mostrou que isso só vale para quem não tem acesso
total à tabela:

- com acesso total (coberto por uma regra com filtro `TRUE`), `INSERT`, `DELETE` e o `MERGE` do
  `insert_overwrite` funcionaram, e as regras continuaram na tabela;
- só com a regra `allUsers`, os três falharam com `User does not have full access for table ...
  due to row access policies`;
- `CREATE OR REPLACE TABLE`, usado pelos modelos `table`, apaga as regras sozinho.

A service account que roda o dbt em prod está no grupo de uma regra com filtro `TRUE`.

## Decisão

Vamos parar de incluir o `pre_hook` que apaga as regras de acesso nos modelos dbt. Modelos novos não
o incluem, e os modelos existentes o perdem quando forem alterados. A revisão de PR confere isso
([checklist](../../processos/como-fazer/revisar-pr.md)).

## Consequências

### Positivas
- Em modelos incrementais, a faixa paga não fica aberta entre o `dbt run` e o registro de metadados.
- Um `dbt run` avulso, pelo deployment "Executa DBT model", deixa de apagar as regras de uma tabela
  incremental sem recriá-las.
- Menos configuração repetida nos modelos.

### Negativas
- O funcionamento depende de a service account do dbt continuar no grupo de uma regra com filtro
  `TRUE`. Se ela sair do grupo, os modelos incrementais `part_bdpro` falham na segunda carga com o
  erro de acesso total.

### Neutras
- Em modelos `table`, o `CREATE OR REPLACE` continua apagando as regras a cada `dbt run`. Elas só
  voltam no registro de metadados, como antes.
- Os modelos antigos com o `pre_hook` continuam funcionando até serem alterados.

## Alternativas consideradas

- **Manter o `pre_hook` em todos os modelos** — descartado porque é desnecessário para quem tem
  acesso total e abre a faixa paga a cada execução até o registro de metadados.
- **Manter o `pre_hook` só nos modelos `table`** — descartado porque o `CREATE OR REPLACE` já apaga
  as regras, então o `pre_hook` não muda nada.
- **Remover o `pre_hook` de todos os 68 modelos de uma vez** — descartado por ora: cada remoção
  precisa ser testada em dev, e a mudança entra junto com as alterações de cada conjunto.

## Status

Aceito na revisão do [PR #2298](https://github.com/basedosdados/pipelines/pull/2298)
(`br_tse_filiacao_partidaria`).
