---
tipo: como-fazer
titulo: Como ajustar recursos de pod e resolver OOM
---
# Como ajustar recursos de pod e resolver OOM

Aumenta CPU/memória dos pods do Prefect 3 e diagnostica `OOMKilled`. Para os limites
atuais e a distinção entre worker pod e job pod, ver
[Recursos dos pods](../referencia/recursos-pods.md).

## Pré-requisitos

- `PREFECT_API_URL` e `PREFECT_API_KEY` exportados (ver
  [Como fazer deploy de um flow](fazer-deploy-de-flow.md), passo 1).
- `kubectl` autenticado no cluster; repositório `bd/iac` clonado.

> **Regra geral:** o que importa para OOM é o **job pod** (executa o flow). Mantenha o
> baseline do pool em 4Gi e use `job_variables` nos flows que precisam de mais — não
> infle o baseline global para todos.

## Diagnosticar OOM

Quando um pod morre por OOM, verifique antes que seja deletado:

```bash
kubectl get pods -n prefect-worker-basedosdados --sort-by='.metadata.creationTimestamp' | tail -10
kubectl describe pod <pod-name> -n prefect-worker-basedosdados | grep -A5 "Last State\|OOMKilled\|Exit Code\|Reason"
```

`OOMKilled` + `Exit Code: 137` → o pod excedeu o memory limit. Aumente o limite com um
dos caminhos abaixo.

## Override por flow (recomendado)

Quando só um flow precisa de mais memória, declare `job_variables` no `flows.py` — afeta
apenas aquele deployment:

```python
br_anatel_telefonia_movel__microdados.deploy(
    name="br_anatel_telefonia_movel__microdados",
    work_pool_name="basedosdados",
    job_variables={
        "memory_limit":   "8Gi",
        "memory_request": "2Gi",
    },
    ...
)
```

O `deploy_flows.py` aplica os `job_variables` no momento do deploy.

## Alterar o limite global do pool (job pods)

Ver os limites atuais:

```bash
source .env
curl -s "$PREFECT_API_URL/work_pools/basedosdados-dev" \
  -H "Authorization: Bearer $PREFECT_API_KEY" | python3 -c "
import json, sys
d = json.load(sys.stdin)
props = d['base_job_template']['variables']['properties']
for k in ['memory_limit', 'memory_request', 'cpu_limit', 'cpu_request']:
    print(f'{k}: {props[k][\"default\"]}')
"
```

Alterar os defaults via Python (ambos os pools de uma vez):

```python
import json, urllib.request, os

api = os.environ['PREFECT_API_URL']
key = os.environ['PREFECT_API_KEY']
headers = {"Authorization": f"Bearer {key}", "Content-Type": "application/json"}

new_defaults = {
    "memory_limit":   "4Gi",
    "memory_request": "1Gi",
    "cpu_limit":      "2",
    "cpu_request":    "500m",
}

for pool in ["basedosdados", "basedosdados-dev"]:
    req = urllib.request.Request(f'{api}/work_pools/{pool}', headers={"Authorization": f"Bearer {key}"})
    with urllib.request.urlopen(req) as r:
        d = json.load(r)
    props = d['base_job_template']['variables']['properties']
    for k, v in new_defaults.items():
        props[k]['default'] = v
    req = urllib.request.Request(f'{api}/work_pools/{pool}',
        data=json.dumps({"base_job_template": d['base_job_template']}).encode(),
        method='PATCH', headers=headers)
    with urllib.request.urlopen(req) as r:
        r.read()
    print(f'✅ {pool} atualizado')
```

## Alterar o pod do worker (polling)

O worker pod só faz polling — raramente precisa de mais recursos. Edite o `values.yaml`
do chart e faça helm upgrade:

```bash
# k8s/prefect_workers/basedosdados-dev/chart/values.yaml (resources.requests/limits)
helm upgrade prefect-worker-basedosdados-dev \
  -n prefect-worker-basedosdados-dev \
  prefecthq/prefect-worker \
  -f k8s/prefect_workers/basedosdados-dev/chart/values.yaml
```

## Verificação

Após o PATCH, confirme com o `curl` de leitura acima que os defaults refletem o novo
valor. No próximo run do flow, `kubectl describe pod` não deve mais mostrar `OOMKilled`.

## Ver também

- [Recursos dos pods (referência)](../referencia/recursos-pods.md)
- [Como fazer deploy de um flow](fazer-deploy-de-flow.md)
- [Como configurar os workers](configurar-workers.md)
