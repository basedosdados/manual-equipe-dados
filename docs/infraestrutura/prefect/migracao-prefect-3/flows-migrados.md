# Prefect 3 — Flows Migrados

> Tracking de flows migrados. Para o guia de migração, ver `Prefect 3 Guia de Migração de Flows.md`.

---

## Critérios de migração

Um flow só é considerado **migrado** quando passa pelos três estágios abaixo, em ordem:

| Estágio | Critério | Como validar |
|---|---|---|
| 1. **Deployado** | `deploy_flows.py` registra o flow no Prefect 3 sem erro e o deployment aparece na UI | `uv run prefect deployment ls` mostra o deployment |
| 2. **Rodado em dev** | Pelo menos um run end-to-end completo (`Completed`) com `target=dev`, escrevendo em `basedosdados-dev` | Run com status `Completed` na UI; tabela atualizada em `basedosdados-dev.<dataset>_staging` |
| 3. **Rodado em prod** | Pelo menos um run end-to-end completo (`Completed`) com `target=prod`, escrevendo em `basedosdados` | Run com status `Completed` na UI; tabela atualizada em `basedosdados.<dataset>` |

**Status agregados:**
- 🟡 **Deployado** — passou no estágio 1, falta rodar
- 🟢 **Dev OK** — passou nos estágios 1 e 2
- ✅ **Migrado** — passou nos três estágios
- ⚠️ **Bloqueado** — falhou em algum estágio, com motivo
- ❌ **Não migrado** — ainda em Prefect 0
- 🚫 **Depreciado** — pipeline aposentado; **NÃO migrar**

---

## 🚫 Pipelines depreciados — NÃO MIGRAR

> **Os pipelines abaixo estão depreciados e não devem ser migrados para Prefect 3 em hipótese alguma.** Qualquer tentativa de migração deve ser bloqueada. Se um agente (humano ou LLM) iniciar a migração de qualquer um destes datasets, **interromper imediatamente** e remover o trabalho.

| Dataset / Flow | Diretório no repo | Motivo |
|---|---|---|
| `br_mercadolivre_ofertas` | `pipelines/datasets/br_mercadolivre_ofertas/` | Pipeline depreciado |
| `br_ons_avaliacao_operacao` | `pipelines/datasets/br_ons_avaliacao_operacao/` | Pipeline depreciado |
| `br_ons_estimativa_custos` | `pipelines/datasets/br_ons_estimativa_custos/` | Pipeline depreciado |
| `world_sofascore_competicoes_futebol` | `pipelines/datasets/world_sofascore_competicoes_futebol/` + `pipelines/crawler/world_sofascore_competicoes_futebol/` | Pipeline depreciado |
| `br_b3_cotacoes` | `pipelines/datasets/br_b3_cotacoes/` | Pipeline depreciado |
| `mundo_transfermarkt_competicoes` | `pipelines/datasets/mundo_transfermarkt_competicoes/` | Pipeline depreciado |
| `br_mg_belohorizonte_smfa_iptu` | `pipelines/datasets/br_mg_belohorizonte_smfa_iptu/` | Pipeline depreciado |
| `br_sp_saopaulo_dieese_icv` | `pipelines/datasets/br_sp_saopaulo_dieese_icv/` | Pipeline depreciado |
| `mundo_transfermarkt_competicoes_internacionais` | `pipelines/datasets/mundo_transfermarkt_competicoes_internacionais/` | Pipeline depreciado |

---

## Tracking

> ⚠️ **Flows IBGE Inflação (IPCA / INPC):** validar apenas via smoke test **4a** (`force_run=false`). A API SIDRA tem limite de 100.000 valores por requisição e a janela default quebra (`IndexError`) quando não há mês novo. Detalhes em [`pipelines/crawler/ibge_inflacao/README.md`](pipelines/crawler/ibge_inflacao/README.md). Para esses flows, **4a `Completed` é suficiente para 🟢 Dev OK** — não rodar 4b.

| Flow | Deployado | Dev | Prod | Status | Observações |
|---|---|---|---|---|---|
| `br_camara_dados_abertos__deputado` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Refatorado p/ factory pattern compartilhado (`_run_camara_dados_abertos` em `pipelines/crawler/camara_dados_abertos/flows.py`); 26 tabelas no total. Run end-to-end `Completed` |
| `br_camara_dados_abertos__deputado_ocupacao` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run end-to-end `Completed` |
| `br_camara_dados_abertos__deputado_profissao` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run end-to-end `Completed` |
| `br_camara_dados_abertos__despesa` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | ZIP da Câmara; run end-to-end `Completed` |
| `br_camara_dados_abertos__evento` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run end-to-end `Completed` |
| `br_camara_dados_abertos__evento_orgao` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run end-to-end `Completed` |
| `br_camara_dados_abertos__evento_presenca_deputado` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run end-to-end `Completed` |
| `br_camara_dados_abertos__evento_requerimento` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run end-to-end `Completed` |
| `br_camara_dados_abertos__frente` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run end-to-end `Completed` |
| `br_camara_dados_abertos__frente_deputado` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run end-to-end `Completed` |
| `br_camara_dados_abertos__funcionario` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run end-to-end `Completed` |
| `br_camara_dados_abertos__licitacao` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Cron movido para `15 9 * * *` (conflito com `frente_deputado` no P0) |
| `br_camara_dados_abertos__licitacao_contrato` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run end-to-end `Completed` |
| `br_camara_dados_abertos__licitacao_item` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run end-to-end `Completed` |
| `br_camara_dados_abertos__licitacao_pedido` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run end-to-end `Completed` |
| `br_camara_dados_abertos__licitacao_proposta` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run end-to-end `Completed` |
| `br_camara_dados_abertos__orgao` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run end-to-end `Completed` |
| `br_camara_dados_abertos__orgao_deputado` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run end-to-end `Completed` |
| `br_camara_dados_abertos__proposicao_autor` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run end-to-end `Completed` |
| `br_camara_dados_abertos__proposicao_microdados` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run end-to-end `Completed` |
| `br_camara_dados_abertos__proposicao_tema` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run end-to-end `Completed` |
| `br_camara_dados_abertos__votacao` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run end-to-end `Completed` |
| `br_camara_dados_abertos__votacao_objeto` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run end-to-end `Completed` |
| `br_camara_dados_abertos__votacao_orientacao_bancada` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run end-to-end `Completed` |
| `br_camara_dados_abertos__votacao_parlamentar` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run end-to-end `Completed` |
| `br_camara_dados_abertos__votacao_proposicao` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run end-to-end `Completed` |
| `br_ibge_ipca__mes_brasil` | ✅ 2026-05-21 | ✅ 2026-05-21 (apenas 4a) | ✅ 2026-06-10 | ✅ Migrado | **Validado só via 4a** (`force_run=false`). 4b não se aplica — ver `pipelines/crawler/ibge_inflacao/README.md` (limite de 100k valores da API SIDRA + `IndexError` em `utils.py:238` quando força janela inexistente) |
| `br_ibge_ipca__mes_categoria_brasil` | ✅ 2026-05-21 | ✅ 2026-05-21 (apenas 4a) | ✅ 2026-06-10 | ✅ Migrado | Mesma restrição IBGE — `force_run=false` obrigatório |
| `br_ibge_ipca__mes_categoria_rm` | ✅ 2026-05-21 | ✅ 2026-05-21 (apenas 4a) | ✅ 2026-06-10 | ✅ Migrado | Mesma restrição IBGE — `force_run=false` obrigatório |
| `br_ibge_ipca__mes_categoria_municipio` | ✅ 2026-05-21 | ✅ 2026-05-21 (apenas 4a) | ✅ 2026-06-10 | ✅ Migrado | Mesma restrição IBGE — `force_run=false` obrigatório |
| `br_ibge_inpc__mes_brasil` | ✅ 2026-05-21 | ✅ 2026-05-21 (apenas 4a) | ✅ 2026-06-10 | ✅ Migrado | Mesma restrição IBGE — `force_run=false` obrigatório |
| `br_ibge_inpc__mes_categoria_brasil` | ✅ 2026-05-21 | ✅ 2026-05-21 (apenas 4a) | ✅ 2026-06-10 | ✅ Migrado | Mesma restrição IBGE — `force_run=false` obrigatório |
| `br_ibge_inpc__mes_categoria_rm` | ✅ 2026-05-21 | ✅ 2026-05-21 (apenas 4a) | ✅ 2026-06-10 | ✅ Migrado | Mesma restrição IBGE — `force_run=false` obrigatório |
| `br_ibge_inpc__mes_categoria_municipio` | ✅ 2026-05-21 | ✅ 2026-05-21 (apenas 4a) | ✅ 2026-06-10 | ✅ Migrado | Mesma restrição IBGE — `force_run=false` obrigatório |
| `br_bcb_estban__agencia` | ✅ 2026-05-21 | ✅ 2026-05-21 | ✅ 2026-06-10 | ✅ Migrado | Tasks/utils/constants em `pipelines/crawler/bcb_estban/`; falta rodar em prod |
| `br_bcb_estban__municipio` | ✅ 2026-05-21 | ✅ 2026-05-21 | ✅ 2026-06-10 | ✅ Migrado | Mesma base do flow de agência; falta rodar em prod |
| `br_inmet_bdmep__microdados` | ⏳ | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Migrado em 2026-05-21; tasks/utils/constants movidos para `pipelines/crawler/inmet_bdmep/` |
| `br_me_comex_stat__municipio_exportacao` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Upload p/ staging OK; falha no `dbt run` é problema do modelo/dados (não da migração). Tasks/utils/constants em `pipelines/crawler/me_comex_stat/`; factory pattern p/ 4 tabelas |
| `br_me_comex_stat__municipio_importacao` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Upload p/ staging OK; falha no `dbt run` não relacionada à migração |
| `br_me_comex_stat__ncm_exportacao` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run end-to-end `Completed` |
| `br_me_comex_stat__ncm_importacao` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Upload p/ staging OK; falha no `dbt run` não relacionada à migração |
| `br_bcb_agencia__agencia` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Tasks/utils/constants em `pipelines/crawler/bcb_agencia/`; upload p/ staging OK, falha no `dbt run` não relacionada à migração |
| `br_anatel_banda_larga_fixa__microdados` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Factory pattern em `pipelines/crawler/anatel/banda_larga_fixa/`; run end-to-end `Completed` |
| `br_anatel_banda_larga_fixa__densidade_municipio` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run end-to-end `Completed` |
| `br_anatel_banda_larga_fixa__densidade_brasil` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run end-to-end `Completed` |
| `br_anatel_banda_larga_fixa__densidade_uf` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Primeira tentativa: pod evicted por OOM (`Container prefect-job was using 2432696Ki, request is 0`) — issue transitória de memória do node, não de migração. Retry rodou end-to-end `Completed` |
| `br_anatel_telefonia_movel__microdados` | ✅ 2026-05-22 | ⚠️ 2026-05-22 | ⏳ | ⚠️ Bloqueado | Pod OOMKilled durante download do ZIP da Anatel (`acessos_telefonia_movel.zip`). Migração estruturalmente OK (factory pattern em `pipelines/crawler/anatel/telefonia_movel/`); bloqueio é de infra (memory limit do pod) |
| `br_anatel_telefonia_movel__densidade_municipio` | ✅ 2026-05-22 | ⚠️ 2026-05-22 | ⏳ | ⚠️ Bloqueado | Pod OOMKilled durante download — mesmo problema do microdados (memory limit) |
| `br_anatel_telefonia_movel__densidade_uf` | ✅ 2026-05-22 | ⚠️ 2026-05-22 | ⏳ | ⚠️ Bloqueado | Pod OOMKilled durante download — mesmo problema do microdados (memory limit) |
| `br_anatel_telefonia_movel__densidade_brasil` | ✅ 2026-05-22 | ⚠️ 2026-05-22 | ⏳ | ⚠️ Bloqueado | Pod OOMKilled durante download — mesmo problema do microdados (memory limit) |
| `br_anp_precos_combustiveis__microdados` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Tasks/utils/constants em `pipelines/crawler/anp_precos_combustiveis/`. Fix: `get_id_municipio` trocou `bd.read_table(billing_project_id=basedosdados)` por `bd.read_sql(..., from_file=True)` (mesmo padrão do br_bcb_estban). Run end-to-end `Completed` |
| `br_bcb_sicor__operacao` | ✅ 2026-05-22 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Tasks/utils/constants em `pipelines/crawler/bcb/`; factory pattern p/ 10 tabelas + flow dedicado p/ `dicionario`. Tabelas grandes — pod com recursos limitados, run dev não executada |
| `br_bcb_sicor__saldo` | ✅ 2026-05-22 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Tabela grande — run dev não executada |
| `br_bcb_sicor__liberacao` | ✅ 2026-05-22 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Tabela grande — run dev não executada |
| `br_bcb_sicor__recurso_publico_complemento_operacao` | ✅ 2026-05-22 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Tabela grande — run dev não executada |
| `br_bcb_sicor__recurso_publico_cooperado` | ✅ 2026-05-22 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Tabela grande — run dev não executada |
| `br_bcb_sicor__recurso_publico_gleba` | ✅ 2026-05-22 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Tabela grande — run dev não executada |
| `br_bcb_sicor__recurso_publico_mutuario` | ✅ 2026-05-22 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Tabela grande — run dev não executada |
| `br_bcb_sicor__recurso_publico_propriedade` | ✅ 2026-05-22 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Tabela grande — run dev não executada |
| `br_bcb_sicor__operacoes_desclassificadas` | ✅ 2026-05-22 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Tabela grande — run dev não executada |
| `br_bcb_sicor__empreendimento` | ✅ 2026-05-22 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | CSV; sem schedule no Prefect 0 — `source_format=csv`, `coverage_type=all_free`, `historical_database=False`. Run dev não executada |
| `br_bcb_sicor__dicionario` | ✅ 2026-05-22 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Flow dedicado (não usa factory); sem schedule. Run dev não executada |
| `br_cgu_beneficios_cidadao__novo_bolsa_familia` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Tasks compartilhadas em `pipelines/crawler/cgu/`; factory pattern; run dev triggered |
| `br_cgu_beneficios_cidadao__garantia_safra` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run dev triggered |
| `br_cgu_beneficios_cidadao__bpc` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run dev triggered |
| `br_cgu_cartao_pagamento__microdados_governo_federal` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run dev triggered |
| `br_cgu_cartao_pagamento__microdados_defesa_civil` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run dev triggered |
| `br_cgu_cartao_pagamento__microdados_compras_centralizadas` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run dev triggered |
| `br_cgu_licitacao_contrato__contrato_compra` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run dev triggered |
| `br_cgu_licitacao_contrato__contrato_item` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run dev triggered |
| `br_cgu_licitacao_contrato__contrato_termo_aditivo` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run dev triggered |
| `br_cgu_licitacao_contrato__licitacao` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run dev triggered |
| `br_cgu_licitacao_contrato__licitacao_empenho` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Sem schedule no P0 — sem cron no P3. Run dev triggered |
| `br_cgu_licitacao_contrato__licitacao_item` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run dev triggered |
| `br_cgu_licitacao_contrato__licitacao_participante` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run dev triggered |
| `br_cgu_servidores_executivo_federal__cadastro_aposentados` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run dev triggered |
| `br_cgu_servidores_executivo_federal__cadastro_pensionistas` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run dev triggered |
| `br_cgu_servidores_executivo_federal__cadastro_servidores` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run dev triggered |
| `br_cgu_servidores_executivo_federal__cadastro_reserva_reforma_militares` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run dev triggered |
| `br_cgu_servidores_executivo_federal__remuneracao` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run dev triggered |
| `br_cgu_servidores_executivo_federal__afastamentos` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run dev triggered |
| `br_cgu_servidores_executivo_federal__observacoes` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run dev triggered |
| `br_cgu_pessoal_executivo_federal__terceirizados` | ✅ 2026-05-22 | ⚠️ 2026-05-22 | ⏳ | ⚠️ Bloqueado | Run dev FAILED — `UnicodeDecodeError: 'utf-8' codec can't decode byte 0xba` ao ler CSV terceirizados (encoding latin-1 na fonte). Migração estruturalmente OK; bloqueio é da fonte/ETL pré-existente |
| `br_cgu_emendas_parlamentares__microdados` | ✅ 2026-05-22 | ⚠️ 2026-05-22 | ⏳ | ⚠️ Bloqueado | Run dev FAILED — `FileNotFoundError: /tmp/input/EmendasParlamentares.csv` (download/unzip falhou silenciosamente no `download_unzip_file`). Migração estruturalmente OK; bloqueio é da fonte/ETL pré-existente |
| `br_denatran_frota__uf_tipo` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Tasks/utils/constants em `pipelines/crawler/denatran_frota/`. Loop sequencial sobre datas (substitui `.map()` do P0); `triggers.all_finished` removido. Run dev triggered |
| `br_denatran_frota__municipio_tipo` | ✅ 2026-05-22 | ✅ 2026-05-22 | ✅ 2026-06-10 | ✅ Migrado | Run dev triggered |
| `br_me_caged__microdados_movimentacao` | ✅ 2026-05-22 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Tasks/utils/constants em `pipelines/crawler/me_caged/`; factory pattern p/ 3 tabelas. Loop sequencial sobre `yearmonths` (substitui `.map()` do P0); `trigger=all_finished` removido. Run dev triggered |
| `br_me_caged__microdados_movimentacao_fora_prazo` | ✅ 2026-05-22 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Run dev triggered |
| `br_me_caged__microdados_movimentacao_excluida` | ✅ 2026-05-22 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Run dev triggered |
| `br_bcb_taxa_selic` | ✅ 2026-05-25 | ❌ | ⏳ | ⚠️ Schedule desativado | **NÃO ATIVAR** até fix do HTTP 406 (BCB API rejeita User-Agent padrão). 6 falhas consecutivas em prod (03-08/06). Fix: adicionar header `User-Agent` na task `get_selic_data`. |
| `br_fgv_igp__igp_di_mes` | ✅ 2026-05-25 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Factory `_run_fgv_igp` em `pipelines/crawler/fgv_igp/flows.py` p/ 7 tabelas. Crons originais P0 (commented out): mensais dia 8, anuais dia 8/janeiro. Run dev pendente |
| `br_fgv_igp__igp_di_ano` | ✅ 2026-05-25 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Mesmo crawler |
| `br_fgv_igp__igp_m_mes` | ✅ 2026-05-25 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Mesmo crawler |
| `br_fgv_igp__igp_m_ano` | ✅ 2026-05-25 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Mesmo crawler |
| `br_fgv_igp__igp_og_mes` | ✅ 2026-05-25 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Mesmo crawler |
| `br_fgv_igp__igp_og_ano` | ✅ 2026-05-25 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Mesmo crawler |
| `br_fgv_igp__igp_10_mes` | ✅ 2026-05-25 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Mesmo crawler — `igp_10_ano` não existia no P0 |
| `br_ans_beneficiario__informacao_consolidada` | ✅ 2026-05-25 | ⏳ (4a ✅, 4b em andamento) | ✅ 2026-06-10 | ✅ Migrado | Deploy OK em `basedosdados-dev`; smoke 4a `Completed` (run `maroon-robin` 1f980e83); 4b `force_run=true` (`burrowing-jerboa` 4b08768b) ainda `Running` |
| `br_bcb_taxa_cambio__taxa_cambio` | ✅ 2026-05-25 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Tasks/utils/constants extraídos p/ `pipelines/crawler/bcb_taxa_cambio/`. `dbt_alias=True`. Run dev pendente |
| `br_bd_indicadores__twitter_metrics` | ✅ 2026-05-25 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Tasks extraídas p/ `pipelines/crawler/bd_indicadores/`. `get_storage_blobs` reimplementada local (não existe no novo utils). Run dev pendente |
| `br_bd_indicadores__twitter_metrics_agg` | ✅ 2026-05-25 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Apenas dbt + download_to_gcs (sem upload). Removida dependência ilegal de var de outro flow (P0). Run dev pendente |
| `br_bd_indicadores__page_views` | ✅ 2026-05-25 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Crawler GA4 real-time. Run dev pendente |
| `br_bd_indicadores__website_user` | ✅ 2026-05-25 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Tabela `website_user` (era `ga_users` no tracking). Crawler GA. Run dev pendente |
| `br_bd_indicadores__contabilidade` | ✅ 2026-05-25 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Google Sheets → CSV. Helpers `_sheet_flow_body` no flows.py |
| `br_bd_indicadores__receitas_planejadas` | ✅ 2026-05-25 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Mesmo helper sheet |
| `br_bd_indicadores__equipes` | ✅ 2026-05-25 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Mesmo helper sheet, `usecols=6` |
| `br_bd_indicadores__pessoas` | ✅ 2026-05-25 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Mesmo helper sheet, `usecols=9` |
| `br_bd_siga_o_dinheiro` | ✅ 2026-05-25 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Template P3: get_table_ids() dinâmico + dbt + download_to_gcs por tabela. Sem schedule |
| `br_cnj_improbidade_administrativa__condenacao` | ✅ 2026-05-25 | ❌ | ⏳ | ⚠️ Schedule desativado | **NÃO ATIVAR** até investigar falhas. ConnectionError intermitente em prod (03-08/06). |
| `br_cvm_administradores_carteira__responsavel` | ✅ 2026-05-25 | ✅ 2026-05-25 | ✅ 2026-06-10 | ✅ Migrado | Tasks compartilhadas em `pipelines/crawler/cvm_administradores_carteira/`; factory pattern p/ 3 tabelas. Fonte (ZIP CVM) sem `last_modified` — `check_if_data_is_outdated` omitido, `force_run` no-op (4b ≡ 4a). Run end-to-end `Completed` |
| `br_cvm_administradores_carteira__pessoa_fisica` | ✅ 2026-05-25 | ✅ 2026-05-25 | ✅ 2026-06-10 | ✅ Migrado | Upload p/ staging OK; falha no `dbt run` não relacionada à migração. `update_django_metadata` ativo (data_registro). `force_run` no-op (4b ≡ 4a) |
| `br_cvm_administradores_carteira__pessoa_juridica` | ✅ 2026-05-25 | ✅ 2026-05-25 | ✅ 2026-06-10 | ✅ Migrado | Upload p/ staging OK; falha no `dbt run` não relacionada à migração. `force_run` no-op (4b ≡ 4a) |
| `br_cvm_fi__documentos_informe_diario` | ✅ 2026-05-25 | ✅ 2026-05-25 | ✅ 2026-06-10 | ✅ Migrado | Crawler P3 em `pipelines/crawler/cvm/`; factory pattern p/ 6 tabelas; `date_column_name` baked por tabela. Run end-to-end `Completed` |
| `br_cvm_fi__documentos_carteiras_fundos_investimento` | ✅ 2026-05-25 | ✅ 2026-05-25 | ✅ 2026-06-10 | ✅ Migrado | Run end-to-end `Completed` |
| `br_cvm_fi__documentos_extratos_informacoes` | ✅ 2026-05-25 | ✅ 2026-05-25 | ✅ 2026-06-10 | ✅ Migrado | Run end-to-end `Completed` |
| `br_cvm_fi__documentos_perfil_mensal` | ✅ 2026-05-25 | ⚠️ 2026-05-25 | ⏳ | ⚠️ Bloqueado | Pod evicted por MemoryPressure no node — issue de infra (memory limit), migração estruturalmente OK |
| `br_cvm_fi__documentos_informacao_cadastral` | ✅ 2026-05-25 | ✅ 2026-05-25 | ✅ 2026-06-10 | ✅ Migrado | `date_column_name={"date":"data_inicio_situacao"}`. Run end-to-end `Completed` |
| `br_cvm_fi__documentos_balancete` | ✅ 2026-05-25 | ⚠️ 2026-05-25 | ⏳ | ⚠️ Bloqueado | Pod OOMKilled durante processamento dos balancetes (12 meses × CSVs grandes). Migração estruturalmente OK; bloqueio é de infra (memory limit do pod) |
| `br_cvm_oferta_publica_distribuicao__dia` | ✅ 2026-05-25 | ✅ 2026-05-25 | ✅ 2026-06-10 | ✅ Migrado | Tasks em `pipelines/crawler/cvm_oferta_publica_distribuicao/`. Fonte (ZIP CVM) sem `last_modified` — `check_if_data_is_outdated` omitido, `force_run` no-op (4b ≡ 4a). Run end-to-end `Completed` |
| `br_ibge_ipca15__mes_brasil` | ✅ 2026-05-25 | ✅ 2026-05-25 (apenas 4a) | ✅ 2026-06-10 | ✅ Migrado | Reusa `crawler/ibge_inflacao/`. Restrição IBGE SIDRA — só 4a. Run end-to-end `Completed` |
| `br_ibge_ipca15__mes_categoria_brasil` | ✅ 2026-05-25 | ✅ 2026-05-25 (apenas 4a) | ✅ 2026-06-10 | ✅ Migrado | Mesmo crawler IBGE. Run end-to-end `Completed` |
| `br_ibge_ipca15__mes_categoria_rm` | ✅ 2026-05-25 | ✅ 2026-05-25 (apenas 4a) | ✅ 2026-06-10 | ✅ Migrado | Mesmo crawler IBGE. Run end-to-end `Completed` |
| `br_ibge_ipca15__mes_categoria_municipio` | ✅ 2026-05-25 | ✅ 2026-05-25 (apenas 4a) | ✅ 2026-06-10 | ✅ Migrado | Mesmo crawler IBGE. Run end-to-end `Completed` |
| `br_ibge_pnadc__microdados` | ✅ 2026-05-25 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Tasks extraídas p/ `pipelines/crawler/ibge_pnadc/`. `check_if_data_is_outdated` + `download_async`. Cron `0 5 15-31 2,5,8,11 *` |
| `br_jota` (sem flows.py) | — | — | — | ❌ Não migrado | Pasta `pipelines/datasets/br_jota/` existe mas sem `flows.py`; nada para migrar — confirmar se é pipeline ativo ou pode ser removido |
| `br_me_cnpj__empresas` | ✅ 2026-05-25 | ✅ 2026-05-25 | ✅ 2026-06-10 | ✅ Migrado | Tasks/utils/constants em `pipelines/crawler/me_cnpj/`; factory pattern p/ 4 tabelas. Run `Completed` |
| `br_me_cnpj__socios` | ✅ 2026-05-25 | ✅ 2026-05-25 | ✅ 2026-06-10 | ✅ Migrado | Run `Completed` |
| `br_me_cnpj__estabelecimentos` | ✅ 2026-05-25 | ✅ 2026-05-25 | ✅ 2026-06-10 | ✅ Migrado | Inclui re-materialização de `br_bd_diretorios_brasil.empresa` + `download_data_to_gcs`. Run `Completed` |
| `br_me_cnpj__simples` | ✅ 2026-05-25 | ✅ 2026-05-25 | ✅ 2026-06-10 | ✅ Migrado | `coverage_type=all_free`, `historical_database=False`. Run `Completed` |
| `br_me_rais__microdados_estabelecimentos` | ✅ 2026-05-25 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Factory `_run_rais` em `pipelines/crawler/me_rais/flows.py`. `resolve_vinculos=False`. Sem schedule (P0 também não tinha). Run dev pendente |
| `br_me_rais__microdados_vinculos` | ✅ 2026-05-25 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Mesmo crawler, `resolve_vinculos=True` (resolve table_id dinamicamente por ano) |
| `br_mp_pep__cargos_funcoes` | ✅ 2026-05-25 | ❌ | ⏳ | ⚠️ Schedule desativado | **NÃO ATIVAR** até fix do Selenium Timeout. Webdriver está expirando em prod (03-08/06). Fix: aumentar timeout ou adicionar retry no webdriver. |
| `br_mp_pep_cargos_funcoes` (sem flows.py) | — | — | — | ❌ Não migrado | Pasta `pipelines/datasets/br_mp_pep_cargos_funcoes/` existe mas sem `flows.py`; possivelmente duplicada por `br_mp_pep` — confirmar |
| `br_ms_cnes__profissional` | ✅ 2026-05-25 | ✅ 2026-05-25 | ✅ 2026-06-10 | ✅ Migrado | Factory pattern em `pipelines/crawler/datasus/` (`_run_cnes` compartilhado p/ 13 tabelas). `ShellTask` → `subprocess.run`; decoradores → P3. Run end-to-end `Completed` |
| `br_ms_cnes__estabelecimento` | ✅ 2026-05-25 | ✅ 2026-05-25 | ✅ 2026-06-10 | ✅ Migrado | Upload p/ staging OK; falha no `dbt run` não relacionada à migração |
| `br_ms_cnes__leito` | ✅ 2026-05-25 | ✅ 2026-05-25 | ✅ 2026-06-10 | ✅ Migrado | Upload p/ staging OK; falha no `dbt run` não relacionada à migração |
| `br_ms_cnes__equipamento` | ✅ 2026-05-25 | ✅ 2026-05-25 | ✅ 2026-06-10 | ✅ Migrado | Upload p/ staging OK; falha no `dbt run` não relacionada à migração |
| `br_ms_cnes__equipe` | ✅ 2026-05-25 | ✅ 2026-05-25 | ✅ 2026-06-10 | ✅ Migrado | Upload p/ staging OK; falha no `dbt run` não relacionada à migração |
| `br_ms_cnes__dados_complementares` | ✅ 2026-05-25 | ✅ 2026-05-25 | ✅ 2026-06-10 | ✅ Migrado | Upload p/ staging OK; falha no `dbt run` não relacionada à migração |
| `br_ms_cnes__servico_especializado` | ✅ 2026-05-25 | ✅ 2026-05-25 | ✅ 2026-06-10 | ✅ Migrado | Upload p/ staging OK; falha no `dbt run` não relacionada à migração |
| `br_ms_cnes__habilitacao` | ✅ 2026-05-25 | ✅ 2026-05-25 | ✅ 2026-06-10 | ✅ Migrado | Upload p/ staging OK; falha no `dbt run` não relacionada à migração |
| `br_ms_cnes__gestao_metas` | ✅ 2026-05-25 | ✅ 2026-05-25 | ✅ 2026-06-10 | ✅ Migrado | Upload p/ staging OK; falha no `dbt run` não relacionada à migração |
| `br_ms_cnes__incentivos` | ✅ 2026-05-25 | ✅ 2026-05-25 | ✅ 2026-06-10 | ✅ Migrado | Upload p/ staging OK; falha no `dbt run` não relacionada à migração |
| `br_ms_cnes__estabelecimento_ensino` | ✅ 2026-05-25 | ✅ 2026-05-25 | ✅ 2026-06-10 | ✅ Migrado | Sem schedule no P0 (mantido sem cron no P3). Run `Completed` |
| `br_ms_cnes__estabelecimento_filantropico` | ✅ 2026-05-25 | ✅ 2026-05-25 | ✅ 2026-06-10 | ✅ Migrado | Upload p/ staging OK; falha no `dbt run` não relacionada à migração |
| `br_ms_cnes__regra_contratual` | ✅ 2026-05-25 | ⚠️ 2026-05-25 | ⏳ | ⚠️ Bloqueado | Sem schedule no P0; nome corrigido para `br_ms_cnes__regra_contratual`. Run FAILED com `ParserError: Error tokenizing data. C error: Expected 31 fields in line 213, saw 32` — fonte CSV malformada; migração estruturalmente OK, bloqueio é de qualidade dos dados pré-existente |
| `br_ms_sia__producao_ambulatorial` | ✅ 2026-05-25 | ✅ 2026-05-25 | ✅ 2026-06-10 | ✅ Migrado | `_run_siasus` em `pipelines/crawler/datasus/flows.py` (pipeline DBF→Parquet). Run levou ~3h20 (decode de 30+ chunks DBF). Upload p/ `gs://basedosdados-dev/staging/br_ms_sia/producao_ambulatorial` OK; falha no `dbt run` não relacionada à migração |
| `br_ms_sia__psicossocial` | ✅ 2026-05-25 | ✅ 2026-05-25 | ✅ 2026-06-10 | ✅ Migrado | Mesmo crawler. Run `Completed` |
| `br_ms_sih__servicos_profissionais` | ✅ 2026-05-25 | ✅ 2026-05-25 | ✅ 2026-06-10 | ✅ Migrado | `_run_sihsus` em `pipelines/crawler/datasus/flows.py` (mesmo `_run_dbf_to_parquet` do SIA, com `source_format=parquet`). Run `Completed` |
| `br_ms_sih__aihs_reduzidas` | ✅ 2026-05-25 | ✅ 2026-05-25 | ✅ 2026-06-10 | ✅ Migrado | Upload p/ staging OK; falha no `dbt run` não relacionada à migração |
| `br_ms_sinan__microdados_dengue` | ✅ 2026-05-25 | ✅ 2026-05-25 | ✅ 2026-06-10 | ✅ Migrado | `_run_sinan` em `pipelines/crawler/datasus/flows.py`. Sem schedule (mantido como no P0); `coverage_type=all_free`, `date_column_name={"date":"data_notificacao"}`. Upload p/ staging OK; 4 testes dbt falharam (não relacionados à migração) |
| `br_poder360_pesquisas__microdados` | ✅ 2026-05-25 | ❌ | ⏳ | ⚠️ Schedule desativado | **NÃO ATIVAR** até fix do crawler. Falha com `KeyError: 'BIG THEME'` e `ConnectionError` em prod (03-08/06). Dado ausente ou mudança na estrutura da fonte. |
| `br_rf_cafir__imoveis_rurais` | ✅ 2026-05-25 | ✅ 2026-05-25 | ✅ 2026-06-10 | ✅ Migrado | Tasks/utils/constants em `pipelines/crawler/rf_cafir/`. 4a Completed; 4b ainda running |
| `br_rf_cno__microdados` | ✅ 2026-05-25 | ✅ 2026-05-25 | ✅ 2026-06-10 | ✅ Migrado | `_run_rf` compartilhado em `pipelines/crawler/rf/flows.py`; factory pattern p/ 4 tabelas. `crawl` task migrada p/ decorator P3 (`retries`/`retry_delay_seconds`). Crons originais (todos `5 4 * * *` filtro `is_weekday`) staggerados para `5/15/25/35 4 * * 1-5`. Upload p/ staging OK (4a + 4b); falha no `dbt run` não relacionada à migração |
| `br_rf_cno__vinculos` | ✅ 2026-05-25 | ✅ 2026-05-25 | ✅ 2026-06-10 | ✅ Migrado | Mesmo crawler. Upload p/ staging OK (4a + 4b); falha no `dbt run` não relacionada à migração |
| `br_rf_cno__areas` | ✅ 2026-05-25 | ✅ 2026-05-25 | ✅ 2026-06-10 | ✅ Migrado | Mesmo crawler. Upload p/ staging OK (4a + 4b); falha no `dbt run` não relacionada à migração |
| `br_rf_cno__cnaes` | ✅ 2026-05-25 | ✅ 2026-05-25 | ✅ 2026-06-10 | ✅ Migrado | Mesmo crawler. Upload p/ staging OK (4a + 4b); falha no `dbt run` não relacionada à migração |
| `br_rj_isp_estatisticas_seguranca__evolucao_mensal_cisp` | ✅ 2026-05-25 | ✅ 2026-05-25 | ✅ 2026-06-10 | ✅ Migrado | `_run_isp` em `pipelines/crawler/isp/flows.py`; factory pattern p/ 6 tabelas. Fix: `bd.read_sql` sem `billing_project_id` explícito (mesmo padrão do br_anp_precos_combustiveis). Upload p/ staging OK; falha no `dbt test` (relationships id_municipio) não relacionada à migração |
| `br_rj_isp_estatisticas_seguranca__evolucao_mensal_uf` | ✅ 2026-05-25 | ✅ 2026-05-25 | ✅ 2026-06-10 | ✅ Migrado | Mesmo crawler. Run `Completed` |
| `br_rj_isp_estatisticas_seguranca__evolucao_mensal_municipio` | ✅ 2026-05-25 | ✅ 2026-05-25 | ✅ 2026-06-10 | ✅ Migrado | Mesmo crawler — trailing whitespace no nome do P0 removido. Run `Completed` |
| `br_rj_isp_estatisticas_seguranca__armas_apreendidas_mensal` | ✅ 2026-05-25 | ⚠️ 2026-05-25 | ⏳ | ⚠️ Bloqueado | Run FAILED com `KeyError: ['quantidade_artefato_explosivo_bomba_fabricacao_caseira', 'quantidade_artefato_explosivo_material_explosivo_caseiro'] not in index` em `clean_data` (colunas esperadas pela arquitetura ausentes na fonte). Migração estruturalmente OK; bloqueio é discrepância de schema fonte↔arquitetura pré-existente |
| `br_rj_isp_estatisticas_seguranca__evolucao_policial_morto_servico_mensal` | ✅ 2026-05-25 | ✅ 2026-05-25 | ✅ 2026-06-10 | ✅ Migrado | Mesmo crawler. Run `Completed` |
| `br_rj_isp_estatisticas_seguranca__feminicidio_mensal_cisp` | ✅ 2026-05-25 | ✅ 2026-05-25 | ✅ 2026-06-10 | ✅ Migrado | Mesmo crawler. Run `Completed` |
| `br_sfb_sicar__area_imovel` | ✅ 2026-05-25 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Tasks extraídas p/ `pipelines/crawler/sfb_sicar/`. `.map()` do P0 convertido p/ loop sequencial sobre 27 UFs. Cron `15 21 15 * *`. Run dev pendente |
| `br_stf_corte_aberta__decisoes` | ✅ 2026-05-25 | ❌ | ⏳ | ⚠️ Schedule desativado | **NÃO ATIVAR** até fix do Selenium Timeout. Webdriver expirando em prod (03-08/06), 6 falhas consecutivas. Fix: aumentar timeout ou adicionar retry no webdriver. |
| `br_tse_eleicoes__candidatos` | ✅ 2026-05-25 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Factory `_run_tse_eleicoes` em `pipelines/crawler/tse_eleicoes/flows.py` p/ 4 tabelas. Sem schedule (P0 também sem). Run dev pendente |
| `br_tse_eleicoes__bens_candidato` | ✅ 2026-05-25 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Mesmo crawler TSE |
| `br_tse_eleicoes__despesas_candidato` | ✅ 2026-05-25 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Mesmo crawler TSE |
| `br_tse_eleicoes__receitas_candidato` | ✅ 2026-05-25 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Mesmo crawler TSE |
| `fundacao_lemann__ano_escola_serie_educacao_aprendizagem_adequada` | ✅ 2026-05-25 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Apenas dbt + download_to_gcs. Cron anual `0 9 1 1 *`. Run dev pendente |

**Legenda extra:**
- ♻️ — deployment registrado anteriormente, mas a assinatura mudou (novo param `force_run`); precisa re-rodar `deploy_flows.py` para atualizar o schema do deployment na UI.
- **"(apenas 4a)"** — flow validado em dev somente pelo smoke test 4a (`force_run=false`). Usado quando 4b (`force_run=true`) não é aplicável por restrição da fonte (ex.: limite de requisições da API SIDRA nos flows IBGE Inflação). Ver README do crawler correspondente.

---

## Próximos a migrar (sugestão de prioridade)

| Flow | Complexidade | Motivo |
|---|---|---|
| `br_bcb_taxa_selic` | Baixa | Só falta o fix do `User-Agent` |
| `br_ibge_inpc` 4 tabelas | Baixa | Dev OK; faltam runs em prod (apenas 4a — `force_run=false`, ver README do crawler) |
| `br_ibge_ipca` 4 tabelas | Baixa | Dev OK; faltam runs em prod (apenas 4a — `force_run=false`, ver README do crawler) |
| outros crawlers IBGE | Média | Verificar se usam `task.run()` nos utils |
| `br_fgv_igp__igp_di_mes` | ✅ 2026-05-25 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Factory `_run_fgv_igp` em `pipelines/crawler/fgv_igp/flows.py` p/ 7 tabelas. Crons originais P0 (commented out): mensais dia 8, anuais dia 8/janeiro. Run dev pendente |
| `br_fgv_igp__igp_di_ano` | ✅ 2026-05-25 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Mesmo crawler |
| `br_fgv_igp__igp_m_mes` | ✅ 2026-05-25 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Mesmo crawler |
| `br_fgv_igp__igp_m_ano` | ✅ 2026-05-25 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Mesmo crawler |
| `br_fgv_igp__igp_og_mes` | ✅ 2026-05-25 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Mesmo crawler |
| `br_fgv_igp__igp_og_ano` | ✅ 2026-05-25 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Mesmo crawler |
| `br_fgv_igp__igp_10_mes` | ✅ 2026-05-25 | ⏳ | ✅ 2026-06-10 | ✅ Migrado | Mesmo crawler — `igp_10_ano` não existia no P0 |
