---
tipo: como-fazer
titulo: Como tratar um código sem rótulo no dicionário
---
# Como tratar um código sem rótulo no dicionário

Passos para quando o teste `custom_dictionary_coverage` acusa um código que está nos dados mas não
tem rótulo no `dicionario` do conjunto. Vale para código novo da fonte e para código antigo que
nunca foi rotulado.

## Pré-requisitos

- Acesso de leitura à staging do conjunto em `basedosdados-dev`.
- Acesso ao CSV do dicionário do conjunto (staging) e, quando existir, ao script que o atualiza
  (por exemplo, `models/<conjunto>/code/update_dicionario.py`).
- Conta no [Fala.BR](https://falabr.cgu.gov.br/), para o pedido via LAI.

## Passos

1. **Confirme que o código vem da fonte.** Procure o código na staging e descarte erro nosso:
   zeros à esquerda perdidos, tipo da coluna diferente do tipo da chave do dicionário (`INT64`
   contra `STRING`), encoding quebrado, valor deslocado de coluna. Anote quantas linhas têm o
   código e a primeira competência em que ele aparece.
2. **Procure o significado nas fontes oficiais**: a documentação da base, os normativos que criam
   os códigos (portarias, resoluções, manuais), as tabelas auxiliares que o próprio órgão publica
   e os sistemas de consulta dele. Guarde a lista do que foi consultado.
3. **Se encontrou o significado**, inclua o código no dicionário com o rótulo oficial e registre no
   README do conjunto o documento que o criou e quando ele aparece nos dados.
4. **Se não encontrou**, inclua o código no dicionário mesmo assim, com o rótulo
   `Código <chave>: sem significado nas fontes oficiais (nota da BD)` (por exemplo,
   `Código 16: sem significado nas fontes oficiais (nota da BD)`). O código vai no próprio
   rótulo para que quem vê só o rótulo (num gráfico ou numa tabela agregada) saiba qual código
   buscar, e o "(nota da BD)" deixa claro que o texto é nosso, não uma categoria do órgão. As
   fontes consultadas ficam no README do conjunto. Mantenha o resto do texto igual em todos os
   conjuntos, para que uma busca por `sem significado nas fontes oficiais` encontre todos os
   casos. Não deixe o código de fora do
   dicionário, não coloque o código no `ignore_values` do teste e não mude o teste para `warn`.
5. **Abra um pedido via LAI** ao órgão responsável pela base, no Fala.BR, pedindo o significado
   do código e desde quando ele é usado. Um pedido pode cobrir vários códigos.
6. **Registre no README do conjunto**, na seção de mudanças do dicionário, uma linha por código:
   tabela e coluna, chave, rótulo, primeira competência, número de linhas, as fontes consultadas
   e, quando for o caso, o número do protocolo e a data do pedido via LAI.
7. **Quando a resposta chegar**, troque o rótulo pelo significado informado e cite a resposta (data
   e protocolo) no README.

## Verificação

- O `custom_dictionary_coverage` passa com severidade `error`.
- O README do conjunto tem, para cada código incluído, a fonte do rótulo ou o pedido via LAI em
  aberto.

## Problemas comuns

- **O código "sem rótulo" some depois de corrigir o tipo ou os zeros à esquerda** — era erro de
  leitura ou de tipo, não código novo. Corrija o tratamento e não inclua nada no dicionário.
- **O teste passou a ficar em `warn` para não bloquear a carga** — nenhum teste fica em `warn`
  (ver [Testes dbt obrigatórios](../../../pipelines/referencia/testes-dbt-obrigatorios.md#definicao)).
  Inclua o código com o rótulo de código sem significado e mantenha o teste em `error`.
- **O mesmo código tem significados diferentes ao longo do tempo** — registre os dois períodos no
  README e use o rótulo vigente na competência mais recente, citando a mudança.

## Ver também

- [Práticas de tratamento](../referencia/praticas-de-tratamento.md) — a regra de que código novo
  entra com proveniência.
- [Princípios de tratamento de dados](../explicacao/principios-de-tratamento.md) — por que o dado
  fica como a fonte publicou.
- [Testes dbt obrigatórios](../../../pipelines/referencia/testes-dbt-obrigatorios.md) — o
  `custom_dictionary_coverage` e a regra de severidade.
- [Como revisar um PR de pipeline](../../../processos/como-fazer/revisar-pr.md) — onde esse ponto
  é conferido.
