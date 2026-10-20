# Abrir um pull request no repositório `pipelines`

> Esta página cobre apenas as **labels específicas de pull requests**. Convenções de título, descrição, revisão e merge serão documentadas em iterações futuras.

## Labels `[PR]`

Pull requests em [`basedosdados/pipelines`](https://github.com/basedosdados/pipelines) usam um conjunto de labels com prefixo `[PR]` na descrição. Elas não devem ser aplicadas a issues — servem como **gatilhos de CI** ou **marcadores de fluxo** específicos de PRs.

| Label | Função | Quando aplicar |
|---|---|---|
| `check-metadata` | Dispara validação de metadados entre BigQuery e API de produção | PRs que alteram colunas, descrições, tipos ou metadados expostos na API |
| `deploy-flow` | Dispara deploy dos flows alterados no work pool `basedosdados-dev` (Prefect 3 staging) | PRs que modificam flows e precisam ser testados em staging antes do merge |
| `table-approve` | Dispara `Table.approve()` no merge, promovendo a tabela de staging para produção | PRs cujo merge deve publicar/atualizar tabelas no projeto `basedosdados` |
| `test-dev-model` | Roda testes DBT nos models modificados em `basedosdados-dev` | PRs que tocam em `models/` e querem validar com dados reais antes do merge |
| `conflict` | Marcador de PR com conflito de merge a resolver | Aplicada (geralmente automaticamente) quando o PR diverge da base |
| `hacktoberfest-accepted` | Marca PRs aprovados na Hacktoberfest | Apenas durante a campanha anual da Hacktoberfest |

## Notas

- As labels `check-metadata`, `deploy-flow`, `table-approve` e `test-dev-model` são **gatilhos** — aplicá-las dispara workflows que custam tempo de CI e podem afetar ambientes (`basedosdados-dev`, `basedosdados`). Aplique apenas quando o PR estiver pronto para o efeito correspondente.
- `conflict` é tipicamente gerenciada por automação. Se aplicada manualmente, sirva como sinal para o autor resolver o rebase/merge antes de seguir a revisão.
- Labels de **tipo de issue** (`bug`, `databug`, `data`, `update`, `feat`, `chore`, `documentation`) **não se aplicam a PRs** — a relação entre PR e issue é feita pelo texto da descrição (`closes #123`).

Para o vocabulário e regras aplicáveis a issues, ver [Abrir uma issue](abrir-issue.md).
