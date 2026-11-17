---
tipo: referencia
titulo: Práticas de tratamento
---
# Práticas de tratamento

## Resumo

| Campo | Valor |
|---|---|
| O que é | Regras de engenharia que garantem os princípios de tratamento no pipeline, no dbt e na revisão de PR |
| Fonte | [Princípios de tratamento de dados](explicacao-principios-de-tratamento.md) |
| Atualização | A cada decisão nova da equipe sobre tratamento |
| Dono | Equipe dados |

## Definição

Cada prática abaixo é verificada na revisão de PR. Um PR que descumpre uma delas não é aprovado
até ser ajustado ou até a exceção ser registrada no README do conjunto.

## Schema / Parâmetros

| Prática | Regra | Caso que motivou |
|---|---|---|
| Entender antes de testar | A chave única vem do que cada linha representa e só depois é testada nos dados. Não se testam combinações até achar uma que passe. Se a chave que faz sentido não passa, isso é o problema a investigar. | PR #2212: várias chaves testadas em `br_rf_cnpj.socios` |
| Zero tolerância de perda de linhas | O pipeline confere linhas lidas contra linhas gravadas (`check_rows_written`, em `pipelines/utils/data_tests/rows.py`) e falha se diferirem. Não existe parâmetro de tolerância. | PR #2212: `ROW_LOSS_TOLERANCE` e uma checagem que comparava com o tamanho do chunk |
| Mudança no dado tem justificativa e custo medido | Toda transformação nova no `.sql` diz por que existe, em qual dos quatro tipos de modificação cabe e quanto processamento acrescenta. Em modelo incremental, o PR diz se precisa de `--full-refresh` e quanto ele custa. | PR #2212: limpeza de caracteres de controle sem justificativa |
| Corrigir a causa, na medida do problema | Quando algo aparece no dado, corrige-se a origem em vez de esconder o sintoma no modelo. Ocorrências raras que vêm da fonte são documentadas, não investigadas. | PR #2212: `distinct` em `socios` (cerca de 300 duplicatas em 28,5 milhões de linhas) |
| Testes baratos e sem repetição | Os testes rodam só na partição nova de cada carga. Um mesmo teste não roda duas vezes na mesma coluna. O custo de um teste novo é medido antes de aplicá-lo em larga escala. | PR #2298 (testes na tabela inteira), PR #2212 (`not_null` repetido), issue [#2322](https://github.com/basedosdados/pipelines/issues/2322) |
| Decisões registradas no README do conjunto | O que foi decidido, as alternativas descartadas e o motivo ficam no README de `pipelines/datasets/<conjunto>/`, não só na descrição do PR. | PR #2212: decisões só na descrição do PR |
| Código novo da fonte entra com proveniência | Um código novo no dicionário cita o documento oficial que o criou e quando ele aparece nos dados. | PR #2320: `tipo_equipe` 83 do CNES (Portaria SAES/MS nº 4.288/2026) |
| Diagnóstico com evidência | Antes de afirmar a causa de um atraso, confere-se a fonte, o Prefect e o BigQuery. A conclusão cita o que foi verificado. | Diagnóstico do Comex Stat de importação (13/11/2026) |

## Exemplo

Registro de uma decisão no README do conjunto (`pipelines/datasets/<conjunto>/README.md`):

```markdown
## Decisões de tratamento

### CEP com 8 dígitos (2026-11)

- **Decisão:** CEPs com menos de 8 dígitos recebem zeros à esquerda; valores só de zeros viram nulo.
- **Tipo de modificação:** padronização para cruzar bases.
- **Por quê:** 82 mil linhas perdiam o zero à esquerda na leitura do CSV.
- **Alternativas descartadas:** manter o valor da fonte (não cruza com o diretório de CEP).
- **Vale a partir de:** `data_referencia` 2026-11; partições anteriores reprocessadas com `--full-refresh`.
```

## Ver também

- [Princípios de tratamento de dados](explicacao-principios-de-tratamento.md) — o porquê destas
  práticas.
- [Testes dbt obrigatórios](../qualidade/referencia/testes-dbt-obrigatorios.md) — os testes que
  aplicam estas práticas em toda tabela.
- [Como revisar um PR de pipeline](../../processos/como-fazer/revisar-pr.md) — o checklist de
  revisão.
