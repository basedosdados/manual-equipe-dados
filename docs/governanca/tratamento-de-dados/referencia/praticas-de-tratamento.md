---
tipo: referencia
titulo: Práticas de tratamento
---
# Práticas de tratamento

## Resumo

| Campo | Valor |
|---|---|
| O que é | Regras de engenharia que garantem os princípios de tratamento no pipeline, no dbt e na revisão de PR |
| Fonte | [Princípios de tratamento de dados](../explicacao/principios-de-tratamento.md) |
| Atualização | A cada decisão nova da equipe sobre tratamento |
| Dono | Equipe dados |

## Definição

Cada prática abaixo é verificada na [revisão de PR](../../../processos/como-fazer/revisar-pr.md).
Um PR que descumpre uma delas não é aprovado até ser ajustado ou até a exceção ser registrada no
README do conjunto.

## Schema / Parâmetros

| Prática | Regra | Exemplo |
|---|---|---|
| Entender a chave única antes de testar | A chave única vem do que cada linha representa e só depois é testada nos dados. Não se testam combinações até achar uma que passe. Se a chave que faz sentido não passa, isso é o problema a investigar. | Uma tabela de sócios em que cada linha é um sócio de uma empresa numa data de referência: a chave sai dessa definição, não de tentativa e erro |
| Zero tolerância de perda de linhas | O pipeline chama `check_rows_written` (`pipelines/utils/data_tests/rows.py`) para conferir linhas lidas contra linhas gravadas e falha se diferirem. Não existe parâmetro de tolerância. | Uma checagem com um parâmetro que aceita perder uma fração das linhas |
| Mudança no dado tem justificativa e custo medido | Toda transformação nova no `.sql` diz por que existe, em qual dos quatro tipos de modificação cabe (ou por que é uma exceção) e quanto processamento acrescenta. Em modelo incremental, o PR diz se precisa de `--full-refresh` e quanto ele custa. | Uma limpeza de texto aplicada a várias colunas de uma tabela grande sem justificativa no PR |
| Corrigir a causa, na medida do problema | Quando algo aparece no dado, corrige-se a origem em vez de esconder o sintoma no modelo. Ocorrências raras que vêm da fonte e cuja origem não está na documentação da base são apenas documentadas, e enviamos um pedido de esclarecimento via LAI ao órgão responsável. O que é "raro" depende do contexto; como referência, menos de 0,1% das linhas. O pedido e a resposta ficam registrados no README do conjunto. | Algumas centenas de linhas repetidas em dezenas de milhões, vindas da fonte |
| Testes baratos e sem repetição | Em [tabelas grandes](../../../pipelines/referencia/testes-dbt-obrigatorios.md#definicao), os testes rodam só na partição nova de cada carga, e o custo de um teste novo é medido antes de aplicá-lo. Tabelas menores podem rodar os testes na tabela toda. Em qualquer tabela, um mesmo tipo de checagem não roda duas vezes na mesma coluna. | Um teste de unicidade que relê todas as coletas de uma tabela que cresce a cada carga |
| Decisões registradas no README do conjunto | O que foi decidido, as alternativas descartadas e o motivo ficam no README de `pipelines/datasets/<conjunto>/`, não só na descrição do PR. | Uma lista de decisões que só existe no texto de um PR já fechado |
| Código novo da fonte entra com proveniência | Um código novo no dicionário cita, no README do conjunto, o documento oficial que o criou e quando ele aparece nos dados. Se o significado não está em nenhuma fonte, o código entra com o rótulo "Código <chave>: sem significado nas fontes oficiais (nota da BD)" e um pedido via LAI ao órgão ([como fazer](../como-fazer/como-tratar-codigo-sem-rotulo.md)). | `tipo_equipe` 83 do CNES, criado pela Portaria SAES/MS nº 4.288/2026 e presente nos dados a partir de 2026-09 |

## Ver também

- [Princípios de tratamento de dados](../explicacao/principios-de-tratamento.md) — o porquê destas
  práticas.
- [Testes dbt obrigatórios](../../../pipelines/referencia/testes-dbt-obrigatorios.md) — os testes que
  aplicam estas práticas em toda tabela.
- [Como revisar um PR de pipeline](../../../processos/como-fazer/revisar-pr.md) — o checklist de
  revisão.
- [Diagnóstico com evidência](../../../processos/referencia/diagnostico-com-evidencia.md) — o que
  conferir antes de afirmar algo sobre os dados.
