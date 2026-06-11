# Prefect 3 — Migração no GKE

## Contexto

O repositório `bd/iac` gerencia a infraestrutura GCP/GKE da Base dos Dados.
Objetivo: migrar a orquestração de workflows de **Prefect 0.15.9** (Prefect 1.x) para **Prefect 3**.

---

## Stack atual (Prefect 0.15.9)

| Componente   | Detalhe                                                            |
| ------------ | ------------------------------------------------------------------ |
| Namespace    | `prefect`                                                          |
| Helm chart   | local em `k8s/prefect/chart/`                                      |
| Banco        | Cloud SQL PostgreSQL 13, database `prefect`, user `prefect`        |
| URL          | `prefect.basedosdados.org`                                         |
| Componentes  | Server + Hasura (v2.8.1) + Apollo Gateway + Towel (scheduler) + UI |
| Agents       | 3x Kubernetes + 1x Vertex AI (`southamerica-east1`)                |
| Job template | `gs://basedosdados-dev/prefect_job_template/template.yaml`         |

---

## O que muda no Prefect 3

| Aspecto | Prefect 0.15.9 | Prefect 3 |
|---|---|---|
| API | GraphQL (Hasura + Apollo) | REST API (FastAPI) |
| Componentes | Server + Hasura + Apollo + Towel + UI | Apenas `prefect-server` |
| Agents | `KubernetesAgent` / `VertexAIAgent` | Workers com Work Pools |
| Job template | YAML no GCS | Configurado no Work Pool |
| Auth | Sem auth nativo | API Keys nativo |
| DB Schema | Incompatível — requer banco novo | Schema próprio Prefect 3 |
| Helm Chart | Chart local customizado | Chart oficial `prefecthq/prefect-helm` |

---

## Estratégia: novo namespace em paralelo

Para evitar downtime e riscos, a migração é feita em um **novo namespace `prefect3`**,
sem derrubar o namespace `prefect` atual até o cutover final.

---

## Progresso

### ✅ Fase 1 — Database (Terraform)

Branch: `feat/prefect3-database`

**Arquivos alterados:**
- `terraform/cloud_sql/main.tf` — adicionados recursos:
  - `random_password.prefect3_db_password` — gera senha automaticamente
  - `google_secret_manager_secret.prefect3_db_password` — armazena no Secret Manager GCP (nome: `prefect3-db-password`)
  - `google_sql_user.prefect3` — usuário `prefect3` no Cloud SQL
  - `google_sql_database.prefect3` — database `prefect3`
- `terraform/cloud_sql/variables.tf` — variáveis `sql_prefect3_user_name` e `sql_prefect3_db_name`
- `terraform/variables.tf` — mesmas variáveis na raiz
- `terraform/main.tf` — repassa variáveis ao módulo `cloudsql`

**Observação:** Senha gerada automaticamente pelo Terraform (sem Vault) e salva no GCP Secret Manager.
Recuperar com: `gcloud secrets versions access latest --secret=prefect3-db-password`

**Como aplicar localmente:**

```bash
export GOOGLE_APPLICATION_CREDENTIALS=/mnt/d/Downloads/terraform-sa.json
export TF_VAR_project_id=basedosdados-dev
export TF_VAR_bucket_name=terraform-data-basedosdados-dev
export TF_VAR_sql_prefect_user_password=fake
export TF_VAR_sql_metabase_user_password=fake
export TF_VAR_sql_id_server_user_password=fake

cd terraform
terraform init

terraform plan \
  -target=random_password.prefect3_db_password \
  -target=google_secret_manager_secret.prefect3_db_password \
  -target=google_secret_manager_secret_version.prefect3_db_password \
  -target=google_sql_user.prefect3 \
  -target=google_sql_database.prefect3

terraform apply \
  -target=random_password.prefect3_db_password \
  -target=google_secret_manager_secret.prefect3_db_password \
  -target=google_secret_manager_secret_version.prefect3_db_password \
  -target=google_sql_user.prefect3 \
  -target=google_sql_database.prefect3
```

---

### ✅ Fase 2 — Namespace K8s

**Arquivos criados em `k8s/prefect3/`:**

| Arquivo | O que faz |
|---|---|
| `namespace.yaml` | Cria o namespace `prefect3` |
| `db-service.yaml` | ExternalName Service → `cloud-sql-proxy.cloud-sql-proxy.svc.cluster.local:5432` |
| `issuer.yaml` | Issuer cert-manager Let's Encrypt (email: luiz.jordao@basedosdados.org) |
| `ingress.yaml` | NGINX ingress `prefect3.basedosdados.org` → `prefect-server:4200` |

**Diferenças em relação ao ingress antigo:**
- Único path `/` (no Prefect 3, UI e API REST são servidos pelo mesmo processo)
- Sem `rewrite-target`

**Como aplicar:**

```bash
kubectl apply -f k8s/prefect3/namespace.yaml
kubectl apply -f k8s/prefect3/
```

---

### ✅ Fase 3 — Helm chart do servidor

> Para entender o que é um chart e um values.yaml, ver **Conceitos Helm.md**

Chart oficial: `prefecthq/prefect-server` — `https://prefecthq.github.io/prefect-helm`
Arquivo: `k8s/prefect3/chart/values.yaml`

**Decisões de configuração:**

| Parâmetro                         | Valor                                   | Motivo                                                                |
| --------------------------------- | --------------------------------------- | --------------------------------------------------------------------- |
| `fullnameOverride`                | `prefect-server`                        | Service fica com nome fixo, referenciado no `ingress.yaml`            |
| `postgresql.enabled`              | `false`                                 | Usa Cloud SQL externo via `cloud-sql-proxy`                           |
| `redis.enabled`                   | `false`                                 | Modo single-deployment (background services rodam junto com o server) |
| `ingress.enabled`                 | `false`                                 | Ingress gerenciado pelo `ingress.yaml` separado                       |
| `secret.create`                   | `false`                                 | Connection string vem de sealed secret próprio                        |
| `secret.name`                     | `prefect3-db-connection`                | Nome do sealed secret com a connection string                         |
| `server.uiConfig.prefectUiApiUrl` | `https://prefect3.basedosdados.org/api` | URL pública da API para o UI                                          |
| `server.resources`                | 100m/128Mi req, 500m/256Mi lim          | Mínimo para ambiente de teste                                         |

**⚠️ Pendente: sealed secret do banco**

O chart espera um secret chamado `prefect3-db-connection` com a chave `connection-string`:
```
postgresql+asyncpg://prefect3:<senha>@cloud-sql-proxy:5432/prefect3
```

Passos para criar o sealed secret **após o terraform apply**:
```bash
# 1. Pegar a senha gerada pelo Terraform no Secret Manager
gcloud secrets versions access latest --secret=prefect3-db-password

# 2. Criar e selar o secret
kubectl create secret generic prefect3-db-connection \
  --namespace prefect3 \
  --from-literal=connection-string="postgresql+asyncpg://prefect3:<senha>@cloud-sql-proxy:5432/prefect3" \
  --dry-run=client -o yaml | \
  kubeseal --namespace prefect3 --format yaml > k8s/prefect3/secret-00_sealed.yaml

# 3. Aplicar
kubectl apply -f k8s/prefect3/secret-00_sealed.yaml
```

**Como instalar o chart:**
```bash
helm repo add prefecthq https://prefecthq.github.io/prefect-helm
helm repo update
helm upgrade --install prefect-server -n prefect3 prefecthq/prefect-server -f k8s/prefect3/chart/values.yaml
```

---

### ✅ Fase 4 — Deploy do servidor (concluído)

**Resultado:** Prefect 3 server rodando no namespace `prefect3` com ingress em `prefect3.basedosdados.org`.

**Problemas encontrados e resolvidos:**

1. **`-target` sem prefixo de módulo** — os recursos ficam dentro de `module.cloudsql`, então o target correto é `module.cloudsql.google_sql_database.prefect3` e não `google_sql_database.prefect3`

2. **Permissão negada no Secret Manager** — nem o usuário pessoal nem a SA do Terraform tinham `secretmanager.versions.access`. Solução: extrair a senha diretamente do Terraform state:
   ```bash
   terraform state pull | python3 -c "import sys,json; state=json.load(sys.stdin); [print(r['instances'][0]['attributes']['result']) for r in state['resources'] if r.get('module') == 'module.cloudsql' and r['type'] == 'random_password' and r['name'] == 'prefect3_db_password']"
   ```

3. **`Invalid IPv6 URL` nos logs** — a senha gerada continha `[` ou `]` (do `override_special`), que quebra o parser de URL do Python. Solução: URL-encode a senha antes de montar a connection string:
   ```bash
   python3 -c "import urllib.parse; print(urllib.parse.quote('<senha>', safe=''))"
   ```
   Se o resultado contiver `%5B` ou `%5D`, usar o valor encoded na connection string.

4. **Annotation deprecated no ingress** — `kubernetes.io/ingress.class: nginx` substituído por `spec.ingressClassName: nginx`

---

### ✅ Fase 5 — Workers (substitutos dos Agents) — CONCLUÍDO

**Resolvido:**
- [x] DNS `prefect3.basedosdados.org → 35.225.94.47` criado
- [x] Domínio adicionado na tabela `Domain` do backend Django (necessário para redirect de autenticação)
- [x] UI acessível em `https://prefect3.basedosdados.org` ✅

**✅ Decisão tomada com o Pisa:** consolidar em 2 workers (prod e dev), sem Vertex AI por ora.

| Worker | Namespace K8s | Work Pool | Status |
|---|---|---|---|
| `basedosdados` (prod) | `prefect-worker-basedosdados` | `basedosdados` | ✅ Online |
| `basedosdados-dev` (dev) | `prefect-worker-basedosdados-dev` | `basedosdados-dev` | ✅ Online |

**Problema encontrado durante instalação:** a imagem `3-latest` não inclui o pacote `prefect-kubernetes`. A tag correta para workers Kubernetes é `3-python3.12-kubernetes`.

**Para retomar:** ver **Prefect 3 Workers.md** com todos os passos detalhados.

**Acesso local à UI (alternativa sem depender do DNS/auth):**
```bash
kubectl --namespace prefect3 port-forward svc/prefect-server 4200:4200
```
Acessa em `http://localhost:4200`

**Branch do IAC:** `feat/prefect3-database` em `basedosdados/iac`
**Issue de acompanhamento:** basedosdados/pipelines#1506

---

### 🔲 Fase 6 — Cutover (após workers e flows migrados)

1. Migrar os flows Python para usar `@flow` / `@task` do Prefect 3
2. Trocar DNS `prefect.basedosdados.org` para apontar para o novo server
3. Desativar namespace `prefect` antigo

---

## Referências

- [Prefect 3 Helm Chart](https://github.com/PrefectHQ/prefect-helm)
- [Prefect 3 Docs — Kubernetes Worker](https://docs.prefect.io/latest/deploy/infrastructure-concepts/workers/)
- [Prefect 3 Docs — Vertex AI Worker](https://docs.prefect.io/latest/deploy/infrastructure-examples/serverless/)
