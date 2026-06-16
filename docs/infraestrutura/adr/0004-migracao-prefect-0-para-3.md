---
tipo: adr
titulo: ADR-0004 — Migração do Prefect 0 para o Prefect 3
status: aceito
data: 2026-06-10
---
# ADR-0004 — Migração do Prefect 0 para o Prefect 3

## Contexto

A orquestração de pipelines rodava em **Prefect 0.15.9** (linha 1.x) no GKE, com um
stack pesado: Server + Hasura + Apollo Gateway + Towel (scheduler) + UI, API em
GraphQL, e 3 agents Kubernetes + 1 agent Vertex AI. A versão está obsoleta e diverge
bastante do Prefect atual.

Atualizar não era um upgrade simples: o **schema de banco do Prefect 3 é incompatível**
com o do Prefect 0, a API passou de GraphQL para REST, os *agents* deram lugar a
*workers* com *work pools*, e o chart Helm local customizado seria substituído pelo
chart oficial. Era inviável migrar in-place sem downtime e risco de perder os flows
agendados em produção.

A comparação detalhada entre os dois stacks está em
[Arquitetura do Prefect 3](../prefect/explicacao/arquitetura-prefect-3.md).

## Decisão

Migrar para o **Prefect 3** subindo o novo stack em um **namespace `prefect3` paralelo**,
sem derrubar o namespace `prefect` antigo até o cutover final. Em concreto:

- Banco **Cloud SQL novo**, provisionado por Terraform.
- Chart oficial `prefecthq/prefect-server` (deployment único; sem Hasura/Apollo/Towel).
- API **REST** e UI servidas pelo mesmo processo.
- **Workers + work pools** no lugar dos agents, **consolidados em 2**: `basedosdados`
  (prod) e `basedosdados-dev` (dev), ambos Kubernetes. **Sem Vertex AI** por ora.

## Consequências

### Positivas
- Stack muito mais simples de operar (um deployment em vez de cinco componentes).
- Versão suportada, com API REST estável e workers configuráveis por work pool.
- Recursos de CPU/memória dos jobs passam a ser governáveis por pool e por flow.

### Negativas
- O Prefect 3 self-hosted não tem API keys nativas — a autenticação externa precisou
  ser resolvida via token Django + nginx.
- Os flows precisaram ser portados para a sintaxe `@flow`/`@task` e para os work pools.

### Neutras
- Durante a transição convivem dois namespaces (`prefect` e `prefect3`) até o cutover.
- A consolidação para 2 workers reduz o número de agents, mas concentra os flows de
  prod num único pool.

## Alternativas consideradas

- **Upgrade in-place do Prefect 0 → 3** — descartado: o schema de banco é incompatível
  e exigiria migração destrutiva, com alto risco de downtime e perda de agendamentos.
- **Permanecer no Prefect 0.15.9** — descartado: versão obsoleta, sem suporte e cada
  vez mais distante do ecossistema atual do Prefect.

## Status

Aceito. Migração concluída para a maioria dos flows; cutover de DNS e desativação do
namespace `prefect` antigo realizados conforme o stack `prefect3` estabilizou.
