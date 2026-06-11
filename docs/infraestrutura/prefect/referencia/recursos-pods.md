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
containers = d['base_job_template']['job_configuration']['job_manifest']['spec']['template']['spec']['containers']
print(json.dumps(containers[0].get('resources', 'não definido'), indent=2))
"
```

---

## 4. Alterar o limite global do pool

```python
import json, urllib.request, os

api = os.environ['PREFECT_API_URL']
key = os.environ['PREFECT_API_KEY']

resources = {
    "requests": {"cpu": "500m", "memory": "1Gi"},
    "limits":   {"cpu": "2",    "memory": "4Gi"}
}

for pool in ["basedosdados", "basedosdados-dev"]:
    req = urllib.request.Request(f'{api}/work_pools/{pool}',
        headers={'Authorization': f'Bearer {key}'})
    with urllib.request.urlopen(req) as r:
        d = json.load(r)

    tmpl = d['base_job_template']
    containers = tmpl['job_configuration']['job_manifest']['spec']['template']['spec']['containers']
    containers[0]['resources'] = resources

    req = urllib.request.Request(f'{api}/work_pools/{pool}',
        data=json.dumps({"base_job_template": tmpl}).encode(),
        method='PATCH',
        headers={'Authorization': f'Bearer {key}', 'Content-Type': 'application/json'})
    with urllib.request.urlopen(req) as r:
        r.read()
    print(f'✅ {pool} atualizado')
```

---

## 5. Override de recursos por flow específico

Quando um flow precisa de mais memória que o padrão do pool, usar `job_variables` no `flows.py` — sobrescreve só para aquele deployment, sem afetar os demais:

```python
@flow(name="br_anatel_telefonia_movel__microdados", log_prints=True)
def br_anatel_telefonia_movel__microdados(...):
    ...

br_anatel_telefonia_movel__microdados.deploy_schedules = [
    {"cron": "0 3 * * *", "timezone": "America/Sao_Paulo"}
]

# Sobrescreve recursos só para este flow
br_anatel_telefonia_movel__microdados.job_variables = {
    "resources": {
        "requests": {"cpu": "1",    "memory": "2Gi"},
        "limits":   {"cpu": "4",    "memory": "8Gi"}
    }
}
```

O `deploy_flows.py` aplica os `job_variables` no momento do deploy. Os outros flows continuam com o padrão do pool.

> **Regra:** manter o pool em 4Gi como baseline e usar `job_variables` nos flows que precisam de mais. Não inflar o baseline global para todos.

---

## 6. Diagnosticar OOM

Quando um pod morre por OOM, verificar antes que seja deletado:

```bash
# Listar pods recentes no namespace
kubectl get pods -n prefect-worker-basedosdados --sort-by='.metadata.creationTimestamp' | tail -10

# Ver motivo da morte
kubectl describe pod <pod-name> -n prefect-worker-basedosdados | grep -A5 "Last State\|OOMKilled\|Exit Code\|Reason"
```

Se aparecer `OOMKilled` e `Exit Code: 137` → o pod excedeu o memory limit. Aumentar o limite via seção 4 ou 5.
