---
tipo: runbook
titulo: Runbook — table-approve falhou
trigger: O workflow table-approve falhou após o merge, ou terminou com sucesso sem atualizar a tabela em prod
---
# Runbook — table-approve falhou

## Sintoma

O workflow `table-approve` do repositório `pipelines` aparece com falha na aba Actions após o merge de um PR. Também se aplica quando o workflow termina com sucesso, mas a tabela em `basedosdados.<dataset>.<tabela>` não foi atualizada ou ficou com dados incorretos.

## Impacto

Quando o workflow falha, a tabela em prod continua na versão anterior. Quando a falha ocorre na materialização, a staging de prod de todas as tabelas do PR já foi substituída pela staging de dev, e a versão anterior fica apenas em `gs://basedosdados-backup/staging/<dataset>/<tabela>/`. Esse backup é sobrescrito a cada nova execução do workflow.

## Diagnóstico

1. Localize a execução do workflow referente ao merge e leia o log dos passos que falharam.

    ```bash
    gh run list -R basedosdados/pipelines --workflow table-approve.yaml --limit 5
    gh run view <run_id> -R basedosdados/pipelines --log-failed
    ```

    Se a execução terminou com sucesso, use `--log` no lugar de `--log-failed`.

2. Procure no log a mensagem que identifica o caso. Quando a falha ocorre na materialização, o log traz os erros do Prefect e o link da execução, onde está o log completo do dbt.

    | Mensagem no log | Caso |
    |---|---|
    | `table-approve label add: false` | A |
    | `No pull request found for this commit.` | B |
    | `TABLE was NOT created` | C |
    | `Unrecognized name: <coluna>` | D |
    | `Not found: Table basedosdados-staging:<dataset>_staging.<tabela>` | E |
    | `Not found: Table basedosdados:<dataset>.<tabela>` | F |
    | `DATA ERROR ON staging.<dataset>.<tabela>` | G |
    | Execução com sucesso, mas períodos ausentes ou valores em colunas trocadas | H |

## Resolução

### Caso A: label ausente no merge

O PR não tinha o label `table-approve` quando o workflow foi executado, e por isso nenhuma etapa rodou. Aplique o label no PR já mergeado e reexecute o workflow. A nova execução lê os labels novamente.

```bash
gh pr edit <numero> -R basedosdados/pipelines --add-label table-approve
gh run rerun <run_id> -R basedosdados/pipelines
```

### Caso B: push direto na main

O commit não veio de um PR, então não há tabelas a materializar. Nenhuma ação é necessária.

### Caso C: staging de dev vazia

Não havia arquivos em `gs://basedosdados-dev/staging/<dataset>/<tabela>/`, e a staging de prod não foi alterada. Carregue a staging de dev com todos os períodos e reexecute o workflow com `gh run rerun <run_id> -R basedosdados/pipelines`.

### Caso D: coluna ausente na staging

O modelo lê uma coluna que não existe na tabela externa de staging. O esquema dessa tabela vem do primeiro arquivo da staging de dev, normalmente o período mais antigo. Regrave todos os períodos da staging de dev com o esquema novo, confirme as colunas com a consulta abaixo e reexecute o workflow.

```sql
select column_name
from `basedosdados-dev.<dataset>_staging`.INFORMATION_SCHEMA.COLUMNS
where table_name = '<tabela>'
```

### Caso E: staging com nome diferente da tabela

O modelo lê uma tabela de staging cujo nome não corresponde a nenhuma tabela publicada, e o table-approve copia apenas as stagings com o mesmo nome dos modelos alterados. Execute o workflow manual `promote-staging` a partir da `main`, informando em `staging_tables` as stagings que faltam e em `models` as tabelas a materializar.

### Caso F: dependência de outro modelo novo

O modelo referencia outro modelo novo do mesmo PR que ainda não foi materializado em prod. Execute o workflow `promote-staging` com a opção `all_models`, que materializa o dataset inteiro em ordem de dependências.

### Caso G: erro na cópia da staging

A cópia entre buckets falhou, e a staging de prod pode ter ficado incompleta. O workflow não é interrompido nesse ponto: a tabela é registrada como `TABLE was NOT created` e a materialização ocorre sobre a staging de prod no estado em que ficou. Reexecute o workflow para refazer a cópia a partir da staging de dev. A versão anterior da staging de prod está em `gs://basedosdados-backup/staging/<dataset>/<tabela>/`, e restaurá-la exige credenciais de prod.

### Caso H: dados incorretos com execução bem-sucedida

Períodos ausentes indicam que a staging de dev não continha todos os períodos, já que a staging de prod é substituída integralmente. Valores em colunas trocadas indicam arquivos CSV com ordem de colunas diferente entre períodos, lidos por posição. Corrija a staging de dev e reexecute o workflow.

Antes de reexecutar, verifique se a versão anterior ainda é necessária: a nova execução sobrescreve o backup com a staging de prod atual, que já contém os dados incorretos.

## Após a resolução

- Confirme o resultado conforme a seção Verificação de [Como materializar uma tabela em prod](../como-fazer/materializar-tabela-em-prod.md).
- Registre no PR a causa da falha e a ação tomada.

## Ver também

- [Table-approve](../explicacao/table-approve.md), sobre o funcionamento do processo
- [Como materializar uma tabela em prod](../como-fazer/materializar-tabela-em-prod.md), sobre os pré-requisitos e os passos
- [Referência do table-approve](../referencia/table-approve.md), com as mensagens de log e os inputs do `promote-staging`
