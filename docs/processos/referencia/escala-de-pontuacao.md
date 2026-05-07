---
tipo: referencia
titulo: Escala de pontuação de tarefas
origem: https://github.com/basedosdados/pipelines/wiki/Prioriza%C3%A7%C3%A3o-e-estimativa-de-tarefas
---
# Escala de pontuação de tarefas

Tabela de referência usada na sprint planning para estimar o esforço de cada card. O fluxo que aplica esta escala está em [`como-fazer/planejar-sprint.md`](../como-fazer/planejar-sprint.md).

## Resumo

| Campo | Valor |
|---|---|
| O que é | Escala discreta de pontos por tarefa, indexada por 4 dimensões |
| Onde é aplicada | Sprint planning da equipe dados |
| Limite por sprint | 13 pontos |
| Granularidade | 1, 2, 3, 5, 8, 13 (Fibonacci) |

## Dimensões avaliadas

- **Incerteza** — quão claro é o resultado esperado e o caminho até ele
- **Complexidade** — dificuldade técnica intrínseca da tarefa
- **Conhecimento** — quanto a equipe já domina o que é necessário
- **Volume** — tamanho do escopo (linhas de código, tabelas tocadas, datasets afetados)

## Tabela de pontuação

| Tipo de Atividade | Incerteza | Complexidade | Conhecimento | Volume | Pontos | Tamanho |
|---|---|---|---|---|---|---|
| Conhecida | Baixo | Baixo | Alto | Muito Baixo | 1 | PPP |
| Conhecida | Baixo | Baixo | Alto | Baixo | 2 | PP |
| Conhecida | Baixo | Baixo | Alto | Médio | 3 | P |
| Complexa / Nova | Baixo | Médio | Médio | Médio | 5 | M |
| Complexa / Nova | Baixo | Alto | Baixo | Alto | 8 | G |
| Complexa / Nova | Alto | Alto | Baixo | Alto | 13 | GG |

## Como ler a tabela

Procure a linha cujas dimensões mais se aproximam do card sendo estimado. Se a tarefa cair entre duas linhas, **arredonde para cima** — é mais comum subestimar do que superestimar. Cards >13 não devem entrar inteiros na sprint; quebre antes de pontuar.

## Ver também

- [Como planejar uma sprint](../como-fazer/planejar-sprint.md)
