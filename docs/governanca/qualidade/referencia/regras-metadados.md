---
tipo: referencia
titulo: Regras de negócio do preenchimento de metadados
---
# Regras de negócio do preenchimento de metadados

Regras que governam a consistência dos metadados de `Dataset`, `Table`,
`RawDataSource` e `Column` no backend. São restrições de negócio que devem ser validadas antes dos conjuntos e tabelas serem considerados aptos a publicação em produção.

Para o passo a passo de como popular cada entidade, ver
[Registrar metadados no backend](../../../infraestrutura/prefect/como-fazer/registrar-metadados.md).
Para o modelo de dados completo, ver a mesma página, seção *Visão geral do
modelo de dados*.

## Tables

### 01 — Todo `ObservationLevel` deve estar ligado a uma `Column`

| Campo | Valor |
|---|---|
| Entidade | `ObservationLevel` ↔ `Column` |
| Vínculo | mapa `observation_levels` em `upload_columns_from_sheet` |
| Severidade | Bloqueia publicação |

**Regra.** Toda entidade de nível de observação deve estar associada a uma
`Column` correspondente da tabela — a coluna que identifica aquela entidade
(ex.: entidade `municipio` ↔ coluna `id_municipio`).

**Por quê.** O `ObservationLevel` descreve a granularidade da tabela: o que cada
linha representa. Sem a coluna correspondente, a granularidade fica
não-auditável e o vínculo entidade→dado se perde — não há como validar que a
tabela de fato observa aquela entidade. O vínculo é firmado no mapa
`observation_levels` passado ao
[`upload_columns_from_sheet`](../../../infraestrutura/prefect/como-fazer/registrar-metadados.md).

### 02 — Toda `Table` deve ter `last update`

| Campo | Valor |
|---|---|
| Entidade | `Table` → `Update` |
| Vínculo | `Update.last_updated` |
| Severidade | Bloqueia publicação |

**Regra.** Toda tabela publicada deve ter um registro `Update` com
`last_updated` preenchido.

- **Pipelines automatizadas:** o `last_updated` é atualizado automaticamente a
  cada execução do pipeline.
- **[Pipelines semi-automatizadas](../../../glossario.md):** o `last_updated` é
  atualizado **manualmente**, no momento da aprovação em prod.

**Por quê.** `last_updated` é a data em que o dado foi efetivamente atualizado
no BD (não o máximo da série). É o sinal de frescor exibido ao usuário no site;
sem ele, a tabela aparece sem data de atualização.

## RawDataSource

### 01 — Toda `Table` deve ter uma `RawDataSource`

| Campo | Valor |
|---|---|
| Entidade | `Table` ↔ `RawDataSource` |
| Vínculo | `Table.raw_data_source_ids` (M2M) |
| Severidade | Bloqueia publicação |

**Regra.** Toda tabela publicada deve estar vinculada a uma
`RawDataSource` correspondente.

**Por quê.** A `RawDataSource` é o ponteiro auditável para a origem do dado:
de onde o dado bruto veio, em que formato, sob qual licença e com qual
disponibilidade. Sem ela, a tabela tratada perde rastreabilidade — não há como
auditar a transformação, reproduzir a captura nem decidir cadência de
atualização. No site público, é o bloco *Fonte original* da tabela.

> A `RawDataSource` é criada **antes** da `Table` (passo 2 da ordem de criação)
> e religada à tabela só na chamada *deferred*, porque o vínculo é M2M e o ID da
> tabela ainda não existe no momento da criação da fonte.

### 02 — Toda `RawDataSource` deve ter `poll`

| Campo | Valor |
|---|---|
| Entidade | `RawDataSource` |
| Vínculo | data de *poll* (última verificação) |
| Severidade | Bloqueia publicação |

**Regra.** Toda fonte bruta deve ter uma data de *poll*: a última data em que
**verificamos** a fonte, mesmo que não houvesse atualização. Comunica ao
usuário "checamos nesta data e não havia novidade".

- **Pipelines automatizadas:** atualizado automaticamente a cada execução.
- **[Pipelines semi-automatizadas](../../../glossario.md):** atualizado
  manualmente na aprovação em prod (mesma regra do `last update` de tabela).

**Por quê.** Distingue "fonte parada" de "fonte não checada". Sem `poll`, o
usuário não sabe se a ausência de dados novos é real ou apenas falta de
verificação.

### 03 — Toda `RawDataSource` deve ter `last update`

| Campo | Valor |
|---|---|
| Entidade | `RawDataSource` |
| Vínculo | data da última atualização real da fonte |
| Severidade | Bloqueia publicação |

**Regra.** Toda fonte bruta deve ter a data da última **atualização real** — o
momento em que a origem de fato mudou —, distinta da data de `poll`. Atualizada
automaticamente nas execuções do pipeline.

**Por quê.** Rastreia quando a origem publicou dados novos pela última vez. É a
base para decidir a cadência de captura e para diferenciar, junto com o `poll`,
uma fonte estagnada de uma fonte apenas não verificada.
