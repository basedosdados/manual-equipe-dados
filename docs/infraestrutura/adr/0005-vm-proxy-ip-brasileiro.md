---
tipo: adr
titulo: ADR-0005 — VM proxy com IP brasileiro pra fontes que bloqueiam IP estrangeiro
status: aceito
data: 2026-09-23
origem: basedosdados/iac#155, basedosdados/iac#156
---
# ADR-0005 — VM proxy com IP brasileiro pra fontes que bloqueiam IP estrangeiro

## Contexto

O cluster GKE de produção roda em `us-central1`, então todo tráfego de saída dos flows sai com IP americano. Algumas fontes de dados bloqueiam requisições vindas de IP não-brasileiro — o caso real que motivou essa decisão foi `br_rf_cnpj` (Receita Federal): as 5 sub-flows do dataset passaram a falhar em produção com `ConnectionError: RemoteDisconnected` (a fonte derruba a conexão, sem retornar um 403 explícito), enquanto a mesma chamada funcionava normalmente rodando localmente, no Brasil.

Duas soluções foram avaliadas:

- **Cluster GKE novo em São Paulo** (`southamerica-east1`): resolveria de forma "nativa" — qualquer pod ali já sairia com IP brasileiro — mas exige manter um cluster inteiro à parte: taxa de gerenciamento fixa (~US$73/mês só de controle, antes de qualquer node), upgrades, observabilidade própria.
- **VM proxy** (`e2-small` com Squid, também em `southamerica-east1`): só a requisição específica bloqueada passa pela VM; o resto do sistema (GKE, GCS, BigQuery, Prefect) continua exatamente como está.

## Decisão

Usar uma **VM proxy** (`e2-small`, Squid, `southamerica-east1-a`) em vez de um cluster novo.

O Squid roda como forward proxy autenticado (usuário/senha), sem interceptação de TLS — pra HTTPS, ele só abre um túnel `CONNECT` e repassa bytes; o handshake TLS acontece direto entre o pod e o destino, o proxy nunca vê o conteúdo da requisição.

A credencial (`BRASIL_PROXY_URL`) chega em todo pod do work pool via o secret `gcp-credentials`, já injetado globalmente — não é preciso configurar nada por flow. Cada dataset decide, no próprio código, se e onde usa o proxy (ver [Usar o proxy com IP brasileiro](../../pipelines/como-fazer/usar-proxy-ip-brasileiro.md)).

## Consequências

### Positivas
- Custo bem menor: ~US$24/mês estimado, contra ~US$95-115/mês do cluster novo (~1/4 a 1/8).
- Não exige manter infraestrutura de cluster adicional (upgrades, observabilidade própria, node pools).
- Fácil de desligar/redimensionar se o padrão de uso mudar — é só uma VM.

### Negativas
- Ponto único de falha: é uma VM só (`e2-small`). Se ela cair, toda chamada que depende dela falha até a VM voltar.
- Não escala pra volume alto: se muitas fontes/datasets passarem a depender desse proxy ao mesmo tempo, o limite prático seria a banda de rede de uma única VM pequena, não o número de conexões que o Squid aceita.
- Cada dataset precisa aplicar o proxy explicitamente no código das chamadas que precisam dele — não é automático (decisão deliberada, ver Alternativas).

### Neutras
- A variável `BRASIL_PROXY_URL` chega em todo pod via secret compartilhado, então tecnicamente está disponível pra qualquer flow — mas só tem efeito se o código explicitamente decidir usá-la.

## Alternativas consideradas

- **Cluster GKE novo em São Paulo** — descartada por custo (~4-8x mais caro) e complexidade de manter mais um cluster, sem workload suficiente pra justificar isso hoje.
- **Setar `HTTP_PROXY`/`HTTPS_PROXY` como env var global do pod** — descartada: desviaria tráfego não relacionado (upload pro GCS, query no BigQuery, heartbeat da API do Prefect, leitura do Vault) pela VM pequena, sem necessidade e com risco de sobrecarregá-la.
- **Work pool dedicado no Brasil** (worker rodando fisicamente em São Paulo, todo pod desse pool já nasce com IP brasileiro) — considerada como evolução futura, não implementada agora: exigiria manter um worker de produção completo fora do GKE (não só um proxy leve), com acesso a credenciais reais (GCS/BigQuery/Vault), e maior custo — uma VM ligada 24/7 mesmo com uso esporádico, diferente dos Jobs efêmeros do GKE (que escalam pra 0 quando ociosos). Só compensaria se o padrão de "precisa de IP brasileiro" crescer bastante.

## Status

Aceito. Revisitar se o número de fontes bloqueadas por geolocalização crescer o suficiente pra justificar um work pool dedicado.
