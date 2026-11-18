---
tipo: referencia
titulo: Recursos dos pods do Prefect 3
---
# Prefect 3 — Recursos dos Pods (CPU / Memória)

---

## 1. Dois tipos de pod, dois lugares para configurar

| Tipo | O que faz | Onde configurar |
|---|---|---|
| **Worker pod** | Processo que faz polling no Prefect server | `values.yaml` do Helm chart |
| **Job pod** | Executa o flow (criado a cada run) | `base_job_template` do work pool via API |

O que importa para OOM é sempre o **job pod**.

---

## 2. Limites atuais dos job pods (configurado em 2026-06-08)

| Pool | Requests | Limits |
|---|---|---|
| `basedosdados` (prod) | 500m CPU / 1Gi RAM | 2 CPU / 4Gi RAM |
| `basedosdados-dev` (dev) | 500m CPU / 1Gi RAM | 2 CPU / 4Gi RAM |

Os valores são defaults do schema do work pool — cada flow pode sobrescrever via `job_variables` sem afetar os demais.

**Por que limitar é melhor que deixar sem limite:**
- Sem limite, o kernel Linux mata processos de forma imprevisível quando o nó fica sob pressão de memória — qualquer pod pode ser vítima
- Com limite, o Kubernetes mata **só o pod que excedeu**, com erro claro (`OOMKilled`, exit code 137), sem afetar outros flows no mesmo nó

**Nós do cluster:** 3 × 4 CPU / ~12.6 GB alocável cada.

---

## 3. Ver limites atuais via API

```bash
source .env
curl -s "$PREFECT_API_URL/work_pools/basedosdados" \
  -H "Authorization: Bearer $PREFECT_API_KEY" | python3 -c "
import json, sys
d = json.load(sys.stdin)
props = d['base_job_template']['variables']['properties']
for k in ['memory_limit', 'memory_request', 'cpu_limit', 'cpu_request']:
    print(f'{k}: {props[k][\"default\"]}')
"
```

---

## 4. Alterar limites

Para mudar o limite global do pool ou sobrescrever os recursos de um flow específico (`job_variables`
no `flows.py`), siga [Como ajustar recursos de pod e resolver OOM](../como-fazer/ajustar-recursos-de-pod.md).

> **Regra:** manter o pool em 4Gi como baseline e usar `job_variables` nos flows que precisam de mais. Não inflar o baseline global para todos.

## 5. Flows com OOM identificados

| Flow | `memory_limit` sugerido |
|---|---|
| `br_anatel_telefonia_movel__densidade_brasil` | `8Gi` |
| `br_anatel_telefonia_movel__densidade_municipio` | `8Gi` |
| `br_anatel_telefonia_movel__densidade_uf` | `8Gi` |
| `br_anatel_telefonia_movel__microdados` | `8Gi` |
| `br_me_rais__*` | a confirmar |

---

## Ver também

- [Como ajustar recursos de pod e resolver OOM](../como-fazer/ajustar-recursos-de-pod.md)
