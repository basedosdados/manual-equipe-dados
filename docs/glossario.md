---
tipo: referencia
titulo: Glossário
---
# Glossário

Termos usados de forma recorrente em toda a documentação da equipe dados.

> Adicione termos aqui em ordem alfabética. Mantenha definições curtas (1–3 frases) e linke para a página de referência ou explicação completa quando houver.

## A

## B

**BD Pro** — assinatura paga da Base dos Dados que dá acesso a tabelas com Row Access Policies específicas (`bdpro_filter`).

## C

## D

**Datalake House** — arquitetura híbrida que combina um *data lake* (arquivos brutos/tratados em object storage) com um *data warehouse* (engine SQL com modelos materializados), unidos por tabelas externas. Na BD: os buckets GCS (`basedosdados-dev`, `basedosdados-staging`) são o lake; os projetos BigQuery (`basedosdados-dev`, `basedosdados-staging`, `basedosdados`) são o warehouse; tabelas `<dataset>_staging` são externas e leem direto do GCS, enquanto `<dataset>` são materializadas pelo dbt. Ver [Fluxo de dados e infraestrutura](onboarding/fluxo-de-dados-e-infraestrutura.md).

## E

## F

## G

## H

## I

## J

## L

## M

**Manual de estilo** — conjunto de regras canônicas da BD para nomeação, tipagem, formato e padronização de datasets, tabelas, colunas e diretórios. Mantido publicamente em <https://basedosdados.org/docs/style_data> e aplicado tanto no preenchimento de metadados (backend GraphQL) quanto na estrutura dos modelos dbt. Ver [Manual de estilo — explicação](governanca/metadados/explicacao-manual-de-estilo.md).

## N

## O

## P

**Pipeline** — tabela cuja atualização é automatizada via Prefect.

**Pipeline semi-automatizada** — tabela com código de ELT/ETL no repositório, mas sem schedule do Prefect; depende de execução humana.

## Q

## R

## S

## T

## U

## V

## Z

**Zona de dev** — ambiente de desenvolvimento e validação na GCP. Componentes:

- **GCS** — bucket `basedosdados-dev`.
- **BigQuery** — projeto `basedosdados-dev`, que contém **as duas camadas no mesmo projeto**: `<dataset>_staging` (tabelas externas sobre o bucket `basedosdados-dev`) e `<dataset>` (modelos materializados pelo dbt com `target=dev`).

Usada pela equipe para validar dados e modelos antes da promoção para prod. Ver [Fluxo de dados e infraestrutura](onboarding/fluxo-de-dados-e-infraestrutura.md).

**Zona de prod** — ambiente de produção na GCP, dividido em **dois projetos BigQuery distintos**:

- **GCS** — bucket `basedosdados-staging` (espelho dos datasets com sufixo _staging `basedosdados-dev` após aprovação).
- **BigQuery `basedosdados-staging`** — camada de tabelas externas (`<dataset>_staging`) sobre o bucket `basedosdados-staging`. **Não é exposto ao público.**
- **BigQuery `basedosdados`** — modelos materializados pelo dbt (`target=prod`) a partir de `basedosdados-staging`. É o projeto consultado pelos usuários no site e no pacote Python.

A separação `basedosdados-staging` ↔ `basedosdados` é o que diferencia prod de dev: em prod, externas e materializadas vivem em **projetos diferentes**; em dev, vivem no mesmo projeto.
