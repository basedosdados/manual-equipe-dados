---
tipo: explicacao
titulo: Princípios de tratamento de dados
---
# Princípios de tratamento de dados

## Contexto

A Base dos Dados publica dados de centenas de fontes, tratados por pessoas diferentes ao longo do
tempo. Sem um princípio comum, cada PR decide sozinho o quanto mexer no dado: remover linhas
repetidas com `select distinct`, limpar caracteres de texto livre, completar ou não o início de uma
série. Essas decisões mudam o que o usuário recebe. Um `select distinct` no modelo, por exemplo,
apaga linhas da fonte sem que nada no site ou na documentação avise quem usa a tabela.

Esta página define o que a BD faz e o que não faz com o dado da fonte. Ela complementa o
[Manual de estilo](https://basedosdados.org/docs/style_data), que define o formato (nomes, tipos,
valores), e serve de base para a revisão de PR.

## A ideia central

### Princípio geral

**Os dados dentro da BD devem ser o mais próximo possível da fonte original.** O nosso objetivo é
simplificar o acesso, não incluir tratamentos: quem usa o dado entende melhor o contexto e sabe que
informação quer tirar dele.

### As únicas modificações que fazemos

Toda modificação no dado da fonte cabe em um destes quatro tipos. Os detalhes de cada um estão no
[Manual de estilo](https://basedosdados.org/docs/style_data).

1. **Padronização para cruzar bases.** Colunas como `id_municipio`, `cnae` e `cep` seguem o mesmo
   formato em todo o datalake. Isso inclui:
    - adicionar a chave principal da entidade (por exemplo, `id_municipio` numa tabela que só tem
      `id_municipio_tse`);
    - tirar os zeros à esquerda de códigos que são só uma lista de categorias (`01` vira `1`) e
      mantê-los quando os dígitos têm significado (CBO, CEP);
    - remover colunas com o nome de uma entidade que já tem diretório. Por exemplo, a tabela não
      leva a coluna `municipio` quando já tem `id_municipio`: o nome vem do diretório de municípios
      pelo `id_municipio` e fica escrito do mesmo jeito em todo o datalake.
2. **Correção de tipo e formato.** Cada valor é guardado com o tipo e o formato do que ele
   representa, iguais em todo o datalake. Isso inclui:
    - guardar códigos como texto (`STRING`), e não como número. Ao ler o arquivo da fonte, um código
      como o de município (`3304557`) pode ser interpretado como número decimal e virar
      `3304557.0`. Como não se faz conta com um código, tiramos o `.0` e o guardamos como texto:
      `3304557.0` vira `"3304557"`;
    - escrever números, datas e horas num formato único: decimal com ponto (`1234,56` vira
      `1234.56`), data `YYYY-MM-DD` (`31/12/2025` vira `2025-12-31`), hora `HH:MM:SS` e data com
      hora no padrão ISO-8601. Valor ausente fica nulo;
    - escrever categorias com inicial maiúscula e o resto minúsculo (`ENSINO MÉDIO` vira
      `Ensino médio`) e tirar espaços extras dos valores de dicionário. Isso deixa o dado mais
      fácil de usar sem alterar nada essencial. Texto livre (nomes, endereços, descrições) fica
      igual ao original, salvo exceção documentada (ver abaixo).
3. **Padronização de unidades.** Valores na mesma unidade em todo o datalake. Isso inclui:
    - moedas antigas convertidas para a atual (por exemplo, Cruzeiro para Real);
    - unidades com base 1 (`BRL` em vez de `1000 BRL`);
    - proporções entre 0 e 100.
4. **Nomes de coluna e estrutura da tabela.** Renomeamos e organizamos as colunas para que todo o
   datalake seja consistente entre si, seguindo estas seções do
   [Manual de estilo](https://basedosdados.org/docs/style_data):
    - "Nomeação de variáveis": nomes em minúsculas, sem acentos, com os prefixos permitidos;
    - "Ordenamento de variáveis": chaves à esquerda, valores quantitativos à direita;
    - "Formatos de tabelas": a prioridade é manter a tabela no formato da fonte. A passagem de
      `wide` para `long` é exceção, usada só quando simplifica muito o uso da tabela. Exemplo: uma
      tabela com uma coluna para cada ano vira uma tabela com as colunas `ano` e `valor`;
    - "Quais variáveis manter, quais adicionar e quais remover": inclui retirar chaves que podem ser
      obtidas a partir de outra chave da tabela. Por exemplo, `regiao` sai quando a tabela tem
      `sigla_uf`, porque a região vem do diretório de UFs;
    - "Particionamento de tabelas": colunas usadas como partição (`ano`, `sigla_uf`) saem da tabela
      e viram partição, sem perda de informação.

Uma transformação que não cabe em nenhum dos quatro tipos (por exemplo, limpar caracteres de texto
livre ou anular um valor malformado) só entra no dado publicado como **exceção documentada**: o
README do conjunto registra o que foi feito, o motivo e a partir de quando vale.

### Compromissos com o dado publicado

| Compromisso | O que significa na prática | Exemplo |
|---|---|---|
| Fidelidade à fonte | Toda transformação que muda um valor publicado (preencher zeros, anular valor malformado, limpar caracteres) é documentada no README do conjunto. Nada muda em silêncio. | Um CEP que perdeu o zero à esquerda na leitura do arquivo e é completado no modelo |
| Nenhuma linha da fonte se perde | Toda linha lida da fonte chega à tabela publicada. Um descarte só é aceito se for uma regra documentada. Isso inclui as duplicatas da fonte: elas ficam na tabela e são descritas no guia de uso. Duplicata criada pelo nosso tratamento é bug e é corrigida. Conferir as linhas também ajuda a encontrar erros de tratamento: uma diferença entre linhas lidas e gravadas aponta um filtro, um join ou uma partição errada. Quando a tabela muda de formato (de `wide` para `long`), a conferência compara com o número de linhas esperado depois da transformação. | Linhas com uma UF fora da lista esperada que não entram em nenhuma partição, ou um `select distinct` que apaga duplicatas da fonte |
| Cobertura temporal igual à da fonte original | A tabela cobre o mesmo período que a fonte original. Exceção: quando o formato do dado muda muito, o que exigiria muito tratamento, e os dados mais antigos não são tão relevantes para o interesse público. | `br_ms_sih.aihs_reduzidas`: a fonte tem dados desde 2008, a tabela só desde 2009 |
| Série sem buracos | Um período faltando no meio da série é falha. Se a própria fonte não publicou, isso fica documentado no README do conjunto e na descrição da tabela ou no guia de uso. | Issue [#2322](https://github.com/basedosdados/pipelines/issues/2322): teste de buracos temporais |
| Histórico consistente | Uma mudança de tratamento vale para a série inteira. Quando não dá para reprocessar, a documentação diz a partir de quando ela vale. | Um modelo incremental que aplica uma limpeza nova só nas partições futuras |

## Trade-offs e alternativas consideradas

- **Tratar o dado para o usuário.** Seria possível entregar tabelas "limpas": sem duplicatas, com
  texto corrigido, com valores suspeitos anulados. Descartamos porque cada limpeza é uma
  interpretação, e quem usa o dado conhece melhor o contexto para fazê-la. Uma limpeza nossa também
  pode tirar alguma informação que não sabemos que é relevante para quem usa o dado.
- **Copiar a fonte sem nenhuma mudança.** Também descartado: sem padronizar chaves, tipos e unidades,
  duas tabelas que descrevem o mesmo município no mesmo ano não se cruzam. As quatro categorias de
  modificação são o mínimo para o datalake funcionar como um conjunto.
- **Facilidade de uso versus proximidade da fonte.** A capitalização das categóricas e a remoção
  das colunas de nome de entidade afastam o dado da grafia original. Aceitamos porque não mudam o
  essencial e porque o nome padronizado fica disponível no diretório.
- **Consistência do histórico versus custo.** Reprocessar a série inteira de uma tabela grande
  (CNPJ, RAIS) pode ser caro. Quando não compensa, documentar a partir de quando a mudança vale é
  aceito. Esconder a diferença não é.

## Ver também

- [Práticas de tratamento](../referencia/praticas-de-tratamento.md) — como a equipe garante estes
  princípios no pipeline, no dbt e na revisão de PR.
- [Testes dbt obrigatórios](../../../pipelines/referencia/testes-dbt-obrigatorios.md) — o conjunto
  mínimo de testes de toda tabela e o princípio que cada um protege.
- [Manual de estilo](../../metadados/explicacao-manual-de-estilo.md) — o papel do manual na
  infraestrutura da BD.
- [Manual de estilo (site público)](https://basedosdados.org/docs/style_data) — fonte canônica das
  regras de formato.
