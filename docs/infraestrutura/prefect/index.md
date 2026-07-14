# Prefect

Documentação da orquestração de pipelines via Prefect 3, rodando em Kubernetes (GKE). Cobre configuração de workers, schedules, integração com o repositório `pipelines` e atualização de cobertura temporal via metadata.

## O que tem aqui

- **[Explicação](explicacao/workers.md)** — como os workers Prefect 3 funcionam no GKE (work pools, job variables, recursos por pod)
- **[Referência](referencia/comandos.md)** — comandos úteis para operar flows, workers e deployments
  - [Recursos dos pods](referencia/recursos-pods.md) — CPU/memória por tipo de flow
  - [Gestão de Schedules](referencia/gestao-de-schedules.md) — sistema de desativação automática de flows com falhas
  - [Service accounts](referencia/service-accounts.md) — SAs do GCP usadas pelos workers
- **[Como fazer](como-fazer/flow-failed-webhook.md)** — configurar a automação Flow Failed Webhook
- **[Runbooks](runbooks/gestao-de-schedules.md)** — operação e troubleshooting da gestão de schedules
- **[Metadata onboarding](metadata-onboarding.md)** — como configurar `register_*` em um flow novo para atualizar cobertura temporal
