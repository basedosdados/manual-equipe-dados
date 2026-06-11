# Prefect 3 — Etapas de Migração de Flows

> Runbook para agentes (humanos ou LLM) que vão migrar um flow Prefect 0 → Prefect 3.
> Para o **como técnico**, ver `prefect3-diretrizes-migracao.md`.
> Para o **status atual** dos flows, ver `prefect3-flows-migrados.md`.

---

## Princípios

1. **Um dataset por iteração.** Não migrar dois datasets ao mesto tempo no mesmo PR. Cada dataset deve ser migrado sequencialmente no mesmo PR; Isto significa que um novo dataset só pode ser migrado quanto todas as etapas do @prefect-flows-migrados.md forem executadas.
2. **Diretriz é lei.** Em conflito entre o flow antigo e o `prefect3-diretrizes-migracao.md`, a diretriz vence. Refatore o flow para se adequar — não invente novo padrão.
3. **Dev antes de prod.** Nenhum flow vai para o pool `basedosdados` sem ter rodado `Completed` no pool `basedosdados-dev` com pelo menos uma execução real.
4. **Tracking obrigatório.** Toda mudança de status (deploy / dev OK / prod OK / bloqueio) precisa ser refletida no `prefect3-flows-migrados.md` na mesma PR.
5. **Sem CI/CD-driven deploy nesta fase.** O deploy é feito manualmente via `uv run python .github/scripts/deploy_flows.py`. Mergear na `main` ainda não dispara redeploy automático.

---

## Pré-requisitos do agente

Antes de abrir a primeira iteração:

- [ ] Ler `prefect3-diretrizes-migracao.md` na íntegra (utils, autenticação, template canônico, armadilhas).
- [ ] Ler `prefect3-flows-migrados.md` para entender o estado e os flows-referência (`br_camara_dados_abertos__deputado`, `br_ibge_ipca__mes_brasil`).
- [ ] Ter `.env` da repo carregada (`PREFECT_API_URL`, `PREFECT_API_KEY`).
- [ ] Conseguir abrir https://prefect3.basedosdados.org e listar deployments via `uv run prefect deployment ls`.

---

## Etapas por dataset_id

Cada iteração de migração é **um dataset_id** e segue **4 etapas obrigatórias**, em ordem. Não pule.

### Etapa 1 — Refatorar o flow

**Objetivo:** o flow do dataset roda em Prefect 3 seguindo o template canônico.

Checklist:

- [ ] Identificar o(s) flow(s) Prefect 0 no diretório `pipelines/datasets/<dataset_id>/flows.py`.
- [ ] Mapear cada construto Prefect 0 → Prefect 3 usando a tabela de conversão (§3 das diretrizes).
- [ ] Para datasets com **múltiplas tabelas e schedules independentes**, usar o padrão **factory** (`_run_<dataset>` compartilhado em `pipelines/crawler/<crawler>/flows.py` + wrappers `@flow` por tabela no `datasets/<dataset_id>/flows.py`). Nunca usar `deepcopy`.
- [ ] Toda chamada `task.run(...)` em utils de crawler vira `task.fn(...)`.
- [ ] Toda assinatura de flow expõe os parâmetros padrão: `dataset_id`, `table_id`, `materialize_after_dump`, `dbt_alias`, `update_metadata`, `target`, **`force_run`**.
- [ ] O bloco `check_if_data_is_outdated` está envolvido em `if not force_run:` (ver §4 das diretrizes — padrão atual, verboso, revisável depois).
- [ ] Schedules cron expostos via `flow.deploy_schedules = [{"cron": "...", "timezone": "America/Sao_Paulo"}]`.
- [ ] `on_failure=[partial(notify_discord_on_failure, secret_path="discord_webhook")]` no decorator do `@flow`.
- [ ] Imports limpos — nunca `from pipelines.datasets import *` (quebra por causa do `__init__.py`).
- [ ] Smoke import local: `uv run python -c "from pipelines.datasets.<dataset_id>.flows import *"` retorna sem erro.

**Saída:** flow refatorado + commit local. **Não fazer push ainda.**

---

### Etapa 2 — Commitar para a branch feat/prefect3-flows-migration

**Objetivo:** o código vive numa branch remota que o Prefect worker pode clonar via `GitRepository`.

Checklist:


- [ ] Commit message no padrão convencional: `feat(<dataset_id>): migrate flow to prefect 3`.

**Saída:** COmmit enviado para a branch pushada para `origin`.

---

### Etapa 3 — Realizar deployment dos flows após registro

**Objetivo:** os flows aparecem como deployments aplicáveis no pool `basedosdados-dev`.

Comando:

```bash
uv run python .github/scripts/deploy_flows.py \
  --pool basedosdados-dev \
  --branch <branch-do-PR> \
  --files pipelines/datasets/<dataset_id>/flows.py
```

Checklist:

- [ ] Output do script termina com `Resultado: N registrados, 0 pulados, 0 com erro`.
- [ ] `uv run prefect deployment ls | grep <dataset_id>` lista todos os deployments esperados.
- [ ] Para cada deployment, abrir a URL retornada e confirmar que **os parâmetros aparecem com defaults corretos** (especialmente `force_run=false`).
- [ ] **Se a assinatura mudou (novo param) num flow já deployado, redeployar.** O schema do deployment na UI só atualiza com novo `deploy_flows.py`.

**Saída:** deployments aplicados, prontos para teste em dev.

---

### Etapa 4 — Testar no ambiente de dev

**Objetivo:** pelo menos um run end-to-end `Completed` em dev, escrevendo em `basedosdados-dev`.

Dois tipos de smoke test, **ambos obrigatórios**:

**4a. Run padrão (idempotente)** — valida o fluxo cron normal:

```bash
uv run prefect deployment run \
  '<flow_name>/<flow_name>' \
  -p target=dev \
  -p materialize_after_dump=false \
  -p update_metadata=false
```

Resultado esperado: ou roda end-to-end e termina `Completed`, ou pula cedo via `check_if_data_is_outdated` (não há dado novo) — ambos contam como sucesso do smoke test do guard.

**4b. Run forçado** — valida o `force_run`:

```bash
uv run prefect deployment run \
  '<flow_name>/<flow_name>' \
  -p target=dev \
  -p materialize_after_dump=false \
  -p update_metadata=false \
  -p force_run=true
```

Resultado esperado: o flow executa ignorando `check_if_data_is_outdated`, escreve em `basedosdados-dev.<dataset_id>_staging.<table_id>`, termina `Completed`.

Checklist:

- [ ] Run 4a termina `Completed` (ou `Completed` com early-return por idempotência) **ou** falha somente na task `run_dbt` após `upload_to_gcs` ter escrito em `basedosdados-dev` com sucesso.
- [ ] Run 4b termina `Completed` e a tabela em `basedosdados-dev` recebe escrita (ou falha somente em `run_dbt` após upload OK).
- [ ] Sem erro de auth no Discord / Vault / Django backend.
- [ ] Artefatos dbt 403 são não-bloqueantes (ver §6.3 das diretrizes).
- [ ] Atualizar `prefect3-flows-migrados.md`:
  - Coluna "Dev" → ✅ + data.
  - Status → 🟢 Dev OK.

> **Falha em `run_dbt` ≠ falha de migração.** Se o upload para `basedosdados-dev.<dataset>_staging.<table>` foi bem-sucedido e a task que falhou é `run_dbt`, considere a migração validada: o problema é do modelo dbt / dos dados (pré-existente) e não da conversão Prefect 0 → Prefect 3. Marque "Dev" como ✅ com observação `Upload p/ staging OK; falha no dbt run não relacionada à migração`. Não investigar nem corrigir o modelo dbt dentro do escopo desta migração.

**Saída:** flow validado em dev.

---


## Como anotar no `prefect3-flows-migrados.md`

A cada transição de estado, **atualizar o tracking no mesmo PR** que produziu a transição:

| Transição | O que atualizar |
|---|---|
| Refatorou + abriu PR | Adicionar linha ao tracking com status 🟡 Pronto p/ deploy |
| Deploy dev OK | Coluna "Deployado" → ✅ + data |
| Run dev `Completed` | Coluna "Dev" → ✅ + data, status → 🟢 Dev OK |
| Deploy prod OK | (sem coluna específica — anotar em observações) |
| Run prod `Completed` | Coluna "Prod" → ✅ + data, status → ✅ Migrado |
| Falha estrutural | Status → ⚠️ Bloqueado + motivo na coluna "Observações" |

**Mudança de assinatura em flow já deployado:**
- Marcar `♻️ <data>` na coluna "Deployado" (até o redeploy).
- Limpar coluna "Dev" se a validação anterior não cobre a nova assinatura.
- Re-rodar a etapa 3 e a etapa 4.

---

## Quando parar e pedir ajuda

- Run em dev falha por motivo estrutural (auth, secret, schema BigQuery) → não silenciar; abrir issue / mensagem e marcar ⚠️ Bloqueado.
- O flow Prefect 0 original tem >500 linhas e/ou usa construções não cobertas nas diretrizes → escalar antes de tentar reescrever. Documentar o motivo no tracking.
- Discrepância entre o que está no `prefect3-diretrizes-migracao.md` e o que parece ser o "jeito certo" no flow atual → escalar para humano; **não criar terceiro padrão**.

---

## Referência rápida de documentos

| Documento | Propósito |
|---|---|
| `prefect3-etapas-migracao-flows.md` (este) | **Como** migrar — runbook de etapas |
| `prefect3-diretrizes-migracao.md` | **O que** mudar — referência técnica (utils, template, armadilhas) |
| `prefect3-flows-migrados.md` | **Estado** — tracking de status por flow |
| `Dockerfile.prefect3` | Imagem usada pelos workers |
