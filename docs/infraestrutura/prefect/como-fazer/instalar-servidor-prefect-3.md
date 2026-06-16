---
tipo: como-fazer
titulo: Como instalar o servidor Prefect 3 no GKE
---
# Como instalar o servidor Prefect 3 no GKE

Sobe o stack do `prefect-server` no namespace `prefect3`: banco no Cloud SQL,
manifests do namespace, sealed secret da connection string e chart Helm oficial. Para
o porquê da arquitetura, ver [Arquitetura do Prefect 3](../explicacao/arquitetura-prefect-3.md).

## Pré-requisitos

- Terraform 1.1.9, `kubectl` (apontando para o cluster GKE), `helm`, `kubeseal` e
  `gcloud` instalados e autenticados.
- Repositório `bd/iac` clonado (os comandos abaixo usam caminhos relativos a ele).

## Passos

### 1. Criar o banco de dados (Terraform)

```bash
cd terraform
source .env
terraform init

terraform plan \
  -target=module.cloudsql.random_password.prefect3_db_password \
  -target=module.cloudsql.google_secret_manager_secret.prefect3_db_password \
  -target=module.cloudsql.google_secret_manager_secret_version.prefect3_db_password \
  -target=module.cloudsql.google_sql_user.prefect3 \
  -target=module.cloudsql.google_sql_database.prefect3

terraform apply \
  -target=module.cloudsql.random_password.prefect3_db_password \
  -target=module.cloudsql.google_secret_manager_secret.prefect3_db_password \
  -target=module.cloudsql.google_secret_manager_secret_version.prefect3_db_password \
  -target=module.cloudsql.google_sql_user.prefect3 \
  -target=module.cloudsql.google_sql_database.prefect3
```

Isso cria o database e o usuário `prefect3` no Cloud SQL e gera a senha automaticamente,
salvando-a no GCP Secret Manager (sem Vault).

### 2. Pegar a senha gerada

```bash
gcloud secrets versions access latest --secret=prefect3-db-password
```

### 3. Criar o sealed secret da connection string

O chart espera o secret `prefect3-db-connection` com a chave `connection-string`. **Faça
URL-encode da senha** antes de montar a string (ver Problemas comuns):

```bash
kubectl create secret generic prefect3-db-connection \
  --namespace prefect3 \
  --from-literal=connection-string="postgresql+asyncpg://prefect3:<senha-encoded>@cloud-sql-proxy:5432/prefect3" \
  --dry-run=client -o yaml | \
  kubeseal --namespace prefect3 --format yaml > k8s/prefect3/secret-00_sealed.yaml
```

### 4. Aplicar os manifests do namespace

```bash
kubectl apply -f k8s/prefect3/namespace.yaml
kubectl apply -f k8s/prefect3/db-service.yaml
kubectl apply -f k8s/prefect3/issuer.yaml
kubectl apply -f k8s/prefect3/secret-00_sealed.yaml
kubectl apply -f k8s/prefect3/ingress.yaml
```

| Manifest | O que faz |
|---|---|
| `namespace.yaml` | Cria o namespace `prefect3` |
| `db-service.yaml` | ExternalName Service → `cloud-sql-proxy.cloud-sql-proxy.svc.cluster.local:5432` |
| `issuer.yaml` | Issuer cert-manager Let's Encrypt |
| `ingress.yaml` | NGINX ingress `prefect3.basedosdados.org` → `prefect-server:4200` |

> No Prefect 3, UI e API REST são servidas pelo mesmo processo: o ingress usa um único
> path `/` e **não** usa `rewrite-target`.

### 5. Instalar o chart do Prefect 3

```bash
helm repo add prefecthq https://prefecthq.github.io/prefect-helm
helm repo update
helm upgrade --install prefect-server -n prefect3 prefecthq/prefect-server \
  -f k8s/prefect3/chart/values.yaml
```

Decisões de configuração relevantes no `values.yaml`:

| Parâmetro                         | Valor                                   | Motivo                                                       |
| --------------------------------- | --------------------------------------- | ------------------------------------------------------------ |
| `fullnameOverride`                | `prefect-server`                        | Service com nome fixo, referenciado no `ingress.yaml`        |
| `postgresql.enabled`              | `false`                                 | Usa Cloud SQL externo via `cloud-sql-proxy`                  |
| `redis.enabled`                   | `false`                                 | Modo single-deployment                                       |
| `ingress.enabled`                 | `false`                                 | Ingress gerenciado pelo `ingress.yaml` separado             |
| `secret.create`                   | `false`                                 | Connection string vem do sealed secret próprio              |
| `secret.name`                     | `prefect3-db-connection`                | Nome do sealed secret com a connection string               |
| `server.uiConfig.prefectUiApiUrl` | `https://prefect3.basedosdados.org/api` | URL pública da API para o UI                                |

## Verificação

```bash
kubectl get pods -n prefect3
kubectl logs -n prefect3 -l app.kubernetes.io/name=prefect-server --tail=50
kubectl get ingress -n prefect3
```

A UI deve responder em `https://prefect3.basedosdados.org`. Sem DNS/auth, acesse via
port-forward:

```bash
kubectl --namespace prefect3 port-forward svc/prefect-server 4200:4200
# http://localhost:4200
```

## Problemas comuns

- **`-target` não encontra o recurso** — os recursos ficam dentro de `module.cloudsql`;
  use o prefixo completo (`module.cloudsql.google_sql_database.prefect3`).
- **Permissão negada no Secret Manager** — se nem a conta pessoal nem a SA do Terraform
  têm `secretmanager.versions.access`, extraia a senha do state:
  ```bash
  terraform state pull | python3 -c "import sys,json; state=json.load(sys.stdin); [print(r['instances'][0]['attributes']['result']) for r in state['resources'] if r.get('module')=='module.cloudsql' and r['type']=='random_password' and r['name']=='prefect3_db_password']"
  ```
- **`Invalid IPv6 URL` nos logs** — a senha contém `[` ou `]`, que quebram o parser de
  URL do Python. URL-encode antes de montar a connection string:
  ```bash
  python3 -c "import urllib.parse; print(urllib.parse.quote('<senha>', safe=''))"
  ```
  Se o resultado contiver `%5B`/`%5D`, use o valor encoded.
- **Annotation deprecada no ingress** — `kubernetes.io/ingress.class: nginx` foi
  substituída por `spec.ingressClassName: nginx`.

## Ver também

- [Como configurar os workers](configurar-workers.md)
- [Arquitetura do Prefect 3 no GKE](../explicacao/arquitetura-prefect-3.md)
- [Service accounts do Prefect](../referencia/service-accounts.md)
