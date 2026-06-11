# Prefect

Documentação da orquestração de pipelines via Prefect 3, rodando em Kubernetes (GKE). Cobre configuração de workers, schedules, integração com o repositório `pipelines` e atualização de cobertura temporal via metadata.

## O que tem aqui

- **[Explicação](explicacao/workers.md)** — como os workers Prefect 3 funcionam no GKE (work pools, job variables, recursos por pod)
- **[Referência](referencia/comandos.md)** — comandos úteis para operar flows, workers e deployments
  - [Recursos dos pods](referencia/recursos-pods.md) — CPU/memória por tipo de flow
  - [Service accounts](referencia/service-accounts.md) — SAs do GCP usadas pelos workers
- **[Metadata onboarding](metadata-onboarding.md)** — como configurar `register_*` em um flow novo para atualizar cobertura temporal
- **[Migração Prefect 0 → 3](migracao-prefect-3/guia-migracao.md)** — guia de migração, estado atual e pendências
