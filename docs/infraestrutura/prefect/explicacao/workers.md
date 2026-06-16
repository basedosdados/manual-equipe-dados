---
tipo: explicacao
titulo: Workers do Prefect 3 (substitutos dos Agents)
---
# Workers do Prefect 3 (substitutos dos Agents)

## Novo conceito: Work Pools

No Prefect 1.x o agent sabia diretamente onde executar os flows (via labels).
No Prefect 3 existe uma camada intermediária chamada **Work Pool**:

```
Prefect Server
     ↓
  Work Pool  (define a infraestrutura: Kubernetes, Vertex AI...)
     ↓
   Worker    (processo que fica "ouvindo" o pool e executa os flows)
     ↓
  Flow Run   (Job Kubernetes ou job no Vertex AI)
```

O **work pool** descreve a infraestrutura de execução (template do job Kubernetes,
recursos, namespace). O **worker** é o processo que faz polling no pool e materializa
cada flow run como um Job no cluster.

## Comparativo: Agents antigos → Workers

| Agent antigo | Namespace antigo | Tipo | Work Pool Prefect 3 |
|---|---|---|---|
| `basedosdados` | `prefect-agent-basedosdados` | Kubernetes | `basedosdados` |
| `basedosdados-perguntas` | `prefect-agent-basedosdados-perguntas` | Kubernetes | consolidado em `basedosdados` |
| `basedosdados-projetos` | `prefect-agent-basedosdados-projetos` | Kubernetes | consolidado em `basedosdados` |
| `basedosdados-dev-gcp-vertex` | `prefect-agent-basedosdados-dev-gcp-vertex` | Vertex AI | `basedosdados-dev` (Kubernetes por ora) |

### Decisão: consolidar em 2 workers

Os 3 agents Kubernetes foram consolidados num único worker de produção
(`basedosdados`) e um worker de dev (`basedosdados-dev`). Vertex AI não é usado neste
momento — os flows de dev também rodam em Kubernetes. O racional está na
[ADR-0004](../../adr/0004-migracao-prefect-0-para-3.md).

## O que muda para os flows

Os flows passam a referenciar o **work pool** no deploy, em vez dos antigos labels do
agent. Conceitualmente, o `work_pool_name` substitui o roteamento por labels. O
passo a passo de deploy está em
[Como fazer deploy de um flow](../como-fazer/fazer-deploy-de-flow.md).

## Ver também

- [Arquitetura do Prefect 3 no GKE](arquitetura-prefect-3.md)
- [Como configurar os workers](../como-fazer/configurar-workers.md)
- [Como ajustar recursos de pod](../como-fazer/ajustar-recursos-de-pod.md)
- [ADR-0004 — Migração do Prefect 0 para o Prefect 3](../../adr/0004-migracao-prefect-0-para-3.md)
- [Prefect 3 — Workers docs](https://docs.prefect.io/latest/deploy/infrastructure-concepts/workers/)
