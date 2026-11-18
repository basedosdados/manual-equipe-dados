---
tipo: adr
titulo: ADR-0001 — Dividir os flows de dataset em estágios encadeados
status: aceito
data: 2026-10-05
pr: https://github.com/basedosdados/pipelines/pull/1932
---
# ADR-0001 — Dividir os flows de dataset em estágios encadeados

## Contexto

Até 2026, cada flow de dataset era monolítico: numa única execução, num único pod Kubernetes,
checava se havia dado novo, baixava, subia para o GCS, materializava e testava em dev, promovia para
prod e atualizava os metadados. A [issue #1867](https://github.com/basedosdados/pipelines/issues/1867)
registrou quatro problemas desse desenho:

1. **Desperdício de recurso:** pods de 4 GB subiam só para verificar se havia dado novo, e na
   maioria das execuções a resposta era "não".
2. **Recursos iguais para cargas diferentes:** o check (leve), o download (médio) e a
   materialização dbt (pesada) rodavam no mesmo pod, com a mesma alocação.
3. **Cota de BigQuery compartilhada:** o teste em dev consumia a cota de `basedosdados-dev` junto
   com o desenvolvimento manual da equipe.
4. **Reruns custosos:** uma falha em prod obrigava a rodar tudo de novo desde o check, mesmo com o
   dado já no storage.

## Decisão

Vamos dividir cada flow de dataset em estágios encadeados, cada um num deployment e num pod
próprios: `check_update` → `extract_and_load` → `build_and_promote`. Em concreto:

- cada estágio dispara o próximo com `run_deployment()` no próprio código (`timeout=0`,
  `as_subflow=True`), sem Automação do Prefect;
- o `build_and_promote` é um deployment único, compartilhado por todos os datasets e sempre no pool
  de prod, porque só a service account desse pool escreve em prod;
- a promoção para prod é derivada do pool em que o `extract_and_load` rodou (`_dev_prefix()`), e
  não declarada pelo dataset;
- o dbt de dev e o de prod rodam no mesmo pod do `build_and_promote`;
- existem duas variantes: `check_update`, para fontes que expõem a data mais recente sem baixar o
  arquivo, e `check_and_download`, para as que não expõem.

A implementação está no `pipelines/utils/stage_dispatch.py` e foi migrada no PR #1932.

## Consequências

### Positivas
- Um check que não acha dado novo roda num pod pequeno e termina ali.
- Cada estágio recebe só os recursos de que precisa.
- Uma falha na promoção pode ser repetida sem baixar o dado de novo.
- O número de deployments cresce de forma linear: N `check_update` + N `extract_and_load` + 1
  `build_and_promote`, em vez de 3×N.
- Um teste em dev não consegue promover para prod: a credencial do pool é a fronteira, e não o
  código do dataset.

### Negativas
- Cada estágio leva de 70 a 90 segundos a mais para subir o pod.
- Não há uma visão agregada de "o que dispara o quê" sem ler o código.
- Uma falha ao disparar o próximo estágio vira exceção dentro do estágio anterior e pode marcá-lo
  como `Failed`, mesmo com o trabalho dele concluído.
- Um teste em dev ocupa o pool de prod durante o `build_and_promote`.

### Neutras
- Há mais runs para acompanhar na interface do Prefect.
- Datasets ainda não migrados continuam como flow único até a migração.

## Alternativas consideradas

- **Encadear os estágios com Automação do Prefect 3 e `emit_event()`** — prototipado e testado de
  ponta a ponta, mas descartado antes do rollout: escalar para cerca de 82 datasets exigiria um
  mecanismo próprio de sincronização das automações, os parâmetros chegam sempre como string (via
  Jinja) e o run disparado não aparece linkado como subflow.
- **Dois flows separados para dev e prod no `build_and_promote`** — permitiria repetir só prod em
  caso de falha, mas esse caso é raro, e o isolamento de cota já vem do `execution_project` do
  target dbt. Não justifica um pod a mais por execução.
- **Deixar o dataset declarar se promove para prod (`ExtractAndLoad.targets`)** — foi a primeira
  versão e foi removida: o dataset podia pedir a promoção mesmo rodando num teste em dev.
- **Uma única variante para todos os datasets** — forçar o check leve é impossível para fontes que
  só informam a data dentro do arquivo, e forçar todos a baixar para checar desperdiçaria a
  economia que motivou a mudança.

## Status

Aceito. Implementado no [PR #1932](https://github.com/basedosdados/pipelines/pull/1932), que fechou
a [issue #1867](https://github.com/basedosdados/pipelines/issues/1867). O funcionamento está
explicado em [Pipeline orientado a eventos](../explicacao/pipeline-orientado-a-eventos.md) e em
[Pools e pods](../../infraestrutura/prefect/explicacao/pools-e-pods.md).
