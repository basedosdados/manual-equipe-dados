# manual-equipe-dados

Documentação como código da equipe dados da Base dos Dados. Fonte oficial e centralizada de processos, infraestrutura, padrões de pipeline, observabilidade e governança.

## Onde está o conteúdo

A documentação vive em [`docs/`](docs/index.md). Comece pelo [índice principal](docs/index.md).

## Lógica da estrutura

Este manual é organizado segundo o framework **[Diátaxis](https://diataxis.fr/)**, criado por Daniele Procida. A premissa do Diátaxis é que toda documentação técnica atende a quatro necessidades distintas, e misturá-las num mesmo documento é a principal causa de docs difíceis de ler:

| Modo | Pergunta do leitor | Orientação |
|---|---|---|
| **Tutorial** | "Como começo?" | Aprendizado guiado, do zero |
| **Como-fazer** | "Como faço X?" | Tarefa específica, leitor já sabe o que precisa |
| **Referência** | "O que é Y?" | Consulta factual de valores, schemas, definições |
| **Explicação** | "Por que é assim?" | Conceitos, decisões, contexto histórico |

A estes quatro modos canônicos, o manual adiciona dois subtipos pragmáticos:

- **Runbook** — subtipo de how-to voltado a resolução de incidentes operacionais. Lido sob pressão, indexado pelo *sintoma*.
- **ADR** (Architecture Decision Record) — subtipo de explicação que registra decisões arquiteturais com contexto, alternativas consideradas e consequências.

### Por que domínio no topo, e não modo

A [documentação oficial do Diátaxis sobre hierarquias complexas](https://diataxis.fr/complex-hierarchies/) recomenda que, em projetos com múltiplos domínios, a estrutura de pastas seja organizada **primeiro por assunto** e só depois pelo modo Diátaxis. A razão: quem chega na doc não pensa "preciso de uma referência", pensa "preciso entender custos da GCP". Forçar o leitor a saber em qual modo sua dúvida cai sobrecarrega a navegação.

Por isso a estrutura deste repositório é:

```
docs/
  <dominio>/              # processos, infraestrutura, pipelines, ...
    index.md              # mapa do domínio
    explicacao/           # "por quê?"
    como-fazer/           # "como faço X?"
    referencia/           # "o que é Y?"
    runbooks/             # "quebrou, e agora?"
    adr/                  # decisões arquiteturais
```

Sub-pastas Diátaxis só existem nos domínios em que o volume justifica. Em domínios menores, os arquivos ficam soltos com prefixo no nome (`como-X.md`, `referencia-Y.md`, `explicacao-Z.md`).

### Convenções derivadas

- **`index.md` em toda pasta** — funciona como mapa do domínio, não como README genérico.
- **Frontmatter obrigatório** com `tipo:` em todo arquivo, permitindo linting e índices automáticos por modo.
- **Uma página por unidade atômica** — uma métrica, uma decisão, um runbook. Não agrupar.
- **Tutoriais só em `onboarding/`** — nos demais domínios eles raramente fazem sentido.

## Como contribuir

Antes de criar ou editar uma página, leia [CONTRIBUTING.md](CONTRIBUTING.md). Ele traz a tabela "se o leitor vai… → modo → template" para escolher o modo certo, além das convenções de naming e do fluxo de PR.

## Renderizar o site localmente

Dependências são gerenciadas com [uv](https://docs.astral.sh/uv/). Instale-o uma vez:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

Para servir a documentação localmente:

```bash
uv sync
uv run mkdocs serve
```

Site servido em <http://127.0.0.1:8000>.

Para adicionar uma nova dependência (ex: um plugin do MkDocs):

```bash
uv add <pacote>
```

Sempre commite o `uv.lock` junto com o `pyproject.toml`.

## Referências

- [Diátaxis — site oficial](https://diataxis.fr/)
- [The Documentation System (Daniele Procida)](https://documentation.divio.com/) — versão anterior do mesmo framework, ainda boa leitura
- [Architecture Decision Records (Michael Nygard)](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions) — formato base dos ADRs deste manual