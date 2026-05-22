# Abrir uma issue no repositório `pipelines`

Esta página define como abrir issues em [`basedosdados/pipelines`](https://github.com/basedosdados/pipelines) de forma consistente. O objetivo é tornar o backlog filtrável por tipo, priorizável por label e legível à primeira leitura — sem depender de quem abriu a issue.

## Princípios

1. **Um tipo, um prefixo, uma label.** O prefixo do título e a label de tipo são redundantes de propósito: o prefixo dá leitura rápida na listagem; a label permite filtro e automação.
2. **Título descreve o quê, não o como.** Nome da tabela, do conjunto ou do componente afetado — não a solução proposta.
3. **Português, minúsculas no prefixo, sem placeholders esquecidos** (`<title>`, `<dataset_id>` etc.).

## Vocabulário canônico

Use **exatamente um** dos sete prefixos abaixo. Não combine (`[bugfix]`, `[bug-fix]`, `[fix]` não são aceitos — viram `[bug]` ou `[databug]` conforme o caso).

| Prefixo | Quando usar | Label |
|---|---|---|
| `[bug]` | Defeito em código, tooling, CI, Action ou pipeline (comportamento errado) | `bug` |
| `[databug]` | Defeito nos **dados publicados** (valor errado, nulo indevido, encoding, coluna ausente) | `databug` |
| `[data]` | **Nova** base, conjunto ou tabela a ser adicionada à BD | `data` |
| `[update]` | Atualização programada ou re-materialização de base **já publicada** | `update` |
| `[feat]` | Nova funcionalidade no código do repositório | `feat` |
| `[chore]` | Manutenção, infra, refactor, migração de dependências, ajustes de CI | `chore` |
| `[docs]` | Documentação (README, comentários, docstrings, arquivos `.md`) | `documentation` |

### `[bug]` vs `[databug]` — como decidir

A pergunta-chave: **se o código estivesse perfeito, o problema desapareceria?**

- **Sim** → `[bug]`. O problema está na lógica do código, na pipeline, na Action ou no tooling.
- **Não, o dado de origem ou a tabela publicada estão errados** → `[databug]`.

Exemplos:

- `[databug] br_tse_eleicoes.perfil_eleitorado_secao` — valores faltando na tabela publicada.
- `[databug] encoding na tabela de CNPJ` — bytes corrompidos no dado.
- `[bug] check_metadata: tratar incompatibilidade entre bool e boolean` — código da Action.
- `[bug] table-approve ignora arquivos fora de models/` — comportamento errado do tooling.

### `[data]` vs `[update]` — como decidir

| Pergunta | `[data]` | `[update]` |
|---|---|---|
| A tabela já existe em `basedosdados.<dataset>`? | Não | Sim |
| Precisa escrever raspagem nova? | Sim | Geralmente não |
| Trabalho é projeto (sprint+) ou re-run/fix? | Projeto | Re-run/fix |

Quando a dúvida persistir, default para `[update]` se o `dataset_id` já está publicado.

## Formato do título

```text
[<prefixo>] <descrição curta em português>
```

- Prefixo entre colchetes, **minúsculo**, seguido de **um espaço**.
- Para issues que afetam uma tabela específica, inclua `dataset_id.table_id` na descrição.
- Sem ponto final, sem emoji no título, sem `:` extra.

### Bons exemplos (extraídos do histórico)

- `[databug] br_ms_cnes`
- `[databug] br_tse_eleicoes.perfil_eleitorado_secao`
- `[update] br_me_rais`
- `[data] br_ibge_censo_agropecuario`
- `[chore] otimizar check_metadata para tabelas com muitas colunas`
- `[docs] ajustar guia de uso do TSE`

### A evitar

| Errado | Por quê | Corrigir para |
|---|---|---|
| `[BugFix] world_oecd_pisa__student` | Casing e prefixo fora do vocabulário | `[databug] world_oecd_pisa__student` |
| `[fix] br_inep_ideb__municipio` | `[fix]` é ambíguo | `[databug]` ou `[update]` conforme o caso |
| `fix(check_metadata): tratar incompatibilidade` | Estilo conventional commit | `[bug] check_metadata: tratar incompatibilidade` |
| `Custo de armazenamento e processamento do GCP` | Sem prefixo | `[chore] revisar custo de GCP` |
| `[bug] <title>` | Placeholder do template não preenchido | Substituir `<title>` por descrição real |

## Labels

Toda issue deve ter **uma única label de tipo** (`bug`, `databug`, `data`, `update`, `feat`, `chore`, `documentation`) — geralmente aplicada pelo template, mas confira ao abrir manualmente.

Labels **ortogonais ao tipo** podem ser combinadas livremente:

- `pro` — pipeline prioritária para BD Pro
- `good first issue` — adequada para contribuidores iniciantes
- `suporte` — issue originada de relato de usuário (HubSpot, Discord, e-mail)
- `milestone-pipelines` — issue alocada na milestone vigente da equipe pipelines
- `duplicate`, `question` — uso usual do GitHub

> **Labels com prefixo `[PR]` na descrição** (`check-metadata`, `deploy-flow`, `table-approve`, `test-dev-model`, `conflict`, `hacktoberfest-accepted`) são gatilhos de CI ou marcadores para pull requests — **não devem ser aplicadas a issues**. Ver [Abrir um PR](abrir-pr.md) para detalhes.

> **Nota:** a label `triage` é referenciada pelo template de bug mas **não existe no repo**. Está prevista a remoção dessa referência no próximo PR de revisão dos templates.

## Templates

Sempre que possível, abra a issue a partir de um [template](https://github.com/basedosdados/pipelines/issues/new/choose). O template aplica prefixo e label corretos automaticamente. Os 4 templates atuais (`bug`, `data`, `feat`, `docs`) serão revisados para reduzir o número de campos e adicionar templates para `[databug]`, `[update]` e `[chore]`.

## Quando editar uma issue antiga

Ao tocar em uma issue que não segue o padrão (review, comentário, etc.), aproveite para:

1. Reescrever o título no formato canônico.
2. Aplicar a label de tipo correta — em especial, reclassificar `[bug]` em `[bug]` ou `[databug]` conforme a origem do problema.
3. Remover labels duplicadas ou desatualizadas.

Não é necessário fazer mutirão retroativo — o backlog se normaliza naturalmente conforme as issues são revisitadas.
