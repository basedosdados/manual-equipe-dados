# Prefect

Documentação da orquestração de pipelines via Prefect 3, rodando em Kubernetes (GKE).
Cobre a arquitetura do stack, configuração de servidor e workers, deploy de flows,
recursos dos pods e registro de metadata.

## Explicação — entender como funciona

- [Arquitetura do Prefect 3 no GKE](explicacao/arquitetura-prefect-3.md) — o stack atual e o que mudou desde o Prefect 0
- [Workers (substitutos dos Agents)](explicacao/workers.md) — work pools, workers e a consolidação em 2 pools

## Como fazer — executar uma tarefa

- [Instalar o servidor Prefect 3 no GKE](como-fazer/instalar-servidor-prefect-3.md) — Terraform, namespace, sealed secret e Helm
- [Configurar os workers](como-fazer/configurar-workers.md) — RBAC, credenciais GCP e work pools via API
- [Fazer deploy de um flow](como-fazer/fazer-deploy-de-flow.md) — autenticação, CI/CD e disparo de runs
- [Ajustar recursos de pod e resolver OOM](como-fazer/ajustar-recursos-de-pod.md) — CPU/memória por pool e por flow
- [Registrar metadados no backend](como-fazer/registrar-metadata.md) — popular `Dataset`, `Table`, `Coverage` etc.

## Referência — consultar valores

- [Recursos dos pods](referencia/recursos-pods.md) — limites atuais de CPU/memória por pool
- [Service accounts do GCP](referencia/service-accounts.md) — SAs usadas pelos workers/agents

## Decisões (ADR)

- [ADR-0004 — Migração do Prefect 0 para o Prefect 3](../adr/0004-migracao-prefect-0-para-3.md)
