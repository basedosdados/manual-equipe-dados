---
tipo: referencia
titulo: Habilidades do engenheiro de dados
---
# Habilidades do engenheiro de dados

## Resumo

| Campo | Valor |
|---|---|
| O que é | Lista do que um engenheiro de dados precisa saber para operar bem a BD, e o que vem depois |
| Fonte | Coordenação da equipe dados |
| Atualização | Quando a infraestrutura ou os processos da equipe mudarem |
| Dono | Equipe dados |

## Definição

As habilidades estão organizadas pelos três passos da [trilha do engenheiro de dados](../explicacao/trilha-do-engenheiro-de-dados.md).
A coluna **Onde aprender** aponta a página do manual que cobre o assunto. "Sem página" indica um
assunto que o manual ainda não cobre.

## Passo 1: operar bem a BD

### O que a pessoa faz

| Habilidade | Onde aprender |
|---|---|
| Executar pipelines de acordo com os princípios estruturados | [Princípios de tratamento](../../governanca/tratamento-de-dados/explicacao/principios-de-tratamento.md), [Práticas de tratamento](../../governanca/tratamento-de-dados/referencia/praticas-de-tratamento.md) |
| Revisar PRs | [Como revisar um PR de pipeline](../como-fazer/revisar-pr.md) |
| Preencher os metadados, entender e corrigir problemas de preenchimento | [Subir metadados](../../governanca/metadados/como-fazer/subir-metadados.md), [Regras de metadados](../../governanca/qualidade/referencia/regras-metadados.md) |
| Aplicar os testes e resolvê-los de acordo com os nossos princípios | [Testes dbt obrigatórios](../../pipelines/referencia/testes-dbt-obrigatorios.md) |
| Trazer diagnósticos dos problemas com evidências | [Diagnóstico com evidência](diagnostico-com-evidencia.md) |

### Base de conhecimentos

#### Git

| Habilidade | Onde aprender |
|---|---|
| Entender os conceitos principais (commit, branch, merge, rebase, conflito) | Sem página |

#### Python

| Habilidade | Onde aprender |
|---|---|
| Conhecer o [Zen of Python](https://peps.python.org/pep-0020/) | Link externo |
| Ler um código Python e saber o que ele está fazendo | Sem página |
| Contornar ou economizar processamento em tabelas muito grandes | Sem página |

#### Requisições

| Habilidade | Onde aprender |
|---|---|
| Fazer requisições e extrair dados com XPath e seletores CSS | Sem página |
| Saber quando usar XPath ou seletor CSS e quando é possível evitar regex | Sem página |
| Fazer requisições assíncronas e com threads, e saber quando vale a pena | Sem página |
| Evitar bloqueio de IP | [Usar o proxy de IP brasileiro](../../pipelines/como-fazer/usar-proxy-ip-brasileiro.md) |

#### SQL e dbt

| Habilidade | Onde aprender |
|---|---|
| Economizar ao fazer queries | Sem página |
| Ler Jinja e escrever macros: loops, condicionais e variáveis | Sem página |

#### BigQuery

| Habilidade | Onde aprender |
|---|---|
| Saber quando particionar uma tabela | [BigQuery](../../infraestrutura/bigquery/index.md) |
| Entender e aplicar row access policies | Sem página |

#### Metadados

| Habilidade | Onde aprender |
|---|---|
| Saber quais tabelas existem no backend | Sem página |
| Saber como essas tabelas se conectam | Sem página |
| Conhecer as principais restrições do modelo | [Regras de metadados](../../governanca/qualidade/referencia/regras-metadados.md) |
| Entender o conceito de nível de observação | Sem página |

#### Processos de dados da BD

| Habilidade | Onde aprender |
|---|---|
| Saber o que o `table-approve` faz | [Table-approve](../../pipelines/explicacao/table-approve.md) |
| Conhecer os testes padrão e como são construídos | [Testes dbt obrigatórios](../../pipelines/referencia/testes-dbt-obrigatorios.md) |
| Entender o fluxo padrão das pipelines | [Pipeline orientado a eventos](../../pipelines/explicacao/pipeline-orientado-a-eventos.md), [Estruturar um flow de dataset](../../pipelines/como-fazer/estruturar-um-flow-de-dataset.md) |

#### Infraestrutura da BD

| Habilidade | Onde aprender |
|---|---|
| Conhecer os três projetos do GCP (`basedosdados-dev`, `basedosdados-staging`, `basedosdados`) | [Fluxo de dados e infraestrutura](../../onboarding/fluxo-de-dados-e-infraestrutura.md) |
| Entender bucket, tabela externa e tabela materializada | [Fluxo de dados e infraestrutura](../../onboarding/fluxo-de-dados-e-infraestrutura.md) |
| Saber como os dados entram em cada estágio | [Fluxo de dados e infraestrutura](../../onboarding/fluxo-de-dados-e-infraestrutura.md) |
| Saber os caminhos para levar dados de dev para prod: `table-approve` e pipelines | [Table-approve](../../pipelines/explicacao/table-approve.md), [Fluxo de dados e infraestrutura](../../onboarding/fluxo-de-dados-e-infraestrutura.md) |

#### Prefect 3

| Habilidade | Onde aprender |
|---|---|
| Entender os conceitos de task, util e flow | [Estruturar um flow de dataset](../../pipelines/como-fazer/estruturar-um-flow-de-dataset.md), [Arquitetura do Prefect 3](../../infraestrutura/prefect/explicacao/arquitetura-prefect-3.md) |

## Passos 2 e 3: outras infraestruturas e montar a nossa

| Habilidade | Passo | Onde aprender |
|---|---|---|
| Trabalhar em infraestruturas que não são da BD | 2 | Sem página |
| Usar outros orquestradores (Dagster, Airflow) | 2 | Sem página |
| Consumir e escrever consultas GraphQL | 2 | Sem página |
| Operar Kubernetes | 2 | [Kubernetes](../../infraestrutura/kubernetes/index.md) |
| Montar uma infraestrutura de Prefect | 3 | [Instalar o servidor Prefect 3](../../infraestrutura/prefect/como-fazer/instalar-servidor-prefect-3.md) |

<!-- TODO(Laura): confirmar em que passo entram GraphQL e Kubernetes. Kubernetes também pode ser
passo 3, porque os workers do Prefect da BD rodam nele. -->

## Ver também

- [Trilha do engenheiro de dados](../explicacao/trilha-do-engenheiro-de-dados.md) — por que a trilha tem três passos
- [Onboarding](../../onboarding/index.md) — trilhas para quem está chegando
