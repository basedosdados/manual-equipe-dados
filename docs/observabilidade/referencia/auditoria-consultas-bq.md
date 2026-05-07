---
tipo: referencia
titulo: Auditoria de consultas no BigQuery (Cloud Audit Logs)
origem: notas internas (example-docs-observabilidade.md)
---
# Auditoria de consultas no BigQuery (Cloud Audit Logs)

## Resumo

| Campo | Valor |
|---|---|
| O que é | Eventos de interação com o BigQuery (consultas, tentativas de acesso) registrados pelo Cloud Audit Logs |
| Fonte | `basedosdados.logs.cloudaudit_googleapis_com_data_access` |
| Exportação | Configurada no projeto `basedosdados`, com export horário para o BigQuery |

## Limitações

- Quando o conjunto tem política de permissionamento de linhas e/ou é aberto (`allAuthenticatedUsers`/`allUsers`), os logs **não coletam identificação do usuário**. Email, IP e query executada não são registrados no projeto `basedosdados` — apenas no projeto do usuário que executou a consulta. Referência: [Data Access Audit Logs](https://cloud.google.com/logging/docs/audit/configure-data-access).
- Logs registram eventos de infraestrutura, não apenas consultas — é necessário filtrar.

## Funil de filtragem

Para gerar métricas confiáveis, os logs brutos passam por três etapas.

### Etapa 1 — Filtragem de evento (o que aconteceu?)

- **Variável**: `protopayload_auditlog.methodName`
- **Filtro**: `'jobservice.insert' OR 'jobservice.query'`
- **Por quê**: garante que o processamento ocorreu e gerou estatísticas de uso (`jobservice.insert` registra a tentativa de envio; `jobcompleted` indica execução real).

### Etapa 2 — Expansão de estrutura (achatamento)

- **Variável**: `protopayload_auditlog.authorizationInfo`
- **Operação**: `UNNEST()`
- **Por quê**: este campo é um `ARRAY` — uma única query pode conter múltiplos objetos de autorização (ex.: verificar acesso à tabela E à row policy). O `UNNEST` transforma em linhas individuais.

### Etapa 3 — Filtragem de escopo (o que foi acessado?)

- **Variável**: `auth.permission` (gerado pelo unnest)
- **Filtro**: `bigquery.tables.getData` OU `bigquery.rowAccessPolicies.getFilteredData`
- **Por quê**: descarta eventos administrativos (listar tabelas, ver metadados) e foca em leitura de dados.

## Tabelas sem permissionamento

Tabelas públicas ou sem RLS têm comportamento linear no log:

- O `resource` termina diretamente no nome da tabela.
- `auth.permission = bigquery.tables.getData`.
- Gera **1 evento** de autorização por tabela consultada.

## Tabelas com permissionamento (BD Pro)

Tabelas com Row Access Policies (ex.: conjunto `br_bd_pro`) geram **múltiplos eventos** de verificação para uma única consulta.

- O `resource` possui sufixos como `/rowAccessPolicies/bdpro_filter` ou `/rowAccessPolicies/allusers_filter`.
- Gera **3 ou mais eventos** por tabela (verificação da tabela base + verificação de cada policy).

### Lógica do campo `granted`

| Valor | Significado |
|---|---|
| `true` | Acesso concedido por essa policy específica |
| `false` | Acesso negado por essa policy específica (mas pode ter sido concedido por outra) |
| `NULL` | Policy avaliada, mas não foi a que concedeu o acesso (usuário entrou por outra rota) |

### Tabela de interpretação

| Tipo de usuário | `permission` | Recurso (policy) | `granted` | Interpretação |
|---|---|---|---|---|
| BD Pro | `bigquery.rowAccessPolicies.getFilteredData` | `bdpro_filter` | `TRUE` | Acesso de usuário BD Pro |
| Público | `bigquery.rowAccessPolicies.getFilteredData` | `allusers_filter` | `TRUE` | Acesso público |
| Público | `bigquery.tables.getData` | `NULL` | `TRUE` | Acesso público |

## Exemplo — SQL para gerar ambas as métricas

```sql
WITH t2 AS (
  SELECT
    EXTRACT(year FROM timestamp) AS ano,
    EXTRACT(month FROM timestamp) AS mes,
    logName,
    protopayload_auditlog.methodName AS name,
    auth.permission,
    protopayload_auditlog.status.code,
    protopayload_auditlog.status.message,
    insertId,
    auth.granted,
    SPLIT(auth.resource, '/')[SAFE_OFFSET(3)] AS id_conjunto,
    SPLIT(auth.resource, '/')[SAFE_OFFSET(5)] AS id_tabela,
    SPLIT(auth.resource, '/')[SAFE_OFFSET(7)] AS politica_permissionamento,
    CASE
      WHEN REGEXP_CONTAINS(LOWER(protopayload_auditlog.requestMetadata.callerSuppliedUserAgent), 'python') THEN 'python'
      WHEN REGEXP_CONTAINS(LOWER(protopayload_auditlog.requestMetadata.callerSuppliedUserAgent), 'rstudio|bigrquery') THEN 'R'
      ELSE 'bigquery'
    END AS tipo_conexao
  FROM `basedosdados.logs.cloudaudit_googleapis_com_data_access`,
  UNNEST(protopayload_auditlog.authorizationInfo) AS auth
  WHERE (protopayload_auditlog.methodName = 'jobservice.insert'
         OR protopayload_auditlog.methodName = 'jobservice.query')
    AND EXTRACT(year FROM timestamp) = 2025
    AND EXTRACT(month FROM timestamp) = 1
    AND auth.permission IN (
      'bigquery.tables.getData',
      'bigquery.rowAccessPolicies.getFilteredData'
    )
)
SELECT
  ano,
  mes,
  id_conjunto,
  COUNTIF(politica_permissionamento IS NOT NULL AND granted = TRUE) AS quantidade_acessos_bd_profilter,
  COUNTIF(politica_permissionamento IS NULL) AS quantidade_acessos_allusers,
  COUNTIF(politica_permissionamento IS NULL AND permission = 'bigquery.tables.getData') AS quantidadade_total_acessos
FROM t2
GROUP BY ALL
ORDER BY 1, 2 DESC;
```

## Ver também

- [O dashboard Central da Equipe Dados](../explicacao/dashboard-central-equipe-dados.md)
