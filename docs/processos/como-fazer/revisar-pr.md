---
tipo: como-fazer
titulo: Como revisar um PR de pipeline
---
# Como revisar um PR de pipeline

Guia para revisar pull requests no repositório
[`basedosdados/pipelines`](https://github.com/basedosdados/pipelines) que criam ou alteram uma
pipeline, um modelo dbt ou os testes de uma tabela. Quem revisa aponta o que mudar; quem abriu o PR
faz as correções.

## Pré-requisitos

- Acesso de revisor ao repositório `basedosdados/pipelines`.
- Acesso de leitura ao BigQuery em `basedosdados-dev` e `basedosdados`, à interface do Prefect e
  ao admin do backend de metadados.
- Ter lido os [princípios de tratamento de dados](../../governanca/tratamento-de-dados/explicacao/principios-de-tratamento.md),
  as [práticas de tratamento](../../governanca/tratamento-de-dados/referencia/praticas-de-tratamento.md)
  e os [testes dbt obrigatórios](../../pipelines/referencia/testes-dbt-obrigatorios.md).

## Passos

Percorra as etapas na ordem. Cada item é uma verificação: se falhar, deixe um comentário no PR
apontando o item.

### 1. Entender o que o PR muda

- [ ] A descrição diz o que muda, por quê e qual issue resolve.
- [ ] As decisões de tratamento (o que foi decidido, o motivo, as alternativas descartadas) estão
  no README do conjunto.
- [ ] O PR mexe em um conjunto só. Mudanças em código compartilhado (`pipelines/utils/`, testes
  genéricos, macros) vêm em PR próprio.

### 2. Conferir labels e CI

As labels são gatilhos de CI ([lista completa](../../pipelines/como-fazer/abrir-pr.md)).

- [ ] `test-dev-model` está no PR se ele mexe em `models/`, e o check "Test DBT dev model" passou.
- [ ] `deploy-flow` está no PR se ele mexe em `flows.py`, e o flow foi rodado em dev.
- [ ] `table-approve` só está no PR se a staging de dev já está completa e final. O workflow só
  materializa tabelas cujo `.sql` mudou no PR: se o PR não altera o modelo, peça uma mudança
  mínima no `.sql` ([como funciona](../../pipelines/explicacao/table-approve.md)).
- [ ] Lint e Pyrefly passaram, e o PR não tem conflito com a `main`. Um erro do Pyrefly costuma
  apontar argumento que não existe mais, como um parâmetro removido do `ExtractAndLoad`.
- [ ] Os comentários do CodeRabbit foram lidos e respondidos. Ele aponta riscos reais, mas também
  casos improváveis: avalie cada um contra o funcionamento da pipeline.

### 3. Revisar o flow

Confira contra [como estruturar um flow de dataset](../../pipelines/como-fazer/estruturar-um-flow-de-dataset.md).

- [ ] O `flows.py` só orquestra: nenhuma lógica de tratamento dentro dele.
- [ ] O fluxo é genérico. Particularidades de uma tabela ficam nos dicionários de `constants.py`,
  e não em `if table_id == "..."` espalhados pelo flow, pelas tasks e pelo `utils.py`. Só ficam no
  flow os `if` de checagens que precisam aparecer como etapa da execução.
- [ ] O `check_update` devolve a data de referência real da fonte. Se a fonte não publica data, a
  data de referência é o dia da coleta no fuso de São Paulo.
- [ ] A frequência do `Cron` combina com a frequência da fonte.

### 4. Revisar a extração em Python

- [ ] A pipeline chama `check_rows_written` antes do upload, com as linhas **lidas** da fonte. Se a
  função de tratamento devolve as linhas gravadas, a checagem compara gravadas com gravadas e não
  pega nada.
- [ ] Não existe parâmetro de tolerância de perda de linhas.
- [ ] Checagens sobre o dado da fonte seguem o padrão de `pipelines/utils/data_tests/`: uma função
  que recebe os dados, com docstring (`Args`, `Raises`) e teste unitário, chamada como task no flow
  para aparecer na execução.
- [ ] Uma nova tentativa da task (`retries`) não duplica linhas: arquivos gravados em modo append
  são limpos antes de reprocessar.

### 5. Revisar o modelo dbt

- [ ] Toda transformação no `.sql` cabe em um dos
  [quatro tipos de modificação](../../governanca/tratamento-de-dados/explicacao/principios-de-tratamento.md#as-unicas-modificacoes-que-fazemos)
  ou é uma exceção registrada no README do conjunto, com o motivo e o custo de processamento.
- [ ] Nenhum `select distinct` ou filtro remove linhas da fonte sem uma regra documentada.
- [ ] Cada partição (ou cluster) tem pelo menos 1 GB. Se não tiver, a tabela não deve ser
  particionada. O tamanho de cada partição está em `INFORMATION_SCHEMA.PARTITIONS` (coluna
  `total_logical_bytes`).
- [ ] O `range` de `partition_by` cobre os próximos anos. Um `end` no passado faz as linhas novas
  irem para `__UNPARTITIONED__`, e as consultas filtradas por ano passam a ler a tabela inteira.
- [ ] Em modelo incremental:
    - o filtro usa a forma padrão (`where <coluna> > (select max(<coluna>) from {{ this }})`), sem
      `run_query` no `INFORMATION_SCHEMA`;
    - a coluna de partição da staging não é convertida para `DATE` no filtro, para reduzir os
      custos do BigQuery;
    - o PR diz se a mudança precisa de `--full-refresh` para o histórico ficar consistente, e
      quanto ele custa.
- [ ] O modelo não traz `pre_hook` que apaga as regras de acesso: a service account do dbt tem
  acesso total à tabela, e as regras são recriadas pelo registro de metadados depois do dbt.

### 6. Revisar os testes dbt

Confira o `schema.yml` contra os [testes dbt obrigatórios](../../pipelines/referencia/testes-dbt-obrigatorios.md).

- [ ] A chave única vem do que cada linha representa.
- [ ] Os testes obrigatórios estão lá, sem a mesma checagem repetida na mesma coluna.
- [ ] Em tabela grande, todo teste usa um placeholder de partição, nunca um `where` escrito à mão.
- [ ] `at_least` é `0.99` e as colunas de chave não estão no `ignore_values`. Qualquer limiar mais
  tolerante tem a medição e o motivo no README.
- [ ] Nenhum teste está em `warn` (`severity: warn`, `warn_if`). Todo teste usa a severidade
  padrão, `error`.
- [ ] Toda exceção nos testes (`ignore_values`, tolerâncias) está documentada e
  justificada no README. Para cada uma, avalie se é um problema que podemos resolver no
  tratamento ou se é inerente à fonte e deve passar.

### 7. Conferir o dado em dev

Use o [diagnóstico com evidência](../referencia/diagnostico-com-evidencia.md): fonte, Prefect e
BigQuery.

- [ ] A cobertura temporal em `basedosdados-dev` é a mesma da fonte original, sem buracos no meio
  da série.
- [ ] Nenhuma coluna ficou inteira ou quase inteira nula por erro de leitura ou de cruzamento com
  diretório.
- [ ] Dados de execuções de teste não vão para prod. Se o PR tem `table-approve`, a staging de dev
  só tem as cargas que devem ser publicadas.

### 8. Revisar o README do conjunto

- [ ] Registra as decisões de tratamento, as exceções aos quatro tipos e as medições que
  justificam limiares de teste.
- [ ] Cita o documento oficial de todo código novo no dicionário e quando ele aparece nos dados.
- [ ] Registra o que precisa ser investigado (por exemplo, linhas sem par no diretório) e os
  pedidos de esclarecimento via LAI, com o link da issue `pedido-lai` que acompanha cada um.
- [ ] É direto: explica o que alguém precisa saber para mexer no conjunto, sem narrar o processo.
  Não inclui lista de alternativas de tratamento que não diz por que foram descartadas.

### 9. Conferir os metadados

- [ ] A frequência de atualização no backend combina com a da fonte e com o `Cron`.
- [ ] Em tabela `part_bdpro`, a faixa paga é definida pela coluna certa e as duas coberturas, livre
  e paga, existem no backend.
- [ ] O check "Metadata validation" passou, se o PR tem `check-metadata`.

### 10. Escrever a review

1. Deixe um comentário por linha para cada item que falhou, dizendo o que mudar e por quê. Seja
   direto e cordial.
2. No corpo da review, resuma os pontos em uma lista e reconheça o que já foi resolvido.
3. Envie como **Request changes** se algum item bloqueia o merge e como **Comment** se são só
   sugestões.
4. Não corrija o código no lugar do autor. O processo de correção é um processo de aprendizagem
   para o colega.

## Verificação

O PR pode ser aprovado quando todos os itens das etapas 1 a 9 passam e o CI está verde. Depois do
merge, confira:

- se o `table-approve` materializou a tabela em `basedosdados` com o período esperado;
- se o deployment novo foi ativado pelo admin do backend (todo deployment nasce pausado, e
  despausar direto no Prefect não persiste);
- se a primeira execução agendada em prod terminou sem erro.

## Problemas comuns

- **O `table-approve` não materializou a tabela** — o PR não alterava o `.sql` do modelo, ou o
  label foi aplicado com a staging de dev incompleta.
- **Um teste dbt quebra sem que o código tenha mudado** — a fonte publicou um código novo que não
  está no dicionário. Inclua o código com a proveniência, em vez de afrouxar o teste. Se o
  significado não está em nenhuma fonte, siga
  [Como tratar um código sem rótulo no dicionário](../../governanca/tratamento-de-dados/como-fazer/como-tratar-codigo-sem-rotulo.md).
- **Coletas de teste aparecem em prod** — a staging de dev tinha cargas de teste quando o PR com
  `table-approve` foi mergeado. Apague essas pastas da staging de dev antes do merge.

## Ver também

- [Princípios de tratamento de dados](../../governanca/tratamento-de-dados/explicacao/principios-de-tratamento.md)
  — o porquê das etapas 4 a 8.
- [Testes dbt obrigatórios](../../pipelines/referencia/testes-dbt-obrigatorios.md) — a
  referência da etapa 6.
- [Abrir um pull request](../../pipelines/como-fazer/abrir-pr.md) — as labels de CI.
- [Como estruturar um flow de dataset](../../pipelines/como-fazer/estruturar-um-flow-de-dataset.md)
  — a referência da etapa 3.
- [Table-approve](../../pipelines/explicacao/table-approve.md) — como o merge leva a tabela para
  prod.
- [Diagnóstico com evidência](../referencia/diagnostico-com-evidencia.md) — a referência da etapa 7.
