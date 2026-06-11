# Plano de migração — flows → camada de metadata refatorada (`register_*`)

> Construído sobre as migrações já concluídas em [`prefect3-flows-migrados.md`](prefect3-flows-migrados.md)
> e o refactor de backend [`refatorar_backend.md`](refatorar_backend.md) (M1–M4b feitos).
> **Todos os flows já estão em Prefect 3.** Esta migração é a **próxima camada**: trocar as
> tasks legadas de metadata pelos dois orquestradores refatorados, usando `br_bcb_agencia`
> como template validado em produção.

---

## 1. O que esta migração é (e o que não é)

**É:** substituir, flow a flow, as três tasks legadas

| Legada | Substituída por |
|---|---|
| `check_if_data_is_outdated` | `register_source_poll_task` |
| `check_if_data_is_outdated_by_size` | `register_source_poll_task` (`source_max_date=None`) |
| `update_django_metadata` | `register_table_materialization_task` |

**Não é:** re-migração Prefect 0 → 3 (já feita), nem mudança de ETL/dbt. É só a borda
flow → backend de metadata.

**Por que agora:** o piloto `br_bcb_agencia` foi validado **end-to-end em produção** (poll +
materialização, com asserção contra oráculo legado e idempotência — runs `amber-saluki` e
`iron-ferret`), e pegou um bug real (`str→date`) que os mocks mascaravam, já corrigido
(`9f444682`). A prova de equivalência offline (type-8, `test_equivalence.py`) cobre os 4
templates de cobertura. A base está pronta para escalar.

---

## 2. Escopo: o que migrar, o que excluir

### 2.1 ✅ DELETADOS — deprecados removidos

Carregavam chamadas legadas mas estavam aposentados. **Removidos** (7 diretórios + imports em
`datasets/__init__.py`) para destravar a remoção das tasks legadas (M5). dbt config em
`dbt_project.yml` e o link estático no bot `botdosdados` foram **mantidos** (escopo é flows; a
página do dataset permanece publicada).

| Flow | Chamadas legadas | Status |
|---|---|---|
| `br_ons_avaliacao_operacao` | 7 | ✅ dir deletado |
| `br_ons_estimativa_custos` | 6 | ✅ dir deletado |
| `mundo_transfermarkt_competicoes` | 6 | ✅ dir deletado |
| `mundo_transfermarkt_competicoes_internacionais` | 4 | ✅ dir deletado |
| `br_mg_belohorizonte_smfa_iptu` | 4 | ✅ dir deletado |
| `br_mercadolivre_ofertas` | 3 (comentadas) | ✅ dir deletado |
| `br_b3_cotacoes` | 2 | ✅ dir deletado |

### 2.2 ⚠️ Fora de escopo — decisão à parte

| Flow | Situação | Resolução |
|---|---|---|
| `cross_update` | **Ainda em Prefect 0** (`prefect.storage`, `case`, `unmapped`); meta-flow que atualiza `nrows`/metadata via `update_django_metadata` | ⏳ **ÚNICO bloqueador restante do M5.** Decidir: aposentar ou reimplementar em P3 sobre `register_table_materialization`. |
| `crawler/world_sofascore_competicoes_futebol` | Ainda em Prefect 0 — não migrado p/ P3 | ✅ **DELETADO** (commit `0d7f9161`): deprecado junto com os demais (ambos os dirs `datasets/` + `crawler/` + import no `__init__`). |
| `utils/metadata/flows.py` (`update_temporal_coverage`) | Meta-flow genérico P0 que rodava `update_django_metadata` sob demanda | ✅ **REIMPLEMENTADO em P3** (commit `3468afbc`): `@flow` sobre `register_table_materialization_task` via `coverage_from_legacy_params`; mesma superfície de parâmetros. |

### 2.3 ✅ Migrar — 18 famílias ativas

Ver tabela mestre na §4.

---

## 3. Os dois padrões de migração

### Padrão A — poll + materialização (espelho do `br_bcb_agencia`)
Fonte tem `last_modified` ⇒ o flow chama `check_if_data_is_outdated` hoje. Migração:

```python
# topo do flow
from pipelines.utils.metadata.domain import DateFormat, PartBdpro, YearMonth  # ajustar spec
from pipelines.utils.metadata.register import (
    register_source_poll_task,
    register_table_materialization_task,
)

# no lugar de check_if_data_is_outdated(...)
is_outdated = register_source_poll_task(
    dataset_id=dataset_id, table_id=table_id,
    source_max_date=data_source_max_date,   # str crua da fonte
    env="prod",
    date_format="<formato da str da fonte>",  # ex.: "%Y-%m"  ← coagido a date no wrapper
)
if not is_outdated:
    return

# no lugar de update_django_metadata(...)
register_table_materialization_task(
    dataset_id=dataset_id, table_id=table_id,
    coverage=<CoverageSpec>,   # ver mapeamento §4
    env="prod", bq_project="basedosdados",
)
```

### Padrão B — só materialização (fonte sem `last_modified`)
Flow não chama `check_if_data_is_outdated` hoje (ex.: ZIP da CVM); só `update_django_metadata`.
Migração: **só** `register_table_materialization_task`. `force_run` permanece no-op (4b ≡ 4a).
Opcional: adicionar `register_source_poll_task(..., source_max_date=None)` para passar a gravar
`Poll` (R15) — melhoria, não exigida para paridade.

### Mapeamento `coverage_type` → `CoverageSpec`

| Legado | Novo |
|---|---|
| `part_bdpro` | `PartBdpro(date_column=…, date_format=…, free_lag=…)` |
| `all_bdpro` | `AllBdpro(date_column=…, date_format=…)` |
| `all_free` | `AllFree(date_column=…, date_format=…)` |
| `historical_database=False` | `NonHistorical()` |

| `date_column_name` legado | `date_column` novo | `date_format` exigido (R4) |
|---|---|---|
| `{"date": "x"}` | `DateOnly(col="x")` | `DateFormat.YEAR_MD` (`%Y-%m-%d`) |
| `{"year":"a","month":"m"}` | `YearMonth(year="a", month="m")` | `DateFormat.YEAR_MONTH` (`%Y-%m`) |
| `{"year":"a","quarter":"q"}` | `YearQuarter(year="a", quarter="q")` | `DateFormat.YEAR_MONTH` |
| `{"year":"a"}` | `YearOnly(col="a")` | `DateFormat.YEAR` (`%Y`) |

> **`time_delta` → `free_lag`:** carregar **fielmente**. Default = `FreeLag(months=6)`.
> Exceções: `anp`/`stf` = `FreeLag(weeks, 6)`; `emendas` = `FreeLag(years, 1)`.
> Errar isso desloca o `free_end` da cobertura.

---

## 4. Tabela mestre dos flows ativos

`P` = padrão (§3). Status = situação atual em [`prefect3-flows-migrados.md`](prefect3-flows-migrados.md).

| # | Flow (família) | Tabelas | P | CoverageSpec | date_column / format | free_lag | Status dev | Notas |
|---|---|---|---|---|---|---|---|---|
| — | **`br_bcb_agencia`** (piloto) | 1 | A | PartBdpro | YearMonth(ano,mes) / YEAR_MONTH | 6m | ✅ **prod** | Referência. Concluído (poll `amber-saluki` + materialização in-pod `iron-ferret`). |
| 1 | `br_bcb_estban` | 2 | A | PartBdpro | YearMonth(ano,mes) / YEAR_MONTH | 6m | ✅ **prod** | **Canary Onda 1 — concluído.** poll `cheerful-manul` (R15 `Poll.latest`=2026-06-01, R16 pulado); materialização in-pod `b0326ba6` (cobertura==oráculo, idempotente, RAP reaplicada, `Table.Update.latest`==BQ lastModified). Commits `de4be9df` (migração), `dffc4514` (cleanup harness). |
| 2 | `br_me_comex_stat` | 4 | A | PartBdpro | YearMonth(ano,mes) / YEAR_MONTH | 6m | ✅ **prod** | poll validado em prod (`96debdb5`, `Poll.latest`=2026-06-01, R16 pulado); materialização == canary (spec idêntica) + type-8 |
| 3 | `br_denatran_frota` | 2 | A | PartBdpro | YearMonth(ano,mes) / YEAR_MONTH | 6m | ✅ **prod** | poll validado em prod (`bcbfa9be`); materialização == canary + type-8 |
| 4 | `br_me_caged` | 3 | A | PartBdpro | YearMonth(ano,mes) / YEAR_MONTH | 6m | ✅ **prod** | poll validado em prod (`37b03b95`); materialização == canary + type-8 |
| 5 | `br_mp_pep` | 1 | B | PartBdpro | YearMonth(ano,mes) / YEAR_MONTH | 6m | 🟢 código migrado + deployado | Padrão B (mat-only; gate `is_up_to_date` mantido). Materialização == canary + type-8; run full (Selenium) não disparado |
| 6 | `br_anp_precos_combustiveis` | 1 | A | PartBdpro | DateOnly(data_coleta) / YEAR_MD | **weeks/6** | 🟢 poll prod ✅ / mat. offline ✅ | **Canary Onda 2.** poll validado em prod R15+R16 (`Poll.latest`=2026-06-01, `Update.latest`=2026-05-29); `DateOnly`+`weeks/6` equivalência offline == legado; query `MAX(data_coleta)` construída/submetida OK in-pod, mas write in-pod **bloqueado por 403 Custom quota** (transitório, BQ prod) — reagendar |
| 7 | `br_rf_cafir` | 1 | A | PartBdpro | DateOnly(data_referencia) / YEAR_MD | 6m | 🟢 poll prod ✅ | early-exit (`Poll.latest`=2026-06-01); mat. in-pod adiada (quota) |
| 8 | `br_stf_corte_aberta` | 1 | A | PartBdpro | DateOnly(data_decisao) / YEAR_MD | **weeks/6** | 🟡 deployado; poll pendente | `weeks/6` equivalência offline ✅; run de poll travou no scrape legado `get_data_source_stf_max_date` (não é do refactor); reconfirmar poll + mat. in-pod |
| 9 | `br_inmet_bdmep` | 1 | A | PartBdpro | DateOnly(data) / YEAR_MD | 6m | 🟢 poll prod ✅ | **1º deploy feito.** early-exit (`Poll.latest`=2026-06-01); mat. in-pod adiada (quota) |
| 10 | `br_cvm_oferta_publica_distribuicao` | 1 | B | PartBdpro | DateOnly(data_comunicado) / YEAR_MD | 6m | 🟢 código migrado + deployado | Padrão B; mat. in-pod adiada (quota) |
| 11 | `br_cnj_improbidade_administrativa` | 1 | B | PartBdpro | DateOnly(data_propositura) / YEAR_MD | 6m | 🟢 código migrado + deployado | Padrão B; mat. in-pod adiada (quota) |
| 12 | `br_poder360_pesquisas` | 1 | B | PartBdpro | DateOnly(data) / YEAR_MD | 6m | 🟢 código migrado + deployado | Padrão B; mat. in-pod adiada (quota) |
| 13 | `br_sfb_sicar` | 1 | B | PartBdpro | DateOnly(data_extracao) / YEAR_MD | 6m | 🟢 código migrado + deployado | Padrão B; mat. in-pod adiada (quota) |
| 14 | `br_bcb_taxa_selic` | 1 | A | **AllBdpro** | DateOnly(data) / YEAR_MD | — | 🟢 código migrado + deployado | **Canary Onda 3.** `all_bdpro/DateOnly/YEAR_MD` == legado (type-8 fixado); poll prod **bloqueado por BCB 406** (anti-bot de IP de nuvem, pré-existente — `get_selic_data` falha antes do poll; run `fuzzy-giraffe`); mesma `register_source_poll` já provada em 7 flows; mat. in-pod adiada (quota). Commit `8cc1c722`. |
| 15 | `br_bcb_taxa_cambio` | 1 | B | **AllBdpro** | DateOnly(data_cotacao) / YEAR_MD | — | 🟢 código migrado + deployado | Padrão B; spec == #14; mat. in-pod adiada (quota). Commit `8cc1c722`. |
| 16 | `br_ibge_pnadc` | 1 | A | **AllFree** | **YearQuarter(ano,trimestre)** / YEAR_MONTH | — | 🟢 código migrado | **Canary Onda 4.** `all_free/YearQuarter/YEAR_MONTH` == legado (type-8 fixado); poll usa datetime da fonte (coerção direta); checar limite SIDRA → talvez só 4a; mat. in-pod adiada. Commit `deae5744`. |
| 17 | `br_ans_beneficiario` | 1 | A | PartBdpro | YearMonth(ano,mes) / **YEAR_MONTH** | 6m | 🟢 código migrado | **Canary Onda 5.** R4-**normalização**: legado emitia `endDay/startDay=1` espúrio (`DATE(ano,mes,1).strftime("%Y-%m-%d")`); domínio força YearMonth↔YEAR_MONTH e **descarta o dia** (granularidade correta). Valores ano/mês idênticos (top YearMonth fixado). ⚠ oráculo in-pod deve comparar **ignorando o dia**. Commit `deae5744`. |
| 18 | `br_cgu_emendas_parlamentares` | 1 | A | PartBdpro | **YearOnly(ano_emenda)** / YEAR | **years/1** | 🟢 código migrado | **Canary Onda 6.** `part_bdpro/YearOnly/years:1` == legado (type-8 fixado); dev bloqueado por fonte (FileNotFound) — só código + offline (plano §5). Commit `deae5744`. |

---

## 5. Ondas (sequenciadas por novidade de spec, não por dataset)

Cada onda introduz **uma variação de spec nova**. O **primeiro flow da onda é o canary**: só
após o run de **prod** dele passar (com idempotência + oráculo) os demais da onda seguem em lote.

| Onda | Variação de spec exercitada | Flows | Canary |
|---|---|---|---|
| **0** ✅ | PartBdpro / YearMonth / 6m | `br_bcb_agencia` | concluído |
| **1** ✅ | (idem — gêmeos do piloto) | ✅ #1 estban, ✅ #2 comex, ✅ #3 denatran, ✅ #4 caged, ✅ #5 mp_pep | **CONCLUÍDA** — canary `br_bcb_estban` (poll+materialização full rigor) + #2–#4 poll validado em prod; #5 Padrão B migrado |
| **2** 🟡 | PartBdpro / **DateOnly** (+ free_lag custom) | código ✅ todos migrados+deployados; poll prod ✅ anp(R15+R16)/cafir/inmet, ⏳ stf (scrape lento); **mat. in-pod adiada** (403 Custom quota BQ prod — rodar batelada pós-reset) | `br_anp_precos_combustiveis` |
| **3** 🟢 | **AllBdpro** / DateOnly | #14 taxa_selic, #15 taxa_cambio | código ✅ + offline ✅ (type-8) + deployado; poll prod bloqueado por **BCB 406** (fonte, pré-existente); mat. in-pod adiada (quota) |
| **4** 🟢 | **AllFree / YearQuarter** | #16 pnadc | código ✅ + offline ✅ (type-8); mat. in-pod adiada |
| **5** 🟢 | YearMonth c/ **normalização R4** | #17 ans | código ✅ + offline ✅ (day-drop intencional); oráculo in-pod ignora o dia |
| **6** 🟢 | **YearOnly / YEAR / years-1** | #18 emendas | código ✅ + offline ✅ (type-8); dev bloqueado por fonte (só código+offline) |

> **Estado §4 (18 famílias):** **100% código migrado + offline-provado (type-8, 95/95) + commitado.**
> Pendências de validação (não-bloqueantes de código): batelada in-pod de materialização (Ondas 2–5, após reset da quota BQ prod) + reconfirmar polls bloqueados por fonte (stf scrape, selic/BCB 406).

### 🟢 Onda 7 — flows genéricos em `pipelines/crawler/*/flows.py` (decisão: dobrar no plano)

Varredura `grep -rl` revelou flows P3 ativos em `pipelines/crawler/` (crawlers **genéricos/multi-tabela**,
`dataset_id`/`table_id` por parâmetro) fora da §4. Decisão do usuário: **migrar como Onda 7**.
**14 famílias migradas** (código ✅ + offline 101/101 + import-check); `world_sofascore` **excluído**
(ainda Prefect 0 — ver §2.2). Commits `7cdfb21d` (batch 1), `c25d1d23` (batch 2 + adaptador).

Novidade desta onda: **`coverage_from_legacy_params`** (em `register.py`) — adaptador runtime que mapeia
os parâmetros legados (`coverage_type`/`date_column_name`/`date_format`/`time_delta`/`historical`) para um
`CoverageSpec`, derivando a granularidade R4 das **chaves** de `date_column_name` (normaliza
`{year,month}`+`%Y-%m-%d`→YEAR_MONTH; `historical=False`→NonHistorical). +6 testes unitários.

| Flow (`crawler/`) | P | CoverageSpec | Notas de migração |
|---|---|---|---|
| `rf` | A | PartBdpro/DateOnly(data_extracao)/YEAR_MD | |
| `anatel/banda_larga_fixa` | A | PartBdpro/YearMonth(ano,mes)/YEAR_MONTH | |
| `anatel/telefonia_movel` | A | PartBdpro/YearMonth(ano,mes)/YEAR_MONTH | |
| `isp` | B | AllFree/YearMonth(ano,mes)/YEAR_MONTH | gate `get_count_lines` mantido |
| `me_rais` | B | PartBdpro/YearOnly(ano)/YEAR+years:1 | `effective_table_id` |
| `cvm_administradores_carteira` | B | PartBdpro/DateOnly(data_registro)/YEAR_MD | guard `table_id in (pessoa_fisica,pessoa_juridica)` |
| `cvm` (`br_cvm_fi`) | A | AllBdpro/DateOnly(`date_column_name["date"]`)/YEAR_MD | spec runtime via param dict |
| `cgu` (4 sub-runs) | A | PartBdpro/YearMonth ou via adaptador | polls→`register_source_poll_task` |
| `camara_dados_abertos` | B | via `coverage_from_legacy_params(meta)` | dict-driven (26 tabelas) |
| `datasus` cnes/SIA/SIH | B | PartBdpro/YearMonth(ano,mes)/YEAR_MONTH | gate FTP mantido |
| `datasus` sinan | A | AllFree/DateOnly(data_notificacao)/YEAR_MD | |
| `ibge_inflacao` (`br_ibge_ipca`) | A | PartBdpro/YearMonth(ano,mes)/YEAR_MONTH | |
| `me_cnpj` | A | simples→NonHistorical; else PartBdpro/YearMonth; estabelecimentos→AllBdpro/DateOnly(data) (diretório) | 3 ramos por `table_id` |
| `tse_eleicoes` | A | AllFree/**YearOnly(ano)**/YEAR | legado usava data_eleicao+`%Y` (year-only); R4 proíbe DateOnly+YEAR → coluna `ano`. ⚠ oráculo in-pod só ano |

**`bcb` (`br_bcb_sicor`) — carve-out ✅ RESOLVIDO** (commit `add2b407`): materialização migrada
(dispatch runtime `coverage_type`/`historical` → PartBdpro/YearMonth ou NonHistorical). O **gate por
tamanho** foi **realocado** para uma opção de primeira classe na camada refatorada:
`register_source_poll_by_size` / `register_source_poll_by_size_task` (preserva a lógica/schema Redis
legados, mas grava no backend via `MetadataClient`). `check_if_data_is_outdated_by_size` agora tem
**zero callers de flow** → pode ser deletado no M5.

**Validação prod/in-pod da Onda 7:** dobrada na batelada in-pod pós-reset de quota (mesma doutrina
das Ondas 2–6). Sem redeploy necessário — os deployments existentes apontam para esta branch e o pod
clona o branch a cada run (mudei só corpos de helper, não entrypoints/schedules).

Rationale: as ondas 1–2 reaproveitam diretamente a prova do piloto + equivalência type-8
(`DateOnly part_bdpro` e `YearMonth part_bdpro` já são casos do `test_equivalence.py`). As
ondas 3–6 exercitam caminhos de domínio **ainda não provados em prod** — por isso cada uma
tem canary próprio.

---

## 6. Portões de validação (doutrina do piloto)

### 6.1 Offline — uma vez, já verde
- `uv run pytest pipelines/utils/tests/metadata/` → **92/92** (inclui type-8 dos 4 templates).
- Reexecutar a cada lote como guarda de regressão.

### 6.2 Por flow (espelha agencia)
1. **Deploy** da branch: `python .github/scripts/deploy_flows.py --pool basedosdados --branch <branch> --files <flow>`.
2. **Run dev** (`target=dev`) `Completed` → escreve em `basedosdados-dev` staging.
3. **Run prod** (`target=prod`) `Completed` + asserções **em-pod** (BQ prod só roda no pod — ver memória `project_prod_bq_validation_in_pod`):
   - cobertura free/pro **== oráculo legado** (`utils.get_coverage_parameters`);
   - **idempotência**: re-run sem dado novo ⇒ `depois == antes`;
   - `Table.Update.latest == BQ.last_modified`;
   - RAP reaplicada (só `part_bdpro`).
4. **Padrão B / fontes sem last_modified:** `force_run` no-op ⇒ validar por 1 run + idempotência.
5. **Restrição SIDRA (IBGE):** só smoke **4a** (`force_run=false`) — não rodar 4b. Aplicar a `pnadc` se confirmado limite.

### 6.3 Atualizar tracking
Após prod OK, marcar a linha em `prefect3-flows-migrados.md` (✅ Migrado) e anotar o run id.

---

## 7. M5 — remoção das tasks legadas (gatilho)

Disparar **somente** quando o último caller ativo migrar **e** os bloqueadores §2.1/§2.2 forem resolvidos:

1. Migrar todas as 18 famílias da §4 ✅ **e as 14 da Onda 7** (`pipelines/crawler/`) ✅.
2. Deletar os 7 diretórios deprecados (§2.1) ✅.
3. Resolver os **3 meta-flows P0** (§2.2): `cross_update`, `crawler/world_sofascore`, `utils/metadata/flows.py` (`update_temporal_coverage`): aposentar ou migrar P0→P3.
4. Resolver o carve-out do `bcb_sicor` (§Onda 7): o gate por tamanho `check_if_data_is_outdated_by_size`
   precisa de destino (manter a task isolada, ou re-arquitetar a deteção de mudança).
5. `grep -r "check_if_data_is_outdated\|check_if_data_is_outdated_by_size\|update_django_metadata" pipelines/datasets pipelines/crawler` → **0 ocorrências** (exceto o carve-out do `bcb` enquanto não resolvido).
6. Deletar as 3 tasks de `pipelines/utils/metadata/tasks.py` + helpers legados (`get_coverage_parameters`, `sync_bdpro_and_free_coverage`) e o `test_equivalence.py` (descartável por design). **Manter `coverage_from_legacy_params`** (adaptador da Onda 7, ainda usado por cgu/camara).
7. Adicionar guarda de CI (grep sem callers em `datasets` **e** `crawler`), precedente: `utils_async.py` (commit `8f4f2cde`).

---

## 8. Riscos e armadilhas

| Risco | Mitigação |
|---|---|
| Deploys de prod apontam para a **feature branch** | Após merge na `main`, redeployar com `--branch main` (mesma nota do agencia). |
| `free_lag` custom não carregado (`anp`/`stf`=weeks/6, `emendas`=years/1) | Tabela §4 fixa o valor; cobertura free diverge se errar. Asserção contra oráculo pega. |
| `br_ans`: combo R4-incompatível no legado | Migração **normaliza** p/ `YEAR_MONTH`; type-8 confirma valores idênticos. |
| Canários de spec novo (pnadc/YearQuarter, emendas/YearOnly) sem prova em prod | Onda dedicada + canary; rodar offline + dev antes de prod. |
| Flows com dev **bloqueado por fonte/infra** (`emendas`, OOMs) | Migrar o **código** (borda de metadata); aceitar dev-bloqueado — bloqueio é pré-existente, não do refactor. Não usar como portão. |
| `cross_update` em P0 importando legado | Bloqueia M5. Decisão explícita antes de remover tasks. |

---

## 9. Resumo executivo

- **18 famílias** (~28 flows) a migrar; **7 deprecados** a deletar; **1 meta-flow P0** (`cross_update`) a decidir.
- **6 ondas** por novidade de spec, cada uma com canary em prod; ondas 1–2 (13 flows) são baixo-risco (reaproveitam prova do piloto + type-8).
- Portões por flow = deploy → dev → prod c/ oráculo + idempotência (doutrina do agencia, in-pod p/ BQ prod).
- **M5** (deletar tasks legadas) só após zero callers + deprecados removidos + `cross_update` resolvido.
