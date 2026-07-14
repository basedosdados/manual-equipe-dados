---
name: nova-pagina
description: Cria uma nova página de documentação seguindo o padrão Diátaxis do manual-equipe-dados (escolhe o modo, usa o template certo de docs/_templates, aplica naming e frontmatter, e registra no index.md e no nav do mkdocs.yml). Use ao adicionar qualquer página nova em docs/.
---

# Criar nova página de documentação

Você vai conduzir, **em português (pt-BR)**, a criação de uma nova página em `docs/`
seguindo o framework **Diátaxis** e as convenções deste repositório. Seu papel é
**impor os padrões** — não apenas escrever o conteúdo, mas garantir que o modo esteja
certo, o arquivo no lugar certo, e que a página seja registrada no `index.md` e no `nav`.

**Fonte da verdade — leia antes de decidir qualquer coisa:**

- `CONTRIBUTING.md` — tabela de decisão de modo, naming, frontmatter, fluxo de PR.
- `README.md` — racional "domínio primeiro, modo depois"; subpasta vs. arquivo plano.
- `docs/_templates/{adr,como-fazer,explicacao,referencia,runbook}.md` — esqueletos.

Nunca reproduza de memória o que está nesses arquivos: leia-os no momento do uso, pois
podem ter mudado.

## Fluxo

### 1. Domínio

Identifique (perguntando ou inferindo do pedido) em qual domínio a página entra:
`processos`, `infraestrutura`, `pipelines`, `observabilidade`, `governanca` ou
`onboarding`. Domínios podem ter subdomínios (ex.: `infraestrutura/prefect/`,
`governanca/metadados/`) — verifique a árvore real em `docs/` antes de assumir.

> Se o conteúdo **não couber em nenhum domínio existente**, **pare**. Não crie pasta
> de domínio nova por conta própria: oriente a pessoa a abrir uma issue primeiro
> (regra do `CONTRIBUTING.md`, passo 1).

### 2. Modo Diátaxis

Aplique a tabela "o que o leitor desta página vai fazer?" do `CONTRIBUTING.md`:

| Se o leitor vai… | Modo | Pasta |
|---|---|---|
| entender por que algo é como é | **explicação** | `<dominio>/explicacao/` |
| executar uma tarefa que já sabe que precisa fazer | **como-fazer** | `<dominio>/como-fazer/` |
| consultar um valor, schema, definição | **referência** | `<dominio>/referencia/` |
| resolver um incidente/problema operacional | **runbook** | `<dominio>/runbooks/` |
| registrar uma decisão arquitetural | **ADR** | `<dominio>/adr/` |
| aprender do zero, em trilha guiada | **tutorial** | `onboarding/` (sem template fixo) |

Se estiver ambíguo, pergunte à pessoa o que o leitor fará com a página — não adivinhe.

### 3. Guarda contra mistura de modos

Se o pedido misturar dois modos (o caso mais comum: **explicar** um conceito **e** dar
os **passos** para executá-lo), **não crie uma página só**. Explique que isso viola o
Diátaxis, proponha dividir em duas páginas (ex.: uma em `explicacao/`, outra em
`como-fazer/`) e faça cada uma linkar a outra na seção "Ver também".

### 4. Template

Leia o template do modo em `docs/_templates/<modo>.md` e use-o como esqueleto.
**Remova a linha** `> **Use este template quando**: …` — ela é orientação do template,
não conteúdo da página. Mantenha exatamente as seções (headings) do template: a ordem
e os títulos são carga estrutural do padrão.

### 5. Onde colocar o arquivo (subpasta vs. arquivo plano)

- Se o domínio (ou subdomínio) **já tem a subpasta do modo** (`<dominio>/<modo>/`),
  coloque o arquivo lá.
- Se o domínio é **pequeno e mantém os arquivos soltos** com prefixo no nome, siga o
  mesmo estilo (`<modo>-X.md` na raiz do domínio). Regra no `README.md`.
- Espelhe sempre o padrão já usado no domínio de destino — verifique a árvore real
  antes de decidir.

### 6. Nome do arquivo

Naming convention do `CONTRIBUTING.md`:

- como-fazer → `como-X.md` (verbo no **infinitivo**: `como-fazer-deploy.md`)
- referência → `referencia-X.md` ou dentro de `referencia/` (ex.: `recursos-pods.md`)
- explicação → `explicacao-X.md` ou dentro de `explicacao/`
- runbook → `runbook-X.md` ou dentro de `runbooks/`
- ADR → `NNNN-titulo-curto.md`, **numerado sequencialmente dentro do domínio**.
  Liste os `NNNN-*.md` já existentes na pasta `adr/` daquele domínio e use o
  **próximo** número, com zero-padding de 4 dígitos (ex.: se existe até `0004`, o novo é `0005`).

### 7. Frontmatter obrigatório

Todo arquivo começa com `---` contendo ao menos `tipo:` e `titulo:`. Por modo:

- **como-fazer / referência / explicação**: `tipo`, `titulo`.
- **ADR**: adicione `status:` (`proposto` | `aceito` | `substituído` | `descontinuado`)
  e `data:` no formato `AAAA-MM-DD` (use a data de hoje). O `titulo` segue
  `ADR-NNNN — <decisão>`. **ADR do domínio `pipelines` também exige `pr:`** — verifique
  ADRs existentes de `pipelines` para o formato.
- **runbook**: adicione `trigger:` (o sintoma/evento que dispara o runbook).

### 8. Conteúdo

Preencha as seções do template junto com a pessoa, respeitando a **pureza do modo**:

- **referência**: factual, densa, sem narrativa; use tabelas.
- **como-fazer / runbook**: passos no imperativo ("Acesse…", "Crie…").
- **explicação / ADR**: narrativo, com o *porquê*, trade-offs e alternativas.

Toda página (exceto `index.md`) **termina com `## Ver também`** linkando ao menos um
modo complementar, com **links relativos** (ex.: `../explicacao/workers.md`).

### 9. Atualizar o `index.md` da pasta

Edite o `index.md` da pasta/domínio de destino (mostre o diff). Dois formatos:

- **index de domínio comum**: adicione um bullet sob o cabeçalho do modo correto
  (`## Explicação — …`, `## Como fazer — …`, `## Referência — …`, `## Decisões (ADR)`),
  no formato `- [Título](caminho-relativo.md) — descrição curta`. Veja
  `docs/infraestrutura/prefect/index.md` como modelo. Crie o cabeçalho do modo se
  ainda não existir.
- **index de ADR**: acrescente uma linha na tabela `| Nº | Título | Status | Data |`.
  Veja `docs/infraestrutura/adr/index.md` como modelo.

### 10. Atualizar o `nav:` do `mkdocs.yml`

Edite o `nav:` do `mkdocs.yml` (mostre o diff). Insira a entrada sob o domínio → grupo
do modo, com **label legível em pt-BR**:

- Grupos por modo: `Como fazer`, `Referência`, `Explicação`, `ADRs`. Crie o grupo se
  ainda não existir, mantendo a ordenação e a indentação já usadas no arquivo.
- Entrada comum: `      - <Label em pt-BR>: <caminho/do/arquivo.md>`
- **ADR** usa o formato com aspas e travessão:
  `      - "ADR-NNNN — Título curto": <caminho>.md`

Não reordene entradas existentes; apenas insira a nova de forma consistente.

### 11. Encerramento

Depois de criar e registrar a página:

1. Instrua a rodar `uv run mkdocs serve` (ou `uv run mkdocs build --strict`) para
   conferir a renderização e checar links quebrados / páginas fora do nav.
2. Lembre do fluxo de PR (`CONTRIBUTING.md`, passo 5), com **Conventional Commits**
   (`docs:` para conteúdo, `feat:` quando fizer sentido).
3. Commits e PRs **sem** assinatura do Claude (sem `Co-Authored-By` e sem rodapé
   "Generated with Claude Code").

## Checklist final

- [ ] Modo Diátaxis correto e **não misturado**.
- [ ] Arquivo no domínio/pasta certo, com naming convention.
- [ ] Frontmatter completo para o modo (incl. `status`/`data`/`trigger`/`pr` quando aplicável).
- [ ] Seções do template preservadas; `> Use este template quando` removido.
- [ ] Seção `## Ver também` com link para modo complementar.
- [ ] `index.md` da pasta atualizado.
- [ ] `nav:` do `mkdocs.yml` atualizado.
