---
tipo: tutorial
titulo: Setup do repositório de pipelines
---

# Setup do repositório de pipelines

Trilha de onboarding para novos membros da equipe dados e colaboradores configurarem o ambiente local do repositório [`basedosdados/pipelines`](https://github.com/basedosdados/pipelines) — o repositório onde vivem os fluxos de ingestão, transformação e materialização dos dados da BD.

## O que é o repositório de pipelines

O [`basedosdados/pipelines`](https://github.com/basedosdados/pipelines) concentra os dois processos de dados da BD:

- **[Pipelines](../glossario.md#p)** — tabelas cuja atualização é automatizada via [Prefect](https://www.prefect.io/), tipicamente para dados com frequência de atualização alta (diária, horária, etc.).
- **[Pipelines semi-automatizadas](../glossario.md#p)** — código de ELT/ETL em [dbt](https://www.getdbt.com/) presente no repositório, mas sem schedule do Prefect; a execução depende de ação humana e é usada para dados que atualizam esporadicamente.

Tudo o que envolve trazer dado novo para o data lake da BD, transformá-lo e materializá-lo nas camadas `raw` → `staging` → `prod` passa por esse repositório. Tabelas disponibilizadas via assinatura **[BD Pro](../glossario.md#b)** também são materializadas a partir daqui.

> Este manual (`manual-equipe-dados`) documenta **processos, padrões e decisões** da equipe. O código das pipelines em si vive no outro repo.

## Setup do ambiente

O passo a passo de instalação (git, uv, dependências, credenciais do dbt, variáveis de ambiente, hooks de pré-commit) é mantido no próprio repositório de pipelines, para ficar sempre sincronizado com o estado real do código.

**Siga o [CONTRIBUTING.md do repositório de pipelines](https://github.com/basedosdados/pipelines/blob/main/CONTRIBUTING.md).**

Ao final do guia você terá:

- O repositório clonado localmente.
- Dependências Python instaladas via `uv sync`.
- Hooks de pré-commit ativos.
- Dependências do `dbt` instaladas (`uv run dbt deps`).
- Variável `BD_SERVICE_ACCOUNT_DEV` apontando para sua conta de serviço de desenvolvimento.
- Arquivo `auth.toml` configurado em `~/.prefect/` (peça a chave para quem coordena a equipe).

> Usuários de Windows: o setup exige WSL 2 (Ubuntu). O link de instalação está no próprio CONTRIBUTING.

## Próximos passos

Depois do ambiente funcionando:

- Leia o domínio [Pipelines](../pipelines/index.md) deste manual para entender padrões de design, módulos reutilizáveis e convenções de validação.
- Leia o domínio [Processos](../processos/index.md) para entender o fluxo de trabalho da equipe (revisão, deploy, governança).

## Ver também

- [Glossário](../glossario.md) — definições de **Pipeline**, **Pipeline semi-automatizada** e demais termos usados no manual.
- [CONTRIBUTING.md — manual-equipe-dados](../../CONTRIBUTING.md) — como contribuir com **este** manual (diferente do CONTRIBUTING do repo de pipelines).
- [Pipelines / Explicação](../pipelines/explicacao/) — por que as pipelines são organizadas como são.
