---
tipo: explicacao
titulo: Trilha do engenheiro de dados
---
# Trilha do engenheiro de dados

## Contexto

A equipe dados recebe pessoas com formações diferentes: algumas chegam sabendo Python e SQL, mas
sem conhecer a infraestrutura da BD; outras já conhecem os dados públicos, mas nunca mantiveram um
pipeline em produção. Esta página descreve em que direção um engenheiro de dados cresce na equipe,
para que a pessoa e quem a acompanha saibam onde ela está e qual é o próximo passo.

## A ideia central

O desenvolvimento acontece em três grandes passos. Cada um leva tempo, em geral anos, e o passo
seguinte depende do anterior.

### 1. Operar bem a BD

A pessoa trabalha com autonomia dentro da infraestrutura que já existe:

- executa pipelines de acordo com os [princípios de tratamento de dados](../../governanca/tratamento-de-dados/explicacao/principios-de-tratamento.md);
- revisa PRs de outras pessoas;
- preenche os metadados, entende e corrige problemas de preenchimento;
- aplica os testes e resolve as falhas de acordo com os nossos princípios;
- traz diagnósticos dos problemas com evidências.

É o passo em que a maior parte da equipe está, e o que esta seção do manual mais detalha. A lista
do que é preciso saber está em [Habilidades do engenheiro de dados](../referencia/habilidades-engenheiro-de-dados.md).

### 2. Operar bem em outros projetos e outras infraestruturas

A pessoa leva o que aprendeu na BD para contextos que não foram montados por nós: outros
orquestradores (Dagster, Airflow), outras APIs (GraphQL), outros ambientes de execução
(Kubernetes). Saber operar em infraestruturas diferentes mostra que a pessoa entendeu os
conceitos, e não só as ferramentas da BD.

### 3. Montar, qualificar e aprimorar a infraestrutura

A pessoa passa a decidir como a infraestrutura da BD funciona: monta um servidor Prefect, propõe
mudanças no fluxo padrão das pipelines, escreve ADRs. Para isso precisa conhecer bem o passo 1,
porque as decisões de infraestrutura mudam o trabalho de toda a equipe.

## Trade-offs e alternativas consideradas

<!-- TODO(Laura): por que três passos e não níveis de cargo (júnior/pleno/sênior)? Por que operar a
BD vem antes de conhecer outras infraestruturas? Como a trilha é usada (1:1, avaliação, onboarding)? -->

## Ver também

- [Habilidades do engenheiro de dados](../referencia/habilidades-engenheiro-de-dados.md) — o que é preciso saber em cada passo
- [Onboarding](../../onboarding/index.md) — trilhas para quem está chegando
