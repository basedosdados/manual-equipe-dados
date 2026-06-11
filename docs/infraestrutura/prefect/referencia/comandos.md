# Prefect 3 — Comandos de Deploy

## Pré-requisitos
- Terraform 1.1.9 instalado
- `kubectl` configurado apontando para o cluster GKE
- `helm` instalado
- `kubeseal` instalado
- `gcloud` autenticado

---

## 1. Terraform — Criar banco de dados

```bash
cd /mnt/d/repositorios/bd/iac/terraform
```

```bash
source .env
```

```bash
terraform init
```

```bash
terraform plan -target=module.cloudsql.random_password.prefect3_db_password -target=module.cloudsql.google_secret_manager_secret.prefect3_db_password -target=module.cloudsql.google_secret_manager_secret_version.prefect3_db_password -target=module.cloudsql.google_sql_user.prefect3 -target=module.cloudsql.google_sql_database.prefect3
```

```bash
terraform apply -target=module.cloudsql.random_password.prefect3_db_password -target=module.cloudsql.google_secret_manager_secret.prefect3_db_password -target=module.cloudsql.google_secret_manager_secret_version.prefect3_db_password -target=module.cloudsql.google_sql_user.prefect3 -target=module.cloudsql.google_sql_database.prefect3
```

---

## 2. Pegar a senha gerada

```bash
gcloud secrets versions access latest --secret=prefect3-db-password
```

---
```
 kubectl create secret generic prefect3-db-connection --namespace prefect3 --from-literal=connection-string="postgresql+asyncpg://prefect3:CPL%5B%3AW%28MCLabTontq8Qoyc@cloud-sql-proxy:5432/prefect3" --dry-run=client -o yaml |    kubeseal --namespace prefect3 --format yaml > /mnt/d/repositorios/bd/iac/k8s/prefect3/secret-00_sealed.yaml
```
## 3. Criar o sealed secret do banco

> Substitua `<senha>` pelo valor obtido no passo anterior

```bash
kubectl create secret generic prefect3-db-connection --namespace prefect3 --from-literal=connection-string="postgresql+asyncpg://prefect3:<senha>@cloud-sql-proxy:5432/prefect3" --dry-run=client -o yaml | kubeseal --namespace prefect3 --format yaml > /mnt/d/repositorios/bd/iac/k8s/prefect3/secret-00_sealed.yaml
```
```bash
kubectl create secret generic prefect3-db-connection --namespace prefect3 --from-literal=connection-string="postgresql+asyncpg://prefect3:CPL[:W(MCLabTontq8Qoyc@cloud-sql-proxy:5432/prefect3" --dry-run=client -o yaml | kubeseal --namespace prefect3 --format yaml > /mnt/d/repositorios/bd/iac/k8s/prefect3/secret-00_sealed.yaml
```

---

## 4. Aplicar os manifests do namespace

```bash
cd /mnt/d/repositorios/bd/iac
```

```bash
kubectl apply -f k8s/prefect3/namespace.yaml
```

```bash
kubectl apply -f k8s/prefect3/db-service.yaml
```

```bash
kubectl apply -f k8s/prefect3/issuer.yaml
```

```bash
kubectl apply -f k8s/prefect3/secret-00_sealed.yaml
```

```bash
kubectl apply -f k8s/prefect3/ingress.yaml
```

---

## 5. Instalar o chart do Prefect 3

```bash
helm repo add prefecthq https://prefecthq.github.io/prefect-helm
```

```bash
helm repo update
```

```bash
helm upgrade --install prefect-server -n prefect3 prefecthq/prefect-server -f /mnt/d/repositorios/bd/iac/k8s/prefect3/chart/values.yaml
```

---

## Verificar o deploy

```bash
kubectl get pods -n prefect3
```

```bash
kubectl logs -n prefect3 -l app.kubernetes.io/name=prefect-server --tail=50
```

```bash
kubectl get ingress -n prefect3
```

---

## 6. Autenticar com a SA do Terraform no GKE

> Necessário para criar RBAC (Roles/RoleBindings) — a conta pessoal não tem essa permissão.
> A SA precisa ter `roles/container.admin` no projeto.

```bash
gcloud auth activate-service-account --key-file=/mnt/d/Downloads/terraform-sa.json
```

```bash
gcloud container clusters get-credentials basedosdados-dev-gke --zone=us-central1-c --project=basedosdados-dev
```

---

## 7. Criar os workers do Prefect 3

```bash
kubectl apply -f k8s/prefect_workers/basedosdados/namespace.yaml
kubectl apply -f k8s/prefect_workers/basedosdados-dev/namespace.yaml
```

```bash
helm repo add prefecthq https://prefecthq.github.io/prefect-helm
helm repo update
```

```bash
helm upgrade --install prefect-worker-basedosdados \
  -n prefect-worker-basedosdados \
  prefecthq/prefect-worker \
  -f k8s/prefect_workers/basedosdados/chart/values.yaml
```

```bash
helm upgrade --install prefect-worker-basedosdados-dev \
  -n prefect-worker-basedosdados-dev \
  prefecthq/prefect-worker \
  -f k8s/prefect_workers/basedosdados-dev/chart/values.yaml
```

---

## 8. Configurar Work Pools via API

> **Obrigatório após criar os work pools na UI.**
> Requer port-forward ativo: `kubectl port-forward svc/prefect-server 4200:4200 -n prefect3`

### Namespace

> Por padrão o Prefect cria jobs no namespace `default`, mas o RBAC do worker
> só permite criar jobs no próprio namespace.

```bash
# Work pool de produção
curl -s http://localhost:4200/api/work_pools/basedosdados | python3 -c "
import sys, json
wp = json.load(sys.stdin)
wp['base_job_template']['variables']['properties']['namespace']['default'] = 'prefect-worker-basedosdados'
with open('/tmp/wp-prod-patch.json', 'w') as f:
    json.dump({'base_job_template': wp['base_job_template']}, f)
"
curl -s -X PATCH http://localhost:4200/api/work_pools/basedosdados \
  -H "Content-Type: application/json" -d @/tmp/wp-prod-patch.json
```

```bash
# Work pool de dev
curl -s http://localhost:4200/api/work_pools/basedosdados-dev | python3 -c "
import sys, json
wp = json.load(sys.stdin)
wp['base_job_template']['variables']['properties']['namespace']['default'] = 'prefect-worker-basedosdados-dev'
with open('/tmp/wp-dev-patch.json', 'w') as f:
    json.dump({'base_job_template': wp['base_job_template']}, f)
"
curl -s -X PATCH http://localhost:4200/api/work_pools/basedosdados-dev \
  -H "Content-Type: application/json" -d @/tmp/wp-dev-patch.json
```

### Credenciais GCP nos pods de flow (envFrom)

> O work pool `basedosdados-dev` foi configurado para injetar o secret `gcp-credentials`
> em todos os pods via `envFrom`. Isso disponibiliza `BASEDOSDADOS_CONFIG`,
> `BASEDOSDADOS_CREDENTIALS_PROD` e `BASEDOSDADOS_CREDENTIALS_STAGING` automaticamente.
> O secret foi criado via sealed secret em `k8s/prefect_workers/basedosdados-dev/secret-01_sealed.yaml`
> usando a SA `prefect@basedosdados-dev.iam.gserviceaccount.com` (só acesso a recursos dev).

```bash
curl -s http://localhost:4200/api/work_pools/basedosdados-dev | python3 -c "
import sys, json
wp = json.load(sys.stdin)
container = wp['base_job_template']['job_configuration']['job_manifest']['spec']['template']['spec']['containers'][0]
container['envFrom'] = [{'secretRef': {'name': 'gcp-credentials'}}]
with open('/tmp/wp-dev-patch.json', 'w') as f:
    json.dump({'base_job_template': wp['base_job_template']}, f)
"
curl -s -X PATCH http://localhost:4200/api/work_pools/basedosdados-dev \
  -H "Content-Type: application/json" -d @/tmp/wp-dev-patch.json
```

---

### TTL dos jobs concluídos

> Por padrão os pods de jobs concluídos ficam para sempre no cluster.
> Com TTL de 60s eles são apagados automaticamente pelo Kubernetes.

```bash
for pool in basedosdados basedosdados-dev; do
  curl -s "http://localhost:4200/api/work_pools/${pool}" -o /tmp/wp.json
  python3 -c "
import json
with open('/tmp/wp.json') as f:
    wp = json.load(f)
wp['base_job_template']['variables']['properties']['finished_job_ttl']['default'] = 60
with open('/tmp/wp-patch.json', 'w') as f:
    json.dump({'base_job_template': wp['base_job_template']}, f)
"
  curl -s -o /dev/null -w "${pool}: HTTP %{http_code}\n" \
    -X PATCH "http://localhost:4200/api/work_pools/${pool}" \
    -H "Content-Type: application/json" -d @/tmp/wp-patch.json
done
```

---

## Verificar os workers

```bash
kubectl get pods -n prefect-worker-basedosdados
kubectl get pods -n prefect-worker-basedosdados-dev
```

```bash
kubectl logs -n prefect-worker-basedosdados deployment/prefect-worker --tail=30
kubectl logs -n prefect-worker-basedosdados-dev deployment/prefect-worker --tail=30
```

---

## 9. Autenticação externa via token Django

O Prefect 3 self-hosted não tem API keys nativas. A autenticação é feita pelo nginx via Django:

1. Cliente envia `Authorization: Bearer <token>` no header
2. O nginx valida o token chamando `https://backend.basedosdados.org/auth/`
3. Se válido, a requisição é repassada para o Prefect API

**Como criar um token permanente:**
1. Acessar `https://backend.basedosdados.org/admin/authtoken/token/`
2. Criar token para o usuário de serviço
3. Usar o token como `PREFECT_API_KEY` (o Prefect envia automaticamente como Bearer)

**Testar o token:**
```bash
curl -s https://prefect3.basedosdados.org/api/work_pools/ \
  -H "Authorization: Bearer <token>" | python3 -m json.tool | head -20
```

**Testar deploy com autenticação:**
```bash
PREFECT_API_URL=https://prefect3.basedosdados.org/api \
PREFECT_API_KEY=<token> \
uv run prefect work-pool ls
```

---

## 10. CI/CD — Configurar GitHub Secrets

Antes de dar push da branch `feat/prefect3`, adicionar em `basedosdados/pipelines` → Settings → Secrets:

| Secret | Descrição |
|---|---|
| `GCP_PROJECT_ID` | `basedosdados-dev` |
| `GCP_SA_KEY` | JSON da SA com permissão de push no GCR |
| `PREFECT3_AUTH_TOKEN` | Token Django permanente |
| `PREFECT3_API_URL` | `https://prefect3.basedosdados.org/api` |

---

## 11. Primeiro build da imagem Docker

Após configurar os secrets, o push da branch dispara o workflow automaticamente.
Para forçar um build manual sem mudar as dependências, pode fazer um commit vazio:

```bash
git commit --allow-empty -m "chore: trigger docker build"
git push origin feat/prefect3
```

Acompanhar em: `https://github.com/basedosdados/pipelines/actions`

---

## 12. Deploy manual de um flow (sem CI/CD)

Para testar um flow específico sem passar pelo CI/CD:

```bash
cd /mnt/d/repositorios/bd/pipelines
```

```bash
PREFECT_API_URL=https://prefect3.basedosdados.org/api \
PREFECT_API_KEY=<token> \
uv run python .github/scripts/deploy_flows.py \
  --pool basedosdados-dev \
  --branch feat/prefect3 \
  --files pipelines/datasets/meu_dataset/flows.py
```

Para deploy de todos os flows:

```bash
PREFECT_API_URL=https://prefect3.basedosdados.org/api \
PREFECT_API_KEY=<token> \
uv run python .github/scripts/deploy_flows.py \
  --pool basedosdados \
  --branch main \
  --all
```

---

## 13. Verificar compatibilidade de dependências com Python 3.12

Antes de fazer push de mudanças no `pyproject.toml`, testar localmente se todas as dependências têm wheel para Python 3.12:

```bash
cd /mnt/d/repositorios/bd/pipelines
uv sync --locked --no-dev --no-install-project --python 3.12
```

Se houver erro de compilação (ex: `longintrepr.h`, `Cython.Compiler`, `pkg_resources`), o pacote não tem wheel para Python 3.12 e precisa ser atualizado para uma versão mais recente no `pyproject.toml`.

**Pacotes que precisaram ser atualizados na migração:**

| Pacote | Versão original | Versão mínima py312 | Motivo |
|---|---|---|---|
| `PyYAML` | 6.0 | 6.0.1 | Cython build error |
| `lxml` | 4.9.2 | 4.9.4 | Sem wheel cp312 |
| `ruamel-yaml-clib` | 0.2.6 | 0.2.8 | `longintrepr.h` removido no py312 |
| `pymssql` | 2.2.5 | 2.3.0 | Cython incompatível |
| `shapely` | 2.0.1 | 2.0.2 | `pkg_resources` build error |
| `grpcio` | 1.56.2 | 1.59.0 | Sem wheel cp312 |

---

## 14. Disparar e acompanhar um flow run via API

**Listar deployments disponíveis:**

```bash
curl -s -X POST https://prefect3.basedosdados.org/api/deployments/filter \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{}' | python3 -c "import sys,json; d=json.load(sys.stdin); [print(x['id'], x['name']) for x in d]"
```

**Disparar um flow run:**

```bash
curl -s -X POST https://prefect3.basedosdados.org/api/deployments/<deployment-id>/create_flow_run \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{}' | python3 -c "import sys,json; r=json.load(sys.stdin); print('Run ID:', r['id'], '\nState:', r['state']['name'])"
```

**Acompanhar o estado do run:**

```bash
curl -s https://prefect3.basedosdados.org/api/flow_runs/<run-id> \
  -H "Authorization: Bearer <token>" | python3 -c "import sys,json; r=json.load(sys.stdin); print(r['state']['name'])"
```
