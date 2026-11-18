---
tipo: explicacao
titulo: Workers do Prefect 3
---
# Prefect 3 — Workers (substitutos dos Agents)

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

---

## Comparativo: Agents atuais → Workers

| Agent atual | Namespace atual | Tipo | Work Pool Prefect 3 |
|---|---|---|---|
| `basedosdados` | `prefect-agent-basedosdados` | Kubernetes | `basedosdados` ✅ |
| `basedosdados-perguntas` | `prefect-agent-basedosdados-perguntas` | Kubernetes | consolidado em `basedosdados` ✅ |
| `basedosdados-projetos` | `prefect-agent-basedosdados-projetos` | Kubernetes | consolidado em `basedosdados` ✅ |
| `basedosdados-dev-gcp-vertex` | `prefect-agent-basedosdados-dev-gcp-vertex` | Vertex AI | `basedosdados-dev` ✅ (Kubernetes por ora) |

Os 3 agents Kubernetes foram consolidados num worker de produção (`basedosdados`) e num de dev
(`basedosdados-dev`), e o Vertex AI não é usado por ora. A decisão e as alternativas estão no
[ADR-0004](../../adr/0004-migracao-prefect-0-para-3.md).

---

## Como configurar

O passo a passo para criar os work pools, o RBAC, os secrets e instalar os workers via Helm está em
[Como configurar os workers](../como-fazer/configurar-workers.md).

---

## Diferenças importantes para os flows

Os flows precisam ser atualizados para usar os novos work pools ao fazer deploy:

```python
# Prefect 3
from prefect import flow
from prefect.deployments import Deployment

@flow
def meu_flow():
    pass

Deployment.build_from_flow(
    flow=meu_flow,
    name="meu-deploy",
    work_pool_name="<nome-do-work-pool>",  # novo — substituiu labels
)
```

---

## Referências

- [Prefect 3 — Workers docs](https://docs.prefect.io/latest/deploy/infrastructure-concepts/workers/)
- [Prefect Helm chart — prefect-worker](https://github.com/PrefectHQ/prefect-helm/tree/main/charts/prefect-worker)
- [Prefect 3 — Vertex AI Worker](https://docs.prefect.io/latest/deploy/infrastructure-examples/serverless/)
