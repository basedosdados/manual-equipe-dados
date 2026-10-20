# Pipelines

Padrões de design, módulos reutilizáveis, validações e tratamentos específicos das pipelines da equipe dados.

## Explicação

- [Table-approve](explicacao/table-approve.md) — como as tabelas alteradas em um PR são materializadas em prod

## Como fazer

- [Abrir uma issue](como-fazer/abrir-issue.md)
- [Abrir um PR](como-fazer/abrir-pr.md)
- [Atualizar dependências](como-fazer/atualizar-dependencias.md) — Python (`uv`) e dbt packages
- [Rodar um flow localmente](como-fazer/rodar-flow-localmente.md) — Prefect 3, sem precisar de deploy em dev na nuvem
- [Materializar uma tabela em prod](como-fazer/materializar-tabela-em-prod.md) — aplicação do label `table-approve`

## Referência

- [Table-approve](referencia/table-approve.md) — caminhos, parâmetros, regras de seleção e mensagens de log

## Runbooks

- [Table-approve falhou](runbooks/table-approve-falhou.md) — diagnóstico e resolução por mensagem de erro
