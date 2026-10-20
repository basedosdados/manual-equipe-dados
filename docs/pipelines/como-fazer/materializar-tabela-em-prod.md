---
tipo: como-fazer
titulo: Como materializar uma tabela em prod com o table-approve
---
# Como materializar uma tabela em prod com o table-approve

## Pré-requisitos

- PR no repositório `pipelines` que altera ao menos um arquivo `models/<dataset>/<modelo>.sql`.
- PR aprovado na revisão.
- `dbt run` e `dbt test` executados com sucesso em dev.
- Colunas novas registradas no backend de prod.

## Passos

1. Confira se a staging de dev tem todos os períodos da tabela. A consulta abaixo deve retornar todos os anos, com um arquivo em cada.

    ```sql
    select ano, count(*) as linhas, count(distinct _FILE_NAME) as arquivos
    from `basedosdados-dev.<dataset>_staging.<tabela>`
    group by ano
    order by ano
    ```

2. Confira se a tabela de staging de dev tem todas as colunas usadas no modelo.

    ```sql
    select column_name
    from `basedosdados-dev.<dataset>_staging`.INFORMATION_SCHEMA.COLUMNS
    where table_name = '<tabela>'
    ```

3. Verifique se algum modelo do PR depende de outro modelo novo do mesmo PR. Se depender, não aplique o label e, após o merge, execute o workflow `promote-staging` com a opção `all_models`.

4. Aplique o label `table-approve` no PR.

    ```bash
    gh pr edit <numero> -R basedosdados/pipelines --add-label table-approve
    ```

5. Faça o merge do PR.

6. Acompanhe a execução do workflow.

    ```bash
    gh run list -R basedosdados/pipelines --workflow table-approve.yaml --limit 3
    gh run watch <run_id> -R basedosdados/pipelines
    ```

## Verificação

O log do workflow deve conter as linhas abaixo, as duas últimas uma vez para cada tabela:

```text
table-approve label add: true
TABLE CREATED: basedosdados-staging.<dataset>_staging.<tabela>
Flow run <id> (<dataset>.<tabela>) finished successfully.
```

Em seguida, compare a contagem de linhas por ano entre prod e dev. Os valores devem ser iguais.

```sql
select ano, count(*) as linhas
from `basedosdados.<dataset>.<tabela>`
group by ano
order by ano
```

## Problemas comuns

- **O job terminou com sucesso, mas nada mudou em prod.** O label não estava no PR no momento do merge. Aplique o label e reexecute o workflow com `gh run rerun <run_id> -R basedosdados/pipelines`.
- **O log mostra `TABLE was NOT created`.** A staging de dev da tabela está vazia. Carregue a staging de dev e reexecute o workflow.
- **O dbt falhou com `Unrecognized name: <coluna>`.** A coluna não existe em todos os períodos da staging de dev. Regrave todos os períodos e reexecute o workflow.

## Ver também

- [Table-approve](../explicacao/table-approve.md), sobre o funcionamento e as limitações do processo
- [Abrir um PR](abrir-pr.md), sobre as labels de PR
