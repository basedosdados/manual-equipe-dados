---
tipo: referencia
titulo: Diagnóstico com evidência
---
# Diagnóstico com evidência

## Resumo

| Campo | Valor |
|---|---|
| O que é | O que conferir antes de afirmar qualquer coisa sobre os dados: atrasos, características inesperadas, valores estranhos |
| Fonte | Fonte original, Prefect e BigQuery |
| Atualização | Quando mudar a infraestrutura de pipelines |
| Dono | Equipe dados |

## Definição

Antes de afirmar algo sobre os dados (em issue, PR, Discord ou resposta a usuário), confira as três
fontes abaixo. A conclusão cita o que foi verificado e onde, para que outra pessoa possa repetir a
checagem.

## Schema / Parâmetros

| Onde conferir | O que olhar | Responde |
|---|---|---|
| Fonte original | Arquivos, API ou página de publicação: qual o período mais recente publicado e quando | A fonte já publicou o dado novo? |
| Prefect | Runs de `Check Update`, `Extract and Load` e `Build and Promote` da tabela: estado, data e logs de erro; deployment pausado ou ativo | O pipeline rodou? Onde parou? |
| BigQuery | Período mais recente em `basedosdados-dev` e em `basedosdados` (partições ou `max` da coluna de data) | O dado chegou em dev? Chegou em prod? |

## Exemplo

Uma conclusão fictícia, no formato esperado:

> A tabela `br_exemplo.microdados` está atrasada porque o `dbt test` falhou em dev no último
> `Build and Promote` (run no Prefect, com o erro no log). A fonte já publicou o mês mais recente,
> e `basedosdados-dev` tem esse mês, mas `basedosdados` ainda está no mês anterior.

## Ver também

- [Práticas de tratamento](../../governanca/tratamento-de-dados/referencia/praticas-de-tratamento.md)
  — as regras de engenharia do tratamento de dados.
- [Como revisar um PR de pipeline](../como-fazer/revisar-pr.md) — o checklist de revisão.
