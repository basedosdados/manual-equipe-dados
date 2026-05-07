---
tipo: como-fazer
titulo: Como revisar um PR de pipeline
origem: https://github.com/basedosdados/pipelines/wiki/Como-corrigir-um-PR
---
# Como revisar um PR de pipeline

Guia para revisores de Pull Requests no repositório [`basedosdados/pipelines`](https://github.com/basedosdados/pipelines). Aplique este checklist antes de aprovar um PR que adiciona ou modifica uma pipeline.

> O título original do guia era "Como corrigir um PR", mas o conteúdo trata de **revisão**. Renomeado na migração.

## Pré-requisitos

- Acesso de revisor ao repositório `basedosdados/pipelines`
- Acesso ao BigQuery do projeto `basedosdados-staging` para inspecionar dados
- Acesso ao Django de metadados para conferir o registro

## Passos

Percorra as quatro áreas abaixo na ordem. Cada item é uma verificação binária — se algum falhar, deixe comentário no PR e bloqueie o merge até a correção.

### 1. Verificar `schedule.py`

- [ ] A frequência de atualização faz sentido para a fonte
- [ ] Parâmetros e `label` são os de produção
- [ ] Os demais parâmetros estão consistentes
- [ ] Não há parâmetro obrigatório no flow que esteja ausente do schedule

### 2. Verificar `flows.py`

- [ ] Os parâmetros do `flow` estão corretos
- [ ] Os parâmetros dentro do `materialization_flow` estão corretos
- [ ] O `time_delta` na task `update_django_metadata` está igual à query no repositório `queries-basedosdados`
- [ ] `upstream_tasks = wait_upload_table` está presente na task `update_django_metadata`
- [ ] O nome do flow foi linkado ao final com o `schedule` correto
- [ ] O arquivo de flow contém **apenas** tasks, parâmetros e cases — nada de lógica embutida (o flow é orquestração, não código)

### 3. Verificar no BigQuery

- [ ] Os dados têm a cobertura temporal indicada nos metadados
- [ ] Os dados estão bem preenchidos (sem colunas inteiras nulas)

### 4. Verificar nos metadados

- [ ] O conjunto tem fonte externa registrada
- [ ] Todos os campos estão preenchidos. Os únicos opcionais são `Arquivos auxiliares` e `Partições no BigQuery`
- [ ] O nome da tabela é amigável à leitura (acentos, conjunções, capitalização correta)
- [ ] A ordem das colunas no Django bate com a do BigQuery

## Verificação

Se todos os itens acima estão marcados, aprove o PR. Se algum falhou, deixe comentário apontando o item específico do checklist.

## Ver também

- [Como subir metadados de um conjunto e suas tabelas](../../governanca/metadados/como-fazer/subir-metadados.md) — rotina relacionada de subida de metadados
