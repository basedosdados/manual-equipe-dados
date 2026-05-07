# Como contribuir com o manual

Este repositório é a fonte oficial da documentação da equipe dados. Adotamos o framework [Diátaxis](https://diataxis.fr) para que toda página tenha um propósito claro.

## Antes de criar uma página: escolha o modo

Pergunte-se: **o que o leitor desta página vai fazer?**

| Se o leitor vai… | O modo é | Pasta / template |
|---|---|---|
| …entender por que algo é como é | **Explicação** | `<dominio>/explicacao/` — [template](docs/_templates/explicacao.md) |
| …executar uma tarefa específica que já sabe que precisa fazer | **Como-fazer** | `<dominio>/como-fazer/` — [template](docs/_templates/como-fazer.md) |
| …consultar um valor, schema, definição | **Referência** | `<dominio>/referencia/` — [template](docs/_templates/referencia.md) |
| …resolver um incidente ou problema operacional | **Runbook** | `<dominio>/runbooks/` — [template](docs/_templates/runbook.md) |
| …registrar uma decisão arquitetural | **ADR** | `<dominio>/adr/` — [template](docs/_templates/adr.md) |
| …aprender do zero, em trilha guiada | **Tutorial** | `onboarding/` — sem template fixo |

> Se a página parece misturar dois modos (ex: explica e dá passos), **divida em duas páginas e linke uma da outra**. Misturar modos é o erro mais comum e o que torna doc difícil de navegar.

## Convenções

- **Frontmatter obrigatório**: todo arquivo começa com `---` contendo ao menos `tipo:` e `titulo:`.
- **Um arquivo por unidade atômica**: uma métrica, uma decisão, um runbook. Não agrupe.
- **Naming convention**:
  - How-to → `como-X.md` (verbo no infinitivo)
  - Reference → `referencia-X.md` ou dentro de pasta `referencia/`
  - Explanation → `explicacao-X.md` ou dentro de pasta `explicacao/`
  - Runbook → `runbook-X.md` ou dentro de pasta `runbooks/`
  - ADR → `NNNN-titulo-curto.md` (numerado sequencialmente dentro do domínio)
- **`index.md` em toda pasta** — funciona como mapa do domínio, não como README genérico.
- **Linke entre modos**: cada página deve ter uma seção "Ver também" com pelo menos um link para um modo complementar.

## Fluxo para adicionar conteúdo

1. Identifique o domínio (`processos/`, `infraestrutura/`, …). Se não couber em nenhum, abra issue antes de criar pasta nova.
2. Identifique o modo usando a tabela acima.
3. Copie o template de `docs/_templates/` para o lugar certo.
4. Preencha o frontmatter, escreva o conteúdo, atualize o `index.md` da pasta para listar a nova página.
5. Abra PR.

## Migrando conteúdo existente

Ao trazer conteúdo de Coda, wiki ou Google Docs, **não copie em bloco**. Quase sempre o material original mistura modos. Quebre em arquivos por modo antes de comitar.
