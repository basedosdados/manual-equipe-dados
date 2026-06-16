---
tipo: como-fazer
titulo: Como fazer deploy de um flow no Prefect 3
---
# Como fazer deploy de um flow no Prefect 3

Cobre a autenticação externa, o CI/CD do repositório `pipelines`, o deploy manual de um
flow e o disparo/monitoramento de um run via API REST.

## Pré-requisitos

- Servidor e workers do Prefect 3 no ar (ver [instalar o servidor](instalar-servidor-prefect-3.md)
  e [configurar os workers](configurar-workers.md)).
- Repositório `bd/pipelines` clonado, com `uv` instalado.
- Um token de autenticação Django (ver passo 1).

## Passos

### 1. Autenticação externa via token Django

O Prefect 3 self-hosted não tem API keys nativas. O nginx valida um token Django:
o cliente envia `Authorization: Bearer <token>`, o nginx valida em
`https://backend.basedosdados.org/auth/` e repassa a requisição para o Prefect.

Para criar um token permanente:

1. Acesse `https://backend.basedosdados.org/admin/authtoken/token/`.
2. Crie um token para o usuário de serviço.
3. Use-o como `PREFECT_API_KEY` (o Prefect o envia automaticamente como Bearer).

Testar:

```bash
curl -s https://prefect3.basedosdados.org/api/work_pools/ \
  -H "Authorization: Bearer <token>" | python3 -m json.tool | head -20

PREFECT_API_URL=https://prefect3.basedosdados.org/api \
PREFECT_API_KEY=<token> \
uv run prefect work-pool ls
```

### 2. Configurar GitHub Secrets (CI/CD)

Antes do push da branch, em `basedosdados/pipelines` → Settings → Secrets:

| Secret | Descrição |
|---|---|
| `GCP_PROJECT_ID` | `basedosdados-dev` |
| `GCP_SA_KEY` | JSON da SA com permissão de push no GCR |
| `PREFECT3_AUTH_TOKEN` | Token Django permanente |
| `PREFECT3_API_URL` | `https://prefect3.basedosdados.org/api` |

### 3. Disparar o build da imagem Docker

O push da branch dispara o workflow automaticamente. Para forçar um build sem mudar
dependências, faça um commit vazio:

```bash
git commit --allow-empty -m "chore: trigger docker build"
git push origin <branch>
```

Acompanhe em `https://github.com/basedosdados/pipelines/actions`.

### 4. Deploy manual de um flow (sem CI/CD)

```bash
cd /path/to/pipelines
PREFECT_API_URL=https://prefect3.basedosdados.org/api \
PREFECT_API_KEY=<token> \
uv run python .github/scripts/deploy_flows.py \
  --pool basedosdados-dev \
  --branch <branch> \
  --files pipelines/datasets/meu_dataset/flows.py
```

Para todos os flows: trocar `--files ...` por `--all` e usar `--pool basedosdados --branch main`.

### 5. Disparar e acompanhar um flow run via API

```bash
# Listar deployments
curl -s -X POST https://prefect3.basedosdados.org/api/deployments/filter \
  -H "Authorization: Bearer <token>" -H "Content-Type: application/json" \
  -d '{}' | python3 -c "import sys,json; [print(x['id'], x['name']) for x in json.load(sys.stdin)]"

# Disparar um run
curl -s -X POST https://prefect3.basedosdados.org/api/deployments/<deployment-id>/create_flow_run \
  -H "Authorization: Bearer <token>" -H "Content-Type: application/json" \
  -d '{}' | python3 -c "import sys,json; r=json.load(sys.stdin); print('Run ID:', r['id'], 'State:', r['state']['name'])"

# Acompanhar o estado
curl -s https://prefect3.basedosdados.org/api/flow_runs/<run-id> \
  -H "Authorization: Bearer <token>" | python3 -c "import sys,json; print(json.load(sys.stdin)['state']['name'])"
```

## Verificação

O deployment aparece em `deployments/filter` e o flow run alcança o estado `Completed`.
Na UI (`https://prefect3.basedosdados.org`), o run aparece sob o work pool correto.

## Problemas comuns

- **Dependência sem wheel para Python 3.12** — antes de dar push em mudanças no
  `pyproject.toml`, valide localmente:
  ```bash
  uv sync --locked --no-dev --no-install-project --python 3.12
  ```
  Erros como `longintrepr.h`, `Cython.Compiler` ou `pkg_resources` indicam pacote sem
  wheel cp312. Pacotes que precisaram subir de versão na migração:

  | Pacote | Original | Mínimo py312 |
  |---|---|---|
  | `PyYAML` | 6.0 | 6.0.1 |
  | `lxml` | 4.9.2 | 4.9.4 |
  | `ruamel-yaml-clib` | 0.2.6 | 0.2.8 |
  | `pymssql` | 2.2.5 | 2.3.0 |
  | `shapely` | 2.0.1 | 2.0.2 |
  | `grpcio` | 1.56.2 | 1.59.0 |
- **Flow sofre OOM (`OOMKilled`, exit 137)** — ver [Como ajustar recursos de pod](ajustar-recursos-de-pod.md).

## Ver também

- [Como configurar os workers](configurar-workers.md)
- [Como ajustar recursos de pod](ajustar-recursos-de-pod.md)
- [Workers (substitutos dos Agents)](../explicacao/workers.md)
