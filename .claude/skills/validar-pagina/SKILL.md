---
name: validar-pagina
description: Valida se uma página de documentação de docs/ respeita os padrões do manual-equipe-dados (localização/naming por modo, frontmatter, seções do template, seção "Ver também", registro no index.md e no nav do mkdocs.yml, links e build). Audita APENAS conformidade estrutural/de padrão — NÃO avalia qualidade, correção ou redação do conteúdo. Use ao revisar uma página nova ou alterada antes de abrir/aprovar PR.
---

# Validar página de documentação

Você vai **auditar** uma página em `docs/` e reportar, item a item, se ela respeita os
padrões deste repositório. Seu papel é ser um **linter de padrões**, não um revisor de
conteúdo.

**Escopo — leia com atenção:**

- ✅ Valida **forma**: localização, naming, frontmatter, seções do template, `Ver
  também`, registro no `index.md` e no `nav`, links relativos, build.
- ❌ **Não** valida conteúdo: não julgue se o texto está correto, completo, bem escrito,
  técnico o suficiente ou factualmente certo. Não reescreva prosa. Não opine sobre
  qualidade. Se um heading do template existe e está preenchido com *algo*, isso
  **passa** — o que está escrito dentro dele não é seu assunto.

**Fonte da verdade — releia no momento do uso (podem ter mudado):**

- `CONTRIBUTING.md` — decisão de modo, naming, frontmatter, fluxo.
- `README.md` — subpasta vs. arquivo plano; "domínio primeiro, modo depois".
- `docs/_templates/{adr,como-fazer,explicacao,referencia,runbook}.md` — esqueletos
  (as seções esperadas por modo).

## Entrada

Identifique **o que validar**:

- Um caminho explícito (ex.: `docs/infraestrutura/prefect/referencia/recursos-pods.md`).
- Ou, se nada for dito, as páginas `.md` **alteradas/adicionadas** no diff atual
  (`git status --short` / `git diff --name-only`), ignorando `docs/index.md` e
  `docs/_templates/`.

Valide cada página de forma independente e produza um relatório por arquivo.

## Verificações

Rode as checagens abaixo. Cada uma resulta em **✅ passa**, **❌ falha** ou **⚠️ n/a**
(quando a regra não se aplica ao modo). Para toda falha, aponte o **arquivo:linha** ou o
trecho exato e **o que o padrão exige** — sem sugerir mudança de conteúdo.

### 1. Modo e localização

- Determine o modo da página pelo `tipo:` do frontmatter e pela pasta em que está.
- Confirme que a pasta corresponde ao modo, conforme a árvore real do domínio:
  `explicacao/`, `como-fazer/`, `referencia/`, `runbooks/`, `adr/`, ou arquivo plano
  com prefixo (`como-X.md`, `referencia-X.md`…) em domínios que seguem esse estilo.
- Confirme que o `tipo:` do frontmatter **bate** com a pasta/prefixo (ex.: um arquivo
  em `referencia/` deve ter `tipo: referencia`).

### 2. Naming convention

Valide o nome do arquivo contra o `CONTRIBUTING.md`:

- como-fazer → `como-X.md` (verbo no infinitivo) **ou** dentro de `como-fazer/`.
- referência → `referencia-X.md` **ou** dentro de `referencia/`.
- explicação → `explicacao-X.md` **ou** dentro de `explicacao/`.
- runbook → `runbook-X.md` **ou** dentro de `runbooks/`.
- ADR → `NNNN-titulo-curto.md` com **4 dígitos** e numeração **sequencial sem buraco**
  dentro da pasta `adr/` do domínio (liste os `NNNN-*.md` existentes e confira que o
  novo é o próximo número).

### 3. Frontmatter

- Abre com bloco `---` … `---` no topo.
- Contém no mínimo `tipo:` e `titulo:`.
- `tipo:` é um dos válidos: `tutorial`, `como-fazer`, `referencia`, `explicacao`,
  `runbook`, `adr`.
- Campos extras obrigatórios por modo:
  - **ADR**: `status:` (`proposto` | `aceito` | `substituído` | `descontinuado`) e
    `data:` em `AAAA-MM-DD`; `titulo` no formato `ADR-NNNN — <decisão>`.
    ADR do domínio **`pipelines`** também exige `pr:` (confira ADRs de pipelines).
  - **runbook**: `trigger:`.
- Valide **presença e formato** dos campos, não o mérito do valor (não julgue se o
  título é "bom", só se existe e segue o formato).

### 4. Seções do template

- Leia o template do modo em `docs/_templates/<modo>.md` e extraia a lista de headings.
- Confirme que a página **preserva as mesmas seções (títulos e ordem)** do template.
  Sinalize seções do template ausentes ou fora de ordem. (Não valide o que está escrito
  dentro delas.)
- Confirme que a linha `> **Use este template quando**: …` foi **removida**.
- Modos sem template fixo (tutorial em `onboarding/`) pulam esta checagem (⚠️ n/a).

### 5. Seção `## Ver também`

- Toda página exceto `index.md` **termina com `## Ver também`**.
- A seção contém ao menos um link, e os links internos são **relativos**
  (ex.: `../explicacao/workers.md`), não absolutos nem URLs do site publicado.
- Valide existência e forma dos links, não se o alvo é "o mais relevante".

### 6. Registro no `index.md` da pasta

- A página está listada no `index.md` do domínio/pasta.
  - index comum: um bullet `- [Título](caminho-relativo.md) — …` sob o cabeçalho do
    modo correto.
  - index de ADR: uma linha na tabela `| Nº | Título | Status | Data |`.
- O caminho relativo no link **resolve** para o arquivo.

### 7. Registro no `nav:` do `mkdocs.yml`

- Existe uma entrada apontando para o arquivo, sob o domínio → grupo do modo
  (`Como fazer`, `Referência`, `Explicação`, `ADRs`).
- O caminho no `nav` **existe** em `docs/` (atenção a renames — o caminho não pode
  apontar para o nome antigo).
- ADR usa o formato `"ADR-NNNN — Título curto": <caminho>.md`.

### 8. Links e build

Rode os checks mecânicos (com `uv run` quando o projeto usa uv):

- `uv run mkdocs build --strict` — deve terminar sem *warnings* de link quebrado nem de
  página fora do `nav` **atribuíveis à página validada**. Warnings pré-existentes em
  outros arquivos não são falha desta página (mencione-os como contexto, à parte).
- Verifique que todos os links relativos da página resolvem para arquivos existentes.

## Relatório

Para cada página, imprima um bloco assim (só a forma; adapte os itens ao modo):

```
### <caminho/da/pagina.md>  —  modo: <modo>

- [✅] Localização e pasta coerentes com o modo
- [✅] Naming convention
- [❌] Frontmatter — falta `data:` (ADR exige `data: AAAA-MM-DD`)
- [✅] Seções do template preservadas; `Use este template quando` removido
- [❌] `## Ver também` — ausente (toda página deve terminar com essa seção)
- [✅] Registrada no index.md da pasta
- [⚠️] nav do mkdocs.yml — n/a (verificar)
- [✅] Build --strict sem warnings atribuíveis a esta página

Veredito: 2 falhas de padrão a corrigir.
```

Termine com um **resumo** consolidado (quantas páginas, quantas passaram 100%, lista das
falhas). Se **tudo passar**, diga explicitamente que a página está conforme os padrões —
**sem** comentar a qualidade do conteúdo.

## Correções (opcional, sob confirmação)

Se a pessoa pedir para **corrigir** as falhas encontradas, faça **apenas ajustes de
forma/padrão** (mover/renomear arquivo, completar frontmatter, adicionar a seção
`Ver também` vazia com placeholder de link, inserir a entrada no `index.md`/`nav`,
corrigir caminho de link). **Nunca** escreva ou reescreva o conteúdo das seções — isso é
trabalho da skill `nova-pagina` ou da pessoa autora. Mostre o diff de cada correção.

Commits/PRs que resultem daqui vão **sem** assinatura do Claude (sem `Co-Authored-By` e
sem rodapé "Generated with Claude Code").

## Checklist final da auditoria

- [ ] Modo detectado e coerente com pasta/prefixo e `tipo:`.
- [ ] Naming convention respeitada (ADR: numeração sequencial de 4 dígitos).
- [ ] Frontmatter presente e completo para o modo.
- [ ] Seções do template preservadas; `> Use este template quando` removido.
- [ ] `## Ver também` presente com link relativo.
- [ ] Registrada no `index.md` da pasta.
- [ ] Registrada no `nav:` do `mkdocs.yml`, com caminho existente.
- [ ] `mkdocs build --strict` sem warnings atribuíveis à página.
- [ ] Relatório emitido **sem** julgamento de conteúdo.
