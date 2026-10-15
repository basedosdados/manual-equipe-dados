---
tipo: como-fazer
titulo: Como rodar e desenvolver um flow localmente (Prefect 3)
---
# Como rodar e desenvolver um flow localmente (Prefect 3)

Testa um flow (monolítico ou já migrado pro pipeline em estágios) na sua
própria máquina, de ponta a ponta, escrevendo só em `basedosdados-dev` —
sem precisar de deploy em dev na nuvem pra ver se uma mudança funciona.
Tudo roda nativo, sem Docker. Resolve a pendência registrada na issue
[basedosdados/pipelines#2178](https://github.com/basedosdados/pipelines/issues/2178).

## TL;DR

> [!IMPORTANT]
> Só funciona pra um dataset **que você já editou, ou acabou de criar**.
> `up` deploya automaticamente o que for diferente de `origin/main` — sem
> nenhuma mudança (nem um `.py` novo), ele sobe o servidor mas não
> deploya dataset nenhum, e não vai ter o que testar.

```sh
cd pipelines
# 1. edite um dataset existente, ou crie um novo — isso vira o que o `up` deploya

scripts/dev_local.sh up     # 2. sobe tudo, deploya só o que você mudou/criou

# 3. abra http://127.0.0.1:4200, ache o deployment, clique em "Run"

scripts/dev_local.sh down   # 4. quando terminar
```

Depois do primeiro deploy, edite à vontade — não precisa rodar `up` de
novo nem redeployar pra mudança de lógica (ver "Editou código depois do
`up`?" mais abaixo). Disparar pela UI é o jeito principal — não precisa
de nenhum `export` no seu terminal pra isso (ver "Rodando um flow"
abaixo). Tem uma seção bônus mais pra frente pra quem preferir disparar
via `prefect deployment run` na linha de comando.

O resto deste guia explica cada peça, o que fazer quando algo foge do
caminho feliz, e por que o sistema é desenhado assim.

## Por que precisa de um servidor, mesmo pra só testar uma função

No Prefect 1, um `Flow` tinha um método `.run()` que executava tudo em
processo, sem depender de nada externo. No Prefect 3 **isso não existe
mais**: qualquer `@task`/`@flow`, mesmo chamado como função Python comum
fora de um deployment, precisa falar com algum backend Prefect — efêmero
(sobe e morre sozinho, lento e instável pra uso repetido) ou um servidor
de verdade. `scripts/dev_local.sh` sobe esse servidor localmente, uma vez
só, e você reusa ele o dia inteiro.

## Setup — `scripts/dev_local.sh up`

Na raiz do repositório `pipelines`:

```sh
scripts/dev_local.sh up
```

Isso faz, nessa ordem:

1. **Carrega o `.env`** do repositório (Vault, credenciais do GCP) —
   automático, não precisa sourcear nada você mesmo.
2. **Sobe um servidor Prefect 3 local** (SQLite, sem precisar de
   Postgres) em `http://127.0.0.1:4200` — UI e API no mesmo
   processo/porta. Estado isolado em `.prefect-local/` (gitignored),
   nunca mistura com sua config real do Prefect.
3. **Cria o work pool `local-dev`** (tipo `process` — roda os flows como
   subprocesso usando o `uv`/venv do próprio repositório) e **sobe um
   worker** ouvindo esse pool.
4. **Deploya os flows genéricos compartilhados** (`build_and_promote`,
   `run_dbt_model`, transferência pra prod) — sempre, mesmo sem mudança
   neles, porque qualquer dataset migrado pro pipeline em estágios
   depende deles pra fechar a cadeia.
5. **Deploya os flows dos datasets que você mudou** — compara seus
   arquivos `.py` contra `origin/main` (commitado ou não, inclusive
   arquivo novo não rastreado) e deploya só o que é diferente. Rápido,
   porque não mexe nos outros ~330 flows do repositório.

Pronto — pra disparar um run pela UI (o jeito principal, ver "Rodando um
flow" abaixo) não precisa de mais nenhum passo. O comando também imprime
uma linha `export PREFECT_API_URL=...`; ela só importa se você for pela
linha de comando (ver a seção bônus no fim deste guia).

### Se você ainda não mudou nada

Se você acabou de trocar de branch e quer testar um dataset que não é
seu, o `up` não deploya nenhum dataset sozinho (só os genéricos). Ou edite
o dataset primeiro, ou force tudo:

```sh
scripts/dev_local.sh up --all
```

Deploya os ~330 flows do repositório inteiro — bem mais lento, use só
quando precisar testar algo que você não tocou.

### Pra derrubar tudo

```sh
scripts/dev_local.sh down
```

Mata servidor e worker, sem deixar processo órfão. O estado em
`.prefect-local/` fica no disco (reusado no próximo `up`) até você apagar
a pasta manualmente, se quiser recomeçar do zero.

## Rodando um flow

### Caso 1 — flow monolítico (ainda não migrado pro pipeline em estágios)

**Roda pela UI igual a qualquer outro flow** — não tem diferença entre
monolítico e em estágios nesse ponto, o botão "Run" funciona do mesmo
jeito pra qualquer deployment. Se o dataset estava entre os `.py`
alterados quando você rodou `up`, ele já foi deployado: ache o
deployment pelo nome do flow, clique "Run", preencha os parâmetros que o
form pedir (a maioria tem um que trava a execução em dev, nomes variam
por dataset: `materialize_after_dump`, `materialize_to_prod` — deixe
como `False`).

**Alternativa**, pra um teste pontual sem depender de deploy nenhum: um
`@flow` do Prefect 3 também é uma função Python comum, chamável direto
(ainda precisa do servidor local rodando — `up` — só não de
deployment/worker):

```python
from pipelines.datasets.br_fgv_igp.flows import br_fgv_igp__igp_di_mes

br_fgv_igp__igp_di_mes(materialize_after_dump=False, force_run=True)
```

### Caso 2 — pipeline em estágios (`check_update` → `extract_and_load` → `build_and_promote`)

Datasets já migrados (ver `pipelines/utils/stage_dispatch.py`) despacham
o próximo estágio via `run_deployment()`, que **exige um deployment
registrado de verdade** — não dá pra só chamar a função e esperar que o
dispatch aconteça sozinho. Se você editou esse dataset antes do `up`, já
está tudo deployado — é só disparar o `check_update` pela UI:

1. Abra `http://127.0.0.1:4200`.
2. Ache o deployment `check_update: <dataset>__<tabela>` (busca por nome
   funciona).
3. Clique em "Run" → "Run now".

Isso dispara a cadeia completa: `check_update` decide se há dado novo e,
se houver, dispara `extract_and_load` e depois `build_and_promote`
sozinho — testado na prática de ponta a ponta, inclusive com `dbt
run`/`dbt test` reais contra o BigQuery dev, só escrevendo em
`basedosdados-dev`. Acompanhe o progresso e os logs na própria página do
flow run que a UI abre.

Prefere a linha de comando? Tem uma seção bônus mais pra frente
("Disparando pelo `prefect deployment run`") cobrindo isso.

> [!NOTE]
> **Por que o despacho local não é um `run_deployment()` "de verdade"**:
> chamar `run_deployment()` de **dentro de um flow já em execução**
> (necessário pra encadear os estágios) quebra com `422 Unprocessable
> Entity` contra o servidor Prefect local — um bug isolado nesta
> investigação que reproduz igual com qualquer subflow, mesmo sem
> `run_deployment()`/deployment nenhum envolvido (não é erro no seu
> dataset: o mesmo `run_deployment()` funciona normalmente fora de um
> flow, e o servidor aceita o mesmo payload via `curl` direto).
> `stage_dispatch.py` detecta quando está rodando no pool local (tag
> `env:local`) e, só nesse caso, chama a função do próximo estágio
> direto (`.fn()`, por fora do mecanismo de subflow que está quebrado)
> em vez de despachar via deployment — o comportamento real em dev/prod
> não muda em nada.
>
> **Efeito colateral**: localmente a cadeia roda **síncrona**
> (`check_update` só termina depois que `extract_and_load` e
> `build_and_promote` terminarem também, e os três aparecem "achatados"
> num flow run só na UI, não como uma árvore de flows pai/filho), ao
> contrário do fire-and-forget de produção — então o comando de
> `deployment run` pode demorar bem mais que rodar só o primeiro estágio
> sozinho.

### Editou código depois do `up`?

**Na maioria das vezes não precisa redeployar** — o worker local lê o
arquivo do disco a cada execução (nada é empacotado no deploy), então
mudança de lógica dentro do flow/task já vale no próximo run. Testado na
prática: editar o corpo de um flow já deployado e rodar de novo sem
redeployar pega a versão nova do código.

Só precisa redeployar quando a mudança é em **metadado do deployment em
si** — `job_variables`, schedule, `deploy_tags`, a assinatura de
parâmetros do `@flow` — ou pra recuperar um flow que caiu com `503`
durante o deploy de tudo no `up`:

```sh
uv run .github/workflows/scripts/deploy_flows.py --local --files pipelines/datasets/meu_dataset/flows.py
```

Isso lê o código **do disco** (não clona do GitHub, não exige
commit/push) e registra o deployment com as mesmas tags/schedule que o
deploy real usaria, só no pool `local-dev`.

## Bônus — disparando pelo `prefect deployment run`

Se preferir a linha de comando em vez da UI:

```sh
export PREFECT_API_URL="http://127.0.0.1:4200/api"   # ver por que abaixo
uv run prefect deployment run 'check_update: meu_dataset__minha_tabela/dev-check_update' --watch
```

`--watch` acompanha os eventos do run até terminar, direto no terminal
(sem precisar abrir a UI).

### Por que isso precisa de `export PREFECT_API_URL`, e a UI não

`scripts/dev_local.sh up` roda como um **processo separado** — qualquer
`export` que ele faça dentro de si mesmo (pro servidor/worker que ele
sobe) não alcança o seu shell interativo depois que o comando termina.
A UI não sofre disso porque é o **navegador** quem fala com o servidor
local, não o seu terminal. Já o `prefect deployment run` que você digita
é outro processo, iniciado por você, que precisa saber pra onde mandar a
chamada — por isso só ele exige o `export` acima (a própria saída do
`up` já mostra essa linha, pronta pra copiar).

> [!WARNING]
> **Confira antes de exportar.** `.env.example`/`.env` já apontam
> `PREFECT_API_URL` pro local por padrão — mas se alguém no seu terminal
> já tiver exportado a URL real (`https://prefect3.basedosdados.org/api`,
> ex. pra rodar `python -m pipelines.diagnostics health`), ela **vence**
> o que está no `.env`, e qualquer `deployment run`/`deploy_flows.py`
> manual que você rodar depois vai direto pra produção. Rode `echo
> $PREFECT_API_URL` antes se tiver dúvida, ou abra um terminal novo.

## Gotchas conhecidos

> [!IMPORTANT]
> **`check_update` precisa das credenciais do Vault mesmo rodando
> local.** Ele registra o poll no backend (`poll_source_for_update_task`),
> que autentica via `VAULT_ADDRESS`/`VAULT_TOKEN`. O `up` já carrega isso
> do `.env` sozinho — se você ver `AttributeError: 'NoneType' object has
> no attribute 'strip'`, confirme que o `.env` existe na raiz do
> repositório (ver `.env.example`).

> [!WARNING]
> **`PREFECT_API_URL` apontando sem querer pra produção.** `dev_local.sh`
> já se protege sozinho (sempre reatribui a URL local internamente, com
> uma trava que aborta se não for local) — o risco real é só no `export`
> manual que você mesmo faz pra rodar `prefect deployment run` na linha
> de comando. Ver seção "Bônus" acima.

> [!WARNING]
> **`~/.prefect/` pertencendo a outro usuário quebra o SQLite.** Se
> `~/.prefect/` (ou o `PREFECT_HOME` que você usar) pertencer a outro
> usuário (ex. `root`, numa imagem/container compartilhado), o Prefect
> falha com um erro críptico de SQLite (`unable to open database file`)
> ao tentar criar o banco. `dev_local.sh` já evita isso usando
> `.prefect-local/` dentro do próprio repositório, que é sua pasta.

> [!NOTE]
> **`--all` pode ter 1-2 falhas isoladas com `503`.** Com ~330 flows
> deployados de uma vez, é normal 1-2 falharem (o backend aqui é SQLite
> via um único processo, não a API real de produção — sob muita escrita
> concorrente ele engasga). O script avisa qual flow falhou; redeploye só
> ele com `deploy_flows.py --local --files <arquivo>`. No modo padrão (só
> o que mudou) isso é raro, já que é bem menos deployment de cada vez.

### Por que não usar `flow.serve()`

`.serve()` parece mais simples (junta deployment+worker num processo só,
sem precisar de `work-pool create`/`worker start` separados), mas **bate
num bug real** neste repositório: ele serializa o flow com `cloudpickle`
pra rodar num subprocesso, e a versão instalada (`cloudpickle==2.0.0`, de
2022) não é compatível com o bytecode do Python 3.11 — `IndexError:
tuple index out of range` dentro do próprio `cloudpickle`, não no nosso
código. O caminho `from_source(<path local>) + deploy()` + worker
separado (o que `dev_local.sh`/`deploy_flows.py --local` fazem) evita
isso por completo, porque o worker **importa** o código fresco em vez de
serializar um objeto já carregado em memória. Se isso for corrigido um
dia (atualizando `cloudpickle`), `.serve()` pode voltar a ser uma opção
mais simples pra testes rápidos de um único flow — por ora, use o
caminho acima.

## O que isso garante (e o que não garante)

- **Nunca escreve em prod.** `build_and_promote` só promove de verdade
  quando `promote_to_prod=True`, e isso é decidido sozinho a partir de
  onde o `extract_and_load` rodou (`_dev_prefix()` em
  `stage_dispatch.py`) — não é uma opção configurável localmente.
- **Precisa de credencial real** pra ver o upload/dbt acontecerem de
  fato: `GOOGLE_APPLICATION_CREDENTIALS` apontando pra uma conta de
  serviço com acesso de escrita em `basedosdados-dev`, e
  `VAULT_ADDRESS`/`VAULT_TOKEN` (tudo isso já vem do `.env`, ver
  `CONTRIBUTING.md`).
- **Não testa paridade de ambiente com produção.** O worker local roda
  com o Python/sistema da sua máquina, não com a imagem Docker real
  (`Dockerfile.prefect3`). Pra maioria dos bugs (lógica, parsing, data)
  isso não importa — mas se um dataset depender de alguma ferramenta de
  sistema (ex. `br_ms_cnes`/DATASUS compila `blast-dbf` com `git`+`make`
  na hora), um bug ligado especificamente a isso pode não aparecer
  localmente. Pra esse caso raro, rodar o worker com a imagem real de
  produção (volume montado com o código local) é uma ferramenta de debug
  pontual — não o padrão do dia a dia.
- **O despacho entre estágios é local, não via `run_deployment()` real**
  (ver nota no Caso 2 acima) — testa a lógica de cada estágio e o
  encadeamento entre eles de ponta a ponta, só não exercita o mecanismo
  de subflow do Prefect em si.

## Ver também

- `infraestrutura/prefect/explicacao/arquitetura-prefect-3.md`, neste
  manual — por que a arquitetura de produção é como é.
- `pipelines/utils/stage_dispatch.py`, no repositório `pipelines` — a
  implementação do pipeline em estágios.
- `CONTRIBUTING.md`, no repositório `pipelines` — setup geral do
  ambiente, seção "Testar uma pipeline localmente".
- [basedosdados/pipelines#2178](https://github.com/basedosdados/pipelines/issues/2178)
  — issue original, com o checklist completo (local ainda não cobre
  dbt-standalone/`target=dev` fora do fluxo de flows, issue #1605).
