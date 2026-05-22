---
tipo: explicacao
titulo: Manual de estilo
---

# Manual de estilo

O **[Manual de estilo](https://basedosdados.org/docs/style_data)** é o documento canônico da Base dos Dados que define **como datasets, tabelas, colunas e diretórios devem ser nomeados, tipados e padronizados** antes de serem publicados. Ele vive no site público da BD e é a referência normativa que todos os colaboradores — equipe interna ou voluntários — devem seguir.

## O que ele cobre

Em linhas gerais, o manual estabelece convenções para:

- **Nomenclatura** de conjuntos (`<pais>_<orgao>_<nome>`), tabelas (`<entidade>`) e colunas (snake_case, sem acentos, sem abreviações ambíguas).
- **Tipagem** das colunas no BigQuery (quando usar `STRING` vs `INT64`, `DATE` vs `STRING`, etc.).
- **Padronização de valores** — unidades, formatos de data, códigos de UF/município, chaves de cruzamento entre bases.
- **Estrutura de diretórios** dos datasets dentro do repositório.
- **Cobertura temporal e espacial** descritas nos metadados.
- **Tabelas auxiliares** (dicionários, diretórios) e quando criá-las.


Em oturas palavras,  as convenções do manual de estilo garantem a usabilidade dos dados pelos usuários finais mediante a padronização de todo o acervo. 

## Quando consultar

- Antes de **criar uma tabela nova** (decidindo nome, partições, colunas).
- Ao **revisar um PR** de dataset, especialmente os que carregam `check-metadata` ou `table-approve`.
- Ao **preencher metadados** no backend antes de marcar a tabela como `under_review`.
- Quando houver dúvida sobre **renomeações** ou **mudanças de tipo** que quebram contratos com consumidores do pacote Python.

## Ver também

- [Manual de estilo (site público da BD)](https://basedosdados.org/docs/style_data) — fonte canônica.
- [Glossário — Manual de estilo](../../glossario.md#m)
- [Fluxo de dados e infraestrutura](../../onboarding/fluxo-de-dados-e-infraestrutura.md) — onde o manual se aplica em cada etapa.
- [Como subir metadados de um conjunto e suas tabelas](como-fazer/subir-metadados.md)
