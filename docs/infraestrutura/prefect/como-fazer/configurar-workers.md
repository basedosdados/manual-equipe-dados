---
tipo: como-fazer
titulo: Como configurar os workers do Prefect 3
---
# Como configurar os workers do Prefect 3

Cria e conecta os workers Kubernetes (`basedosdados` e `basedosdados-dev`) ao servidor
Prefect 3, incluindo RBAC, credenciais GCP e a configuração dos work pools via API. Para
o conceito de work pools e workers, ver [Workers](../explicacao/workers.md).

## Pré-requisitos

- Servidor Prefect 3 instalado (ver [Como instalar o servidor](instalar-servidor-prefect-3.md)).
- `kubectl`, `helm`, `kubeseal` e `gcloud` autenticados; repositório `bd/iac` clonado.
- Port-forward ativo para acessar a UI/API local:
  `kubectl --namespace prefect3 port-forward svc/prefect-server 4200:4200`.

## Passos

### 1. Criar os Work Pools no servidor

Na UI (`http://localhost:4200`) → **Work Pools → Create**, tipo `kubernetes`. Crie um
pool por worker (`basedosdados` e `basedosdados-dev`).

### 2. Autenticar com a SA do Terraform no GKE

Necessário para criar RBAC — a conta pessoal não tem permissão. A SA precisa de
`roles/container.admin`:

```bash
gcloud auth activate-service-account --key-file=/path/to/terraform-sa.json
gcloud container clusters get-credentials basedosdados-dev-gke \
  --zone=us-central1-c --project=basedosdados-dev
```

### 3. Criar RBAC para os workers

Os workers precisam criar e gerenciar Jobs no próprio namespace:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prefect-worker
  namespace: <namespace-do-worker>
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: prefect-worker
  namespace: <namespace-do-worker>
rules:
  - apiGroups: ["batch"]
    resources: ["jobs"]
    verbs: ["create", "get", "list", "watch", "delete"]
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: prefect-worker
  namespace: <namespace-do-worker>
subjects:
  - kind: ServiceAccount
    name: prefect-worker
roleRef:
  kind: Role
  name: prefect-worker
  apiGroup: rbac.authorization.k8s.io
```

### 4. Criar sealed secrets com credenciais GCP

Diferente do server, os workers precisam das credenciais GCP para criar Jobs com acesso
ao GCP, BigQuery, GCS etc. Os secrets (mesmos dos agents antigos): `gcp-credentials`
(`BASEDOSDADOS_CONFIG`, `BASEDOSDADOS_CREDENTIALS_PROD/STAGING`), `vault-credentials`,
`gcp-sa` (`creds.json`), `credentials-dev`/`credentials-prod`. Ver
[Service accounts](../referencia/service-accounts.md) para quais SAs usar.

```bash
kubectl create secret generic gcp-credentials \
  --namespace <namespace-do-worker> \
  --from-literal=BASEDOSDADOS_CONFIG=... \
  --dry-run=client -o yaml | \
  kubeseal --namespace <namespace-do-worker> --format yaml > secret-01_sealed.yaml
```

### 5. Criar os values.yaml e instalar via Helm

Usando o chart oficial `prefecthq/prefect-worker`:

```yaml
# k8s/prefect_workers/<nome>/chart/values.yaml
fullnameOverride: "prefect-worker-<nome>"
worker:
  apiConfig: "server"
  config:
    workPool: "<nome-do-work-pool>"
  serverApiConfig:
    apiUrl: "http://prefect-server.prefect3.svc.cluster.local:4200/api"
  resources:
    requests: {cpu: 100m, memory: 128Mi}
    limits:   {cpu: 500m, memory: 256Mi}
  extraEnvVarsSecret: "gcp-credentials"
  extraVolumes:
    - name: gcp-sa
      secret: {secretName: gcp-sa}
  extraVolumeMounts:
    - name: gcp-sa
      mountPath: /mnt/
      readOnly: true
  env:
    - name: GOOGLE_APPLICATION_CREDENTIALS
      value: /mnt/creds.json
```

```bash
kubectl apply -f k8s/prefect_workers/basedosdados/namespace.yaml
kubectl apply -f k8s/prefect_workers/basedosdados-dev/namespace.yaml

helm repo add prefecthq https://prefecthq.github.io/prefect-helm
helm repo update

helm upgrade --install prefect-worker-basedosdados \
  -n prefect-worker-basedosdados \
  prefecthq/prefect-worker \
  -f k8s/prefect_workers/basedosdados/chart/values.yaml

helm upgrade --install prefect-worker-basedosdados-dev \
  -n prefect-worker-basedosdados-dev \
  prefecthq/prefect-worker \
  -f k8s/prefect_workers/basedosdados-dev/chart/values.yaml
```

### 6. Configurar os work pools via API

**Obrigatório após criar os pools na UI.** Requer o port-forward ativo.

**Namespace dos jobs** — por padrão o Prefect cria jobs em `default`, mas o RBAC só
permite o próprio namespace do worker:

```bash
# Work pool de produção
curl -s http://localhost:4200/api/work_pools/basedosdados | python3 -c "
import sys, json
wp = json.load(sys.stdin)
wp['base_job_template']['variables']['properties']['namespace']['default'] = 'prefect-worker-basedosdados'
json.dump({'base_job_template': wp['base_job_template']}, open('/tmp/wp-prod-patch.json','w'))
"
curl -s -X PATCH http://localhost:4200/api/work_pools/basedosdados \
  -H "Content-Type: application/json" -d @/tmp/wp-prod-patch.json
# Repetir para basedosdados-dev → namespace prefect-worker-basedosdados-dev
```

**Credenciais GCP nos pods de flow (`envFrom`)** — injeta o secret `gcp-credentials` em
todos os pods do pool, disponibilizando `BASEDOSDADOS_CONFIG` e as credenciais
automaticamente:

```bash
curl -s http://localhost:4200/api/work_pools/basedosdados-dev | python3 -c "
import sys, json
wp = json.load(sys.stdin)
c = wp['base_job_template']['job_configuration']['job_manifest']['spec']['template']['spec']['containers'][0]
c['envFrom'] = [{'secretRef': {'name': 'gcp-credentials'}}]
json.dump({'base_job_template': wp['base_job_template']}, open('/tmp/wp-dev-patch.json','w'))
"
curl -s -X PATCH http://localhost:4200/api/work_pools/basedosdados-dev \
  -H "Content-Type: application/json" -d @/tmp/wp-dev-patch.json
```

**TTL dos jobs concluídos** — sem isso os pods concluídos ficam para sempre no cluster:

```bash
for pool in basedosdados basedosdados-dev; do
  curl -s "http://localhost:4200/api/work_pools/${pool}" -o /tmp/wp.json
  python3 -c "
import json
wp = json.load(open('/tmp/wp.json'))
wp['base_job_template']['variables']['properties']['finished_job_ttl']['default'] = 60
json.dump({'base_job_template': wp['base_job_template']}, open('/tmp/wp-patch.json','w'))
"
  curl -s -o /dev/null -w "${pool}: HTTP %{http_code}\n" \
    -X PATCH "http://localhost:4200/api/work_pools/${pool}" \
    -H "Content-Type: application/json" -d @/tmp/wp-patch.json
done
```

## Verificação

Na UI, o work pool deve mostrar o worker como **online**. Via kubectl:

```bash
kubectl get pods -n prefect-worker-basedosdados
kubectl logs -n prefect-worker-basedosdados deployment/prefect-worker --tail=30
```

## Problemas comuns

- **Worker não sobe / falta `prefect-kubernetes`** — a imagem `3-latest` não inclui o
  pacote para workers Kubernetes. Use a tag `3-python3.12-kubernetes`.
- **Jobs criados no namespace errado (`default`)** — o passo 6 (namespace via API) não
  foi aplicado; o RBAC só autoriza o namespace do próprio worker.

## Ver também

- [Workers (substitutos dos Agents)](../explicacao/workers.md)
- [Como instalar o servidor Prefect 3 no GKE](instalar-servidor-prefect-3.md)
- [Como fazer deploy de um flow](fazer-deploy-de-flow.md)
- [Como ajustar recursos de pod](ajustar-recursos-de-pod.md)
