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

Quando só um flow precisa de mais memória, use `job_variables` no `flows.py` — afeta
apenas aquele deployment:

```python
br_anatel_telefonia_movel__microdados.job_variables = {
    "resources": {
        "requests": {"cpu": "1", "memory": "2Gi"},
        "limits":   {"cpu": "4", "memory": "8Gi"},
    }
}
```

O `deploy_flows.py` aplica os `job_variables` no momento do deploy.

## Alterar o limite global do pool (job pods)

Os limites dos pods que executam os flows ficam no `base_job_template` do work pool,
via API. Ver os atuais:

```bash
source .env
curl -s "$PREFECT_API_URL/work_pools/basedosdados-dev" \
  -H "Authorization: Bearer $PREFECT_API_KEY" \
  | python3 -m json.tool | grep -A5 -i "memory\|cpu\|limit\|request"
```

Alterar via PATCH:

```bash
source .env
curl -s -X PATCH "$PREFECT_API_URL/work_pools/basedosdados-dev" \
  -H "Authorization: Bearer $PREFECT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "base_job_template": {
      "variables": {
        "properties": {
          "finished_job_ttl": {"default": 60},
          "resources": {
            "default": {
              "requests": {"cpu": "500m", "memory": "1Gi"},
              "limits":   {"cpu": "2",    "memory": "4Gi"}
            }
          }
        }
      }
    }
  }'
```

Para alterar os dois pools de uma vez, em Python:

```python
import json, urllib.request, os
api = os.environ['PREFECT_API_URL']
key = os.environ['PREFECT_API_KEY']
resources = {"requests": {"cpu": "500m", "memory": "1Gi"},
             "limits":   {"cpu": "2",    "memory": "4Gi"}}
for pool in ["basedosdados", "basedosdados-dev"]:
    req = urllib.request.Request(f'{api}/work_pools/{pool}',
        headers={'Authorization': f'Bearer {key}'})
    with urllib.request.urlopen(req) as r:
        tmpl = json.load(r)['base_job_template']
    tmpl['job_configuration']['job_manifest']['spec']['template']['spec']['containers'][0]['resources'] = resources
    req = urllib.request.Request(f'{api}/work_pools/{pool}',
        data=json.dumps({"base_job_template": tmpl}).encode(), method='PATCH',
        headers={'Authorization': f'Bearer {key}', 'Content-Type': 'application/json'})
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

Após o PATCH, confirme com o `curl` de leitura acima que `resources` reflete o novo
valor. No próximo run do flow, `kubectl describe pod` não deve mais mostrar `OOMKilled`.

## Ver também

- [Recursos dos pods (referência)](../referencia/recursos-pods.md)
- [Como fazer deploy de um flow](fazer-deploy-de-flow.md)
- [Como configurar os workers](configurar-workers.md)
