---
tipo: como-fazer
titulo: Como subir metadados de um conjunto e suas tabelas
origem: https://github.com/basedosdados/pipelines/wiki/Rotina-de-subida-de-metadados
---
# Como subir metadados de um conjunto e suas tabelas

Rotina padrão para registrar metadados de um novo conjunto de dados (dataset) e suas tabelas na plataforma da Base dos Dados.

## Pré-requisitos

- Organização, fontes externas e temas já cadastrados no sistema
- Acesso de edição ao Django de metadados
- Tabelas já materializadas no BigQuery (`basedosdados` ou `basedosdados-staging`)

## Passos

### Etapa 1 — Criar dataset e tabelas

1. No campo `Datasets`, criar o conjunto e preencher os metadados básicos.
2. Configurar as **cloud tables** associadas.
3. Adicionar as colunas e seus respectivos metadados.
4. Dentro de cada tabela, completar:
   - Versão
   - Descrição
   - Demais campos obrigatórios

### Etapa 2 — Verificação inicial

1. Acessar [basedosdados.org](https://basedosdados.org).
2. Localizar o conjunto recém-criado.
3. Selecionar as tabelas preenchidas.
4. Validar visualmente que os campos foram corretamente completados.

### Etapa 3 — Preencher metadados complementares

Voltar ao Django e configurar:

- Frequência de atualização
- Cobertura temporal
- Vinculação com diretórios
- Nível de observação
- Partições no BigQuery (em nível de coluna, quando aplicável)

### Etapa 4 — Verificação final

1. Atualizar as tabelas modificadas no BigQuery.
2. Realizar validação completa do preenchimento (todos os campos obrigatórios, ordem das colunas batendo com BQ, nomes amigáveis).
3. Finalizar o processo.

## Verificação

A subida está completa quando:

- O conjunto aparece em [basedosdados.org](https://basedosdados.org) com todas as tabelas listadas
- Todos os campos obrigatórios estão preenchidos
- A ordem das colunas no Django bate com a do BigQuery
- A frequência de atualização e a cobertura temporal estão consistentes com a fonte

## Problemas comuns

- **Campo obrigatório vazio bloqueia publicação** — confira a lista de obrigatórios; só `Arquivos auxiliares` e `Partições no BigQuery` são opcionais.
- **Ordem de colunas divergente entre Django e BQ** — corrija no Django para refletir o BigQuery, não o contrário.

## Ver também

- [Como revisar um PR de pipeline](../../../processos/como-fazer/revisar-pr.md) — checklist relacionado, inclui validação de metadados
