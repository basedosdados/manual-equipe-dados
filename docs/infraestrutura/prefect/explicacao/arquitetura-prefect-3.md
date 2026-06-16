---
tipo: explicacao
titulo: Arquitetura do Prefect 3 no GKE
---
# Arquitetura do Prefect 3 no GKE

O Prefect orquestra os flows da Base dos Dados rodando em Kubernetes (GKE). Esta
página explica como o stack está montado e o que mudou em relação ao Prefect 0.15.9
(linha 1.x), para dar contexto a quem opera ou evolui a infraestrutura. A decisão de
migrar e suas alternativas estão registradas em
[ADR-0004](../../adr/0004-migracao-prefect-0-para-3.md).

## O stack antigo (Prefect 0.15.9)

O Prefect 0 era um conjunto de componentes que conversavam por GraphQL:

| Componente   | Detalhe                                                            |
| ------------ | ------------------------------------------------------------------ |
| Namespace    | `prefect`                                                          |
| Helm chart   | local em `k8s/prefect/chart/`                                      |
| Banco        | Cloud SQL PostgreSQL 13, database `prefect`, user `prefect`        |
| URL          | `prefect.basedosdados.org`                                         |
| Componentes  | Server + Hasura (v2.8.1) + Apollo Gateway + Towel (scheduler) + UI |
| Agents       | 3× Kubernetes + 1× Vertex AI (`southamerica-east1`)                |
| Job template | `gs://basedosdados-dev/prefect_job_template/template.yaml`         |

## O que muda no Prefect 3

| Aspecto      | Prefect 0.15.9                          | Prefect 3                                |
| ------------ | --------------------------------------- | ---------------------------------------- |
| API          | GraphQL (Hasura + Apollo)               | REST API (FastAPI)                       |
| Componentes  | Server + Hasura + Apollo + Towel + UI   | Apenas `prefect-server`                  |
| Agents       | `KubernetesAgent` / `VertexAIAgent`     | Workers com Work Pools                   |
| Job template | YAML no GCS                             | Configurado no Work Pool                 |
| Auth         | Sem auth nativo                         | API Keys nativo (ver ressalva abaixo)    |
| DB Schema    | Incompatível — requer banco novo        | Schema próprio do Prefect 3              |
| Helm Chart   | Chart local customizado                 | Chart oficial `prefecthq/prefect-helm`   |

A simplificação central é que o Prefect 3 colapsa cinco componentes em um único
`prefect-server` (UI + API REST no mesmo processo), e troca os agents — que sabiam
diretamente onde executar os flows — pela camada de **work pools** intermediada por
**workers**. Esse conceito está detalhado em [Workers](workers.md).

> **Ressalva sobre auth:** o Prefect 3 *self-hosted* não expõe API keys nativas. A
> autenticação externa é feita por token Django validado pelo nginx — ver
> [Como fazer deploy de um flow](../como-fazer/fazer-deploy-de-flow.md).

## Estratégia de migração: namespace paralelo

Para evitar downtime, o stack do Prefect 3 subiu em um **namespace `prefect3` novo**, em
paralelo ao `prefect` antigo, com banco Cloud SQL próprio. O cutover (troca de DNS e
desativação do namespace antigo) só aconteceu depois que workers e flows estavam
estáveis no `prefect3`. O racional completo está na [ADR-0004](../../adr/0004-migracao-prefect-0-para-3.md).

## Ver também

- [ADR-0004 — Migração do Prefect 0 para o Prefect 3](../../adr/0004-migracao-prefect-0-para-3.md)
- [Workers (substitutos dos Agents)](workers.md)
- [Como instalar o servidor Prefect 3 no GKE](../como-fazer/instalar-servidor-prefect-3.md)
