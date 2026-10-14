---
tipo: como-fazer
titulo: Como rodar um flow localmente (Prefect 3)
---
# Como rodar um flow localmente (Prefect 3)

Testa um flow (monolítico ou já migrado pro pipeline em estágios) na sua
própria máquina, escrevendo só em `basedosdados-dev` — sem precisar de
deploy em dev na nuvem pra ver se uma mudança funciona. Resolve a
pendência registrada na issue
[basedosdados/pipelines#2178](https://github.com/basedosdados/pipelines/issues/2178).

## Por que precisa de um servidor, mesmo pra só testar uma função

No Prefect 1, um `Flow` tinha um método `.run()` que executava tudo em
processo, sem depender de nada externo. No Prefect 3 **isso não existe
mais**: qualquer `@task`/`@flow`, mesmo chamado como função Python comum
fora de um deployment, precisa falar com algum backend Prefect — efêmero
(sobe e morre sozinho, lento e instável pra uso repetido) ou um servidor
de verdade.

Testado na prática: chamar uma `@task` sem `PREFECT_API_URL` configurado
faz o Prefect tentar subir um servidor efêmero e, dependendo do ambiente,
isso trava com timeout. Com um servidor real rodando, a mesma chamada
funciona de primeira.

## Setup — uma vez só

Na raiz do repositório `pipelines`:

```sh
scripts/dev_local.sh up
```

Isso sobe, tudo nativo (sem Docker):

- Um servidor Prefect 3 local (SQLite, sem precisar de Postgres) em
  `http://127.0.0.1:4200` — UI e API no mesmo processo/porta.
- Um work pool `local-dev`, tipo `process` (roda os flows como
  subprocesso usando o próprio `uv`/venv do projeto).
- Um worker ouvindo esse pool.

O estado fica em `.prefect-local/` (gitignored), isolado da sua config
real do Prefect — não precisa se preocupar em misturar com credencial de
produção.

O comando imprime a variável que você precisa exportar antes de qualquer
teste:

```sh
export PREFECT_API_URL="http://127.0.0.1:4200/api"
```

> [!WARNING]
> **Gotcha real:** se `~/.prefect/` (ou o `PREFECT_HOME` que você usar)
> pertencer a outro usuário (ex. `root`, numa imagem/container
> compartilhado), o Prefect falha com um erro críptico de SQLite
> (`unable to open database file`) ao tentar criar o banco. `dev_local.sh`
> já evita isso usando `.prefect-local/` dentro do próprio repositório,
> que é sua pasta.

Pra derrubar tudo (servidor + worker):

```sh
scripts/dev_local.sh down
```

## Caso 1 — flow monolítico (ainda não migrado)

Um `@flow` do Prefect 3 já é uma função Python comum, chamável direto —
não precisa de deployment nenhum pra esse caso. A maioria dos flows
monolíticos já aceita um parâmetro que trava a execução em dev (nomes
variam por dataset: `materialize_after_dump`, `materialize_to_prod`):

```python
from pipelines.datasets.br_fgv_igp.flows import br_fgv_igp__igp_di_mes

br_fgv_igp__igp_di_mes(materialize_after_dump=False, force_run=True)
```

Isso ainda precisa do servidor local rodando (ver setup acima) — só não
precisa de deployment/worker.

## Caso 2 — pipeline em estágios (`check_update` → `extract_and_load`)

Datasets já migrados (ver `pipelines/utils/stage_dispatch.py`) despacham
o próximo estágio via `run_deployment()`, que **exige um deployment
registrado de verdade** — não dá pra só chamar a função e esperar que o
dispatch aconteça sozinho.

### Deploy pro pool local

Use o mesmo script que já faz deploy de produção
(`.github/workflows/scripts/deploy_flows.py`), com a flag `--local`: ele
lê o código **do disco**, não clona do GitHub — então funciona com
mudanças ainda não commitadas.

```sh
uv run .github/workflows/scripts/deploy_flows.py --local --files pipelines/datasets/meu_dataset/flows.py
```

Isso registra os deployments com as mesmas tags/schedule que o deploy
real usaria, só no pool `local-dev` em vez de `basedosdados-dev`/
`basedosdados` — e como o nome do pool contém "dev", o script já aplica
sozinho as mesmas regras de dev (prefixo `dev-`, sem schedule ativo, tag
`env:dev`).

### Disparar o run

```sh
uv run prefect deployment run 'check_update: meu_dataset__minha_tabela/dev-check_update'
```

Ou pela própria UI (`http://127.0.0.1:4200`), na página do deployment —
o botão "Run" faz exatamente a mesma chamada. Os dois jeitos disparam a
cadeia completa: `check_update` decide se há dado novo e, se houver,
dispara `extract_and_load` sozinho via `run_deployment()` — do mesmo
jeito que aconteceria em produção, só que local e só escrevendo em
`basedosdados-dev`.

### Por que não usar `flow.serve()`

`.serve()` parece mais simples (junta deployment+worker num processo só,
sem precisar de `work-pool create`/`worker start` separados), mas **bate
num bug real** neste repositório: ele serializa o flow com `cloudpickle`
pra rodar num subprocesso, e a versão instalada (`cloudpickle==2.0.0`,
de 2022) não é compatível com o bytecode do Python 3.11 —
`IndexError: tuple index out of range` dentro do próprio `cloudpickle`,
não no nosso código. O caminho `from_source(<path local>) + deploy()` +
worker separado (o que `dev_local.sh`/`deploy_flows.py --local` fazem)
evita isso por completo, porque o worker **importa** o código fresco em
vez de serializar um objeto já carregado em memória.

Se isso for corrigido um dia (atualizando `cloudpickle`), `.serve()`
pode voltar a ser uma opção mais simples pra testes rápidos de um único
flow — por ora, use o caminho acima.

## O que isso garante (e o que não garante)

- **Nunca escreve em prod.** `build_and_promote` só promove de verdade
  quando `promote_to_prod=True`, e isso é decidido sozinho a partir de
  onde o `extract_and_load` rodou (`_dev_prefix()` em
  `stage_dispatch.py`) — não é uma opção configurável localmente.
- **Precisa de credencial real** pra ver o upload/dbt acontecerem de
  fato: `BD_SERVICE_ACCOUNT_DEV` apontando pra uma conta de serviço com
  acesso de escrita em `basedosdados-dev` (mesma credencial que o resto
  do setup do repositório já usa — ver `CONTRIBUTING.md`).
- **Não testa paridade de ambiente com produção**: o worker local roda
  com o Python/sistema da sua máquina, não com a imagem Docker real
  (`Dockerfile.prefect3`). Pra maioria dos bugs (lógica, parsing, data)
  isso não importa — mas se um dataset depender de alguma ferramenta de
  sistema (ex. `br_ms_cnes`/DATASUS compila `blast-dbf` com `git`+`make`
  na hora), um bug ligado especificamente a isso pode não aparecer
  localmente. Pra esse caso raro, rodar o worker com a imagem real de
  produção (volume montado com o código local) é uma ferramenta de debug
  pontual — não o padrão do dia a dia.

## Ver também

- `infraestrutura/prefect/explicacao/arquitetura-prefect-3.md`, neste
  manual — por que a arquitetura de produção é como é.
- `pipelines/utils/stage_dispatch.py`, no repositório `pipelines` — a
  implementação do pipeline em estágios.
- `CONTRIBUTING.md`, no repositório `pipelines` — setup geral do
  ambiente, seção "Testar uma pipeline localmente".
