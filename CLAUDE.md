# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## O que é este repositório

Documentação como código da equipe dados da Base dos Dados, publicada em
`basedosdados.github.io/manual-equipe-dados` via MkDocs Material. Organizada pelo
framework **Diátaxis**: dentro de cada domínio (`processos`, `infraestrutura`,
`pipelines`, `observabilidade`, `governanca`, `onboarding`), o conteúdo se divide por
**modo** — explicação, como-fazer, referência, runbook, ADR — nunca misturando dois
modos numa mesma página. "Domínio primeiro, modo depois": veja `README.md` e
`CONTRIBUTING.md` para a árvore de decisão completa, naming convention e frontmatter
obrigatório por modo — não repita essas regras de memória, releia os arquivos.

## Comandos

```bash
uv sync                        # instala dependências
uv run mkdocs serve            # serve local com live-reload
uv run mkdocs build --strict   # build de verificação (quebra em links/nav inválidos)
```

**Importante:** o workflow de deploy (`.github/workflows/deploy-docs.yml`) roda
`mkdocs gh-deploy --force` **sem** `--strict`. Isso significa que uma página fora do
`nav:` do `mkdocs.yml`, ou um link relativo quebrado, builda e publica **silenciosamente
sem aparecer no menu do site** — não há rede de segurança no CI para esse erro. A única
forma de pegar isso antes do merge é rodando `uv run mkdocs build --strict` manualmente
ou usando a skill `validar-pagina` (veja abaixo). Isso já aconteceu de verdade neste
repositório (PR #8, `pipelines/como-fazer/rodar-flow-localmente.md` ficou órfã por dias).

## Skills deste repositório — use-as

Este repositório já tem duas skills do Claude Code feitas sob medida para o fluxo de
documentação, em `.claude/skills/`:

- **`nova-pagina`**: conduz a criação de qualquer página nova em `docs/` — escolhe o
  modo Diátaxis certo, usa o template de `docs/_templates/`, aplica naming/frontmatter, e
  **registra a página no `index.md` da pasta e no `nav:` do `mkdocs.yml`**. Use sempre
  que for criar uma página nova, em vez de escrever o arquivo à mão.
- **`validar-pagina`**: audita uma página já criada/alterada contra os padrões
  estruturais do repositório (localização, naming, frontmatter, seções do template,
  `## Ver também`, registro no `index.md` e no `nav:`, `mkdocs build --strict`). Use
  antes de abrir ou aprovar um PR com mudança em `docs/`. Ela só avalia forma, não
  qualidade de conteúdo.

Ambas as skills já trazem a regra: commits e PRs resultantes vão **sem** atribuição do
Claude Code (sem `Co-Authored-By`, sem rodapé "Generated with Claude Code").
