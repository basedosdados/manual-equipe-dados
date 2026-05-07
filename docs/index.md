# Manual da Equipe Dados

Fonte oficial de documentação da equipe dados da Base dos Dados. Centraliza processos, infraestrutura, padrões de pipeline, observabilidade e governança que antes estavam fragmentados em wikis, Google Docs e Coda.

## Como esta documentação é organizada

Seguimos o framework [Diátaxis](https://diataxis.fr): cada página atende a **um modo** de leitura específico. Em vez de escolher um modo no topo, você escolhe primeiro o **domínio** (custos, observabilidade, pipelines…) e dentro dele encontra:

- **Explicação** — entender por que algo é como é (conceitos, decisões, contexto)
- **Como-fazer** — executar uma tarefa específica passo a passo
- **Referência** — consultar valores, schemas, definições de métrica
- **Runbook** — resolver um problema operacional recorrente
- **ADR** — registro de decisão arquitetural

Para criar uma nova página, leia [CONTRIBUTING.md](https://github.com/basedosdados/manual-equipe-dados/blob/main/CONTRIBUTING.md) — ele explica como escolher o modo certo.

## Mapa dos domínios

| Domínio | Para quê | Status |
|---|---|---|
| [Onboarding](onboarding/index.md) | Entrada de novos membros da equipe | 🔴 não iniciado |
| [Processos](processos/index.md) | Fluxo de trabalho da equipe (issues, PRs, suporte) | 🟡 em construção |
| [Infraestrutura](infraestrutura/index.md) | Componentes técnicos: BigQuery, Prefect, GitHub | 🟡 em construção |
| [Pipelines](pipelines/index.md) | Padrões de design, dicionários, tratamento de dados | 🔴 não iniciado |
| [Observabilidade](observabilidade/index.md) | Painéis Metabase/Grafana, métricas, auditoria | 🟡 em construção |
| [Governança](governanca/index.md) | Metadados, qualidade, acessos GCP, custos | 🟡 em construção |

## Convenções

- Toda pasta tem um `index.md` que serve de mapa do domínio.
- Todo arquivo tem frontmatter com `tipo:` (`tutorial`, `como-fazer`, `referencia`, `explicacao`, `runbook`, `adr`).
- Nomes de arquivo carregam o modo quando útil: `como-X.md`, `referencia-Y.md`, `runbook-Z.md`.
- Uma página por unidade atômica: uma métrica, uma decisão, um runbook.

## Glossário

Termos transversais usados em toda a documentação: [glossario.md](glossario.md).
