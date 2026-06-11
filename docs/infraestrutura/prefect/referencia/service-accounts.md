# Contas de Serviço GCP — Prefect (0 e 3)

Mapeamento completo das service accounts usadas nos workers/agents do Prefect, como chegam no pod e para que servem.

---

## Permissões por Service Account

### `dbt-rpc@basedosdados`

| Projeto                | Roles                                   | GCS                                         | BigQuery         |
| ---------------------- | --------------------------------------- | ------------------------------------------- | ---------------- |
| `basedosdados`         | `roles/editor` + `roles/bigquery.admin` | ✅ leitura e escrita (bucket `basedosdados`) | ✅ admin completo |
| `basedosdados-dev`     | `roles/bigquery.admin`                  | ❌ sem acesso                                | ✅ admin completo |
| `basedosdados-staging` | —                                       | ❌                                           | ❌                |

**Uso:** exclusiva para rodar dbt — tem BigQuery admin nos dois projetos. Em prod também tem GCS via `roles/editor`, mas em dev **não tem acesso ao bucket `basedosdados-dev`**. Por isso não substitui as SAs `prefect@` para uploads.

---

### `dbt-rpc@basedosdados-dev`

| Projeto | Roles | GCS | BigQuery |
|---|---|---|---|
| `basedosdados-dev` | `roles/bigquery.admin` | ❌ sem acesso | ✅ admin completo |
| `basedosdados` | — | ❌ | ❌ |

**Uso:** equivalente dev da `dbt-rpc@basedosdados` — apenas BigQuery no projeto `basedosdados-dev`.

---

### `prefect@basedosdados`

| Projeto | Roles | GCS | BigQuery |
|---|---|---|---|
| `basedosdados` | `CustomRole` + `roles/editor` | ✅ leitura e escrita (bucket `basedosdados`) | ✅ |
| `basedosdados-dev` | `CustomRole` | ❌ sem GCS | ✅ BigQuery apenas |
| `basedosdados-staging` | `roles/editor` | ✅ leitura e escrita (bucket `basedosdados-staging`) | ✅ |

> `CustomRole` em `basedosdados` contém apenas `bigquery.rowAccessPolicies.setIamPolicy`.
> `CustomRole` em `basedosdados-dev` contém permissões BigQuery avançadas (datasets, jobs, connections, etc.).

**Uso:** SA principal da lib `basedosdados` em prod — upload GCS ao bucket `basedosdados` e acesso BigQuery.

---

### `prefect@basedosdados-dev`

| Projeto | Roles | GCS | BigQuery |
|---|---|---|---|
| `basedosdados-dev` | `CustomRole` + `roles/editor` + `roles/storage.admin` + `roles/ml.developer` | ✅ admin completo (bucket `basedosdados-dev`) | ✅ |
| `basedosdados` | — | ❌ | ❌ |

**Uso:** SA principal da lib `basedosdados` em dev — acesso completo ao projeto `basedosdados-dev` incluindo GCS, BigQuery e Vertex AI.

---

### `prefect@basedosdados-staging`

| Projeto | Roles | GCS | BigQuery |
|---|---|---|---|
| `basedosdados` | `CustomRole` + `roles/editor` | ✅ leitura e escrita (bucket `basedosdados`) | ✅ |
| `basedosdados-dev` | `CustomRole` | ❌ sem GCS | ✅ BigQuery apenas |
| `basedosdados-staging` | `roles/editor` | ✅ leitura e escrita (bucket `basedosdados-staging`) | ✅ |

**Uso:** SA da lib `basedosdados` para o ambiente staging — acesso aos buckets `basedosdados` e `basedosdados-staging`.

---

## Por que existem SAs separadas para dbt e prefect?

A lib `basedosdados` faz **upload de arquivos ao GCS** antes de o dbt rodar. As SAs `prefect@` têm acesso ao Storage (`roles/editor` ou `roles/storage.admin`). As SAs `dbt-rpc@` têm apenas BigQuery — foram criadas com escopo mínimo para rodar modelos dbt sem permissão de escrita no Storage.

Fluxo típico de um flow:
1. Download do dado bruto → sem credencial GCP
2. Upload para GCS → `prefect@` (via lib `basedosdados`, lê `BASEDOSDADOS_CREDENTIALS_STAGING/PROD`)
3. dbt run/test → `dbt-rpc@` (lê `/credentials-dev/dev.json` ou `/credentials-prod/prod.json`)

---

## Prefect 0 — namespace `prefect-agent-basedosdados` (prod)

O job template (`k8s/prefect/job_template/job_template.yaml`) injeta **5 secrets** em cada pod de flow.

### 1. `gcp-credentials` → env vars (envFrom)

| Variável | Service Account | Projeto GCP | Quem usa |
|---|---|---|---|
| `BASEDOSDADOS_CREDENTIALS_STAGING` | `prefect@basedosdados-staging.iam.gserviceaccount.com` | `basedosdados-staging` | lib `basedosdados` — acesso ao bucket/BQ de staging |
| `BASEDOSDADOS_CREDENTIALS_PROD` | `prefect@basedosdados.iam.gserviceaccount.com` | `basedosdados` | lib `basedosdados` — acesso ao bucket/BQ de prod |
| `BASEDOSDADOS_CONFIG` | *(não é SA — é o `config.toml`)* | — | lib `basedosdados` — define `bucket_name=basedosdados` e paths das credenciais |

> As duas credenciais chegam como JSON **base64-encoded**. A lib `basedosdados` decodifica internamente.

---

### 2. `gcp-sa` → arquivo `/mnt/creds.json` + `GOOGLE_APPLICATION_CREDENTIALS`

| Arquivo | Service Account | Projeto GCP | Quem usa |
|---|---|---|---|
| `/mnt/creds.json` | `prefect@basedosdados.iam.gserviceaccount.com` | `basedosdados` | ADC (Application Default Credentials) — libs Python que chamam a API Google sem credencial explícita (ex.: `google-cloud-storage`, `bigquery` direto) |

> É a **mesma SA** de `BASEDOSDADOS_CREDENTIALS_PROD`. Existe separada porque algumas libs lêem `GOOGLE_APPLICATION_CREDENTIALS` em vez da variável da lib BD.

---

### 3. `credentials-dev` → arquivo `/credentials-dev/dev.json`

| Arquivo | Service Account | Projeto GCP | Quem usa |
|---|---|---|---|
| `/credentials-dev/dev.json` | `dbt-rpc@basedosdados.iam.gserviceaccount.com` | `basedosdados` | dbt com `--target dev` — lido via `profiles.yml` |

> ⚠️ **Atenção:** usa a SA do projeto **prod** (`basedosdados`), não do dev. O "target dev" do dbt neste agent aponta para um schema dev dentro do projeto de prod, não para o projeto `basedosdados-dev`.

---

### 4. `credentials-prod` → arquivo `/credentials-prod/prod.json`

| Arquivo | Service Account | Projeto GCP | Quem usa |
|---|---|---|---|
| `/credentials-prod/prod.json` | `dbt-rpc@basedosdados.iam.gserviceaccount.com` | `basedosdados` | dbt com `--target prod` — lido via `profiles.yml` |

> Mesma SA que `credentials-dev` neste namespace — ambas são `dbt-rpc@basedosdados`.

---

### 5. `vault-credentials` → env vars (envFrom)

| Variável | Valor | Quem usa |
|---|---|---|
| `VAULT_ADDRESS` | `http://vault.vault.svc.cluster.local:8200/` | Flows que lêem segredos do Vault |
| `VAULT_TOKEN` | token de acesso | Idem |

---

## Prefect 0 — namespace `prefect-agent-basedosdados-dev-gcp-vertex` (dev)

Mesma estrutura de 5 secrets, porém todas as SAs são do projeto **`basedosdados-dev`**.

| Secret | Arquivo/Variável | Service Account | Projeto GCP |
|---|---|---|---|
| `gcp-credentials` | `BASEDOSDADOS_CREDENTIALS_STAGING` | `prefect@basedosdados-dev.iam.gserviceaccount.com` | `basedosdados-dev` |
| `gcp-credentials` | `BASEDOSDADOS_CREDENTIALS_PROD` | `prefect@basedosdados-dev.iam.gserviceaccount.com` | `basedosdados-dev` |
| `gcp-sa` | `/mnt/creds.json` | `prefect@basedosdados-dev.iam.gserviceaccount.com` | `basedosdados-dev` |
| `credentials-dev` | `/credentials-dev/dev.json` | `dbt-rpc@basedosdados-dev.iam.gserviceaccount.com` | `basedosdados-dev` |
| `credentials-prod` | `/credentials-prod/prod.json` | `dbt-rpc@basedosdados-dev.iam.gserviceaccount.com` | `basedosdados-dev` |

---

## Prefect 3 — estado atual (2026-05-19)

Secret `gcp-credentials` por namespace, gerenciado via **Sealed Secrets**. O `entrypoint.sh` (embutido na imagem) decodifica as credenciais antes de iniciar o flow. `vault-credentials` e `gcp-sa` ainda não existem nos workers Prefect 3.

### `prefect-worker-basedosdados-dev` (worker dev)

| Variável | Service Account | Projeto GCP | Quem usa |
|---|---|---|---|
| `BASEDOSDADOS_CREDENTIALS_STAGING` | `prefect@basedosdados-dev.iam.gserviceaccount.com` | `basedosdados-dev` | lib `basedosdados` (env var) + dbt via `entrypoint.sh` → `/credentials-dev/dev.json` |
| `BASEDOSDADOS_CREDENTIALS_PROD` | `prefect@basedosdados-dev.iam.gserviceaccount.com` | `basedosdados-dev` | ⚠️ Temporário — mesma SA do STAGING |
| `BASEDOSDADOS_CONFIG` | *(config.toml)* | — | `bucket_name=basedosdados-dev` |

### `prefect-worker-basedosdados` (worker prod) ✅ validado

| Variável | Service Account | Projeto GCP | Quem usa |
|---|---|---|---|
| `BASEDOSDADOS_CREDENTIALS_STAGING` | `prefect@basedosdados-dev.iam.gserviceaccount.com` | `basedosdados-dev` | ⚠️ Workaround — dev SA para upload ao bucket `basedosdados-dev` durante testes |
| `BASEDOSDADOS_CREDENTIALS_PROD` | `prefect@basedosdados.iam.gserviceaccount.com` | `basedosdados` | lib `basedosdados` — upload GCS ao bucket `basedosdados` |
| `BASEDOSDADOS_CONFIG` | *(config.toml)* | — | `bucket_name=basedosdados` |
| `DBT_SERVICE_ACCOUNT` | `dbt-rpc@basedosdados.iam.gserviceaccount.com` | `basedosdados` | dbt — `entrypoint.sh` decodifica para `/credentials-dev/dev.json` e `/credentials-prod/prod.json` |
| `GOOGLE_APPLICATION_CREDENTIALS` | `/credentials-prod/prod.json` *(env var estática)* | — | ADC — libs Python que não usam credencial explícita |

---

## Como as credenciais chegam no pod — Prefect 3

O `entrypoint.sh` (copiado na imagem pelo `Dockerfile.prefect3`) decodifica as variáveis antes de iniciar o flow:

```sh
echo "$BASEDOSDADOS_CREDENTIALS_STAGING" | base64 -d > /credentials-dev/dev.json
echo "$BASEDOSDADOS_CREDENTIALS_PROD"    | base64 -d > /credentials-prod/prod.json
```

| Consumidor | Como usa |
|---|---|
| lib `basedosdados` | lê `BASEDOSDADOS_CREDENTIALS_STAGING` diretamente (base64) |
| dbt target=dev | lê `/credentials-dev/dev.json` via `profiles.yml` |
| dbt target=prod | lê `/credentials-prod/prod.json` via `profiles.yml` |

---

## Pendências de credenciais no Prefect 3

| # | O que falta | Prioridade |
|---|---|---|
| 1 | `gcp-sa` + `GOOGLE_APPLICATION_CREDENTIALS` em ambos workers | Alta — necessário para flows que usam ADC |
| 2 | `vault-credentials` em ambos workers | Média — necessário para flows que lêem segredos do Vault |
| 3 | SA de dbt separada para dbt nos workers (atualmente usa a SA da lib `basedosdados`) | Média — no Prefect 0 existe `dbt-rpc@` separada da `prefect@` |
| 4 | Trocar `BASEDOSDADOS_CREDENTIALS_PROD` no worker dev para SA de dev real | Baixa — workaround funcional |
| 5 | Trocar `BASEDOSDADOS_CREDENTIALS_STAGING` no worker prod para `prefect@basedosdados-staging` quando ambiente de testes for separado | Baixa — workaround funcional para testes |
