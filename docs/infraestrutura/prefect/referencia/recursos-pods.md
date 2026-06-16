---
tipo: referencia
titulo: Recursos dos pods do Prefect 3 (CPU / Memória)
---
# Recursos dos pods do Prefect 3 (CPU / Memória)

Valores de referência dos recursos de CPU e memória dos pods do Prefect 3. Para alterar
limites ou diagnosticar OOM, ver
[Como ajustar recursos de pod](../como-fazer/ajustar-recursos-de-pod.md).

## Dois tipos de pod, dois lugares de configuração

| Tipo | O que faz | Onde se configura |
|---|---|---|
| **Worker pod** | Processo que faz polling no Prefect server | `values.yaml` do Helm chart |
| **Job pod** | Executa o flow (criado a cada run) | `base_job_template` do work pool, via API |

O que importa para OOM é sempre o **job pod**.

## Limites dos job pods (configurado em 2026-06-08)

| Pool | Requests | Limits |
|---|---|---|
| `basedosdados` (prod) | 500m CPU / 1Gi RAM | 2 CPU / 4Gi RAM |
| `basedosdados-dev` (dev) | 500m CPU / 1Gi RAM | 2 CPU / 4Gi RAM |

## Limites dos worker pods (ambos os workers)

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi
```

## Pools disponíveis

| Pool | Namespace | Quando usar |
|---|---|---|
| `basedosdados` | `prefect-worker-basedosdados` | flows de prod |
| `basedosdados-dev` | `prefect-worker-basedosdados-dev` | flows de teste / dev |

## Capacidade do cluster

Nós do cluster: 3 × 4 CPU / ~12.6 GB alocável cada.

## Por que limitar é melhor que não limitar

- Sem limite, o kernel Linux mata processos de forma imprevisível quando o nó fica sob
  pressão de memória — qualquer pod pode ser vítima.
- Com limite, o Kubernetes mata **só o pod que excedeu**, com erro claro
  (`OOMKilled`, exit code 137), sem afetar outros flows no mesmo nó.

## Ver também

- [Como ajustar recursos de pod e resolver OOM](../como-fazer/ajustar-recursos-de-pod.md)
- [Workers (substitutos dos Agents)](../explicacao/workers.md)
