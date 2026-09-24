# VM proxy com IP brasileiro

> Decisão arquitetural e alternativas consideradas: [ADR-0005](../../adr/0005-vm-proxy-ip-brasileiro.md). Uso no código: [Usar o proxy com IP brasileiro](../../../pipelines/como-fazer/usar-proxy-ip-brasileiro.md). Esta página é a referência técnica de como a peça funciona.

## Visão geral do fluxo

Só as chamadas de rede que precisam de IP brasileiro passam pela VM — todo o resto do pod (upload pro GCS, query no BigQuery, heartbeat da API do Prefect, leitura do Vault) continua saindo direto do GKE, sem tocar a VM:

```
┌─────────────────────────────── us-central1 (GKE) ───────────────────────────────┐
│                                                                                   │
│   Pod do flow                                                                    │
│                                                                                   │
│   ├── requests.get(url, proxies=brasil_proxy_dict())  ─┐                        │
│   │                                                     │  só as chamadas        │
│   └── httpx.AsyncClient(proxy=brasil_proxy_url())       │  que precisam          │
│                                                          │  de IP brasileiro     │
│   ├── upload pro GCS ────────────────────┐              │                        │
│   ├── query no BigQuery ─────────────────┤  direto,     │                        │
│   ├── heartbeat da API do Prefect ───────┤  sem proxy   │                        │
│   └── leitura do Vault ──────────────────┘              │                        │
│              │                                           │                        │
└──────────────┼───────────────────────────────────────────┼────────────────────────┘
               │                                            │
               │ (egress normal do GKE,                     │ TCP :3128,
               │  IP americano)                             │ Basic Auth
               ▼                                            ▼
        ┌─────────────┐                     ┌──────────────────────────────┐
        │ GCS/BigQuery │                     │   VM brasil-proxy             │
        │ Prefect API  │                     │   southamerica-east1-a        │
        │ Vault        │                     │   Squid (porta 3128)           │
        └─────────────┘                     └───────────────┬────────────────┘
                                                               │
                                                               │ CONNECT (HTTPS) /
                                                               │ GET encaminhado (HTTP)
                                                               │ IP brasileiro
                                                               ▼
                                              ┌──────────────────────────────┐
                                              │  fonte bloqueada              │
                                              │  (ex.: Receita Federal)       │
                                              └──────────────────────────────┘
```

## O que é o Squid, e o que ele faz aqui

Squid é um proxy HTTP/HTTPS de propósito geral. Na VM, ele está configurado como **forward proxy autenticado**, na porta 3128 — sem cache, sem interceptação de TLS (sem "SSL-bump"). Configuração completa (instalada pelo `metadata_startup_script` do Terraform, no primeiro boot):

```
http_port 3128
auth_param basic program /usr/lib/squid/basic_ncsa_auth /etc/squid/passwd
auth_param basic realm proxy
acl authenticated proxy_auth REQUIRED
http_access allow authenticated
http_access deny all
```

- `http_port 3128`: onde o Squid escuta.
- `auth_param basic program ...`: valida usuário/senha contra um arquivo `htpasswd` local (`/etc/squid/passwd`).
- `acl authenticated proxy_auth REQUIRED` + `http_access allow authenticated` + `http_access deny all`: só deixa passar tráfego já autenticado; sem credencial válida, recusa com `407 Proxy Authentication Required`.

Não há allowlist de IP nesse setup (não existe Cloud NAT gerenciado por Terraform hoje, então não dá pra garantir de onde o tráfego do GKE realmente sai) — **a autenticação é o único controle de acesso da VM**.

## Como uma requisição HTTPS atravessa o proxy (sem o Squid nunca ver o conteúdo)

O Squid nunca descriptografa nada. Pra HTTPS, o fluxo é:

1. O cliente (`requests`/`httpx`) manda `CONNECT <host>:443` pro Squid, autenticado.
2. O Squid, depois de validar a autenticação, abre uma conexão TCP crua com o destino e **encana bytes** entre as duas pontas — vira um túnel opaco.
3. O handshake TLS acontece **direto entre o cliente e o servidor de destino**, através desse túnel. O Squid só vê bytes criptografados passando.
4. Do ponto de vista do destino, a conexão TLS chegou do IP da VM — é isso que resolve o bloqueio geográfico.

Pra HTTP puro (não é o caso de nenhuma chamada hoje), o Squid decodificaria e encaminharia ele mesmo, podendo inspecionar cabeçalhos — mas todas as fontes que usam esse proxy até agora são HTTPS.

## Como a credencial chega no pod

```
Terraform (random_password)
        │
        ├─► Secret Manager (brasil-proxy-password) — referência/backup
        │
        └─► BRASIL_PROXY_URL = http://usuario:senha@<ip-da-vm>:3128
                    │
                    ▼  kubeseal
        secret sealed no repositório iac (chave adicionada ao secret
        gcp-credentials já existente, não um secret novo — ver ADR-0005)
                    │
                    ▼  kubectl apply (decriptado só dentro do cluster)
        Secret real "gcp-credentials" no namespace do worker
                    │
                    ▼  envFrom (já configurado no base_job_template
                    │   do work pool, sem mudança adicional)
        Env var BRASIL_PROXY_URL em todo pod novo
                    │
                    ▼
        brasil_proxy_url() / brasil_proxy_dict() (pipelines/utils/utils.py)
```

A credencial nunca passa pelo GitHub Actions nem fica salva na configuração de nenhuma deployment do Prefect — só existe no Secret Manager (registro) e no Secret real do Kubernetes.

## Capacidade e concorrência

O Squid lida com múltiplas conexões concorrentes por padrão — é orientado a eventos, não "uma thread por conexão". Mesmo numa VM pequena (`e2-small`), aguenta dezenas de conexões simultâneas sem esforço, porque com essa config (sem cache, sem interceptação de TLS) cada conexão é só repasse de bytes.

O limite prático não é número de conexões, é **banda de rede da VM**: se muitos downloads grandes acontecerem ao mesmo tempo pela mesma VM, o gargalo seria a banda disponível, não o Squid recusando conexão. Vale reavaliar o tamanho da VM se mais datasets passarem a depender desse proxy simultaneamente.

## O que acontece se a VM cair

As chamadas de rede que dependem do proxy falham com erro de conexão (timeout ou recusa) — nada mais no sistema é afetado, porque nenhuma outra parte do pod depende do proxy. O Prefect já tem retry configurado nas tasks que usam o proxy, então uma instabilidade curta se resolve sozinha; uma queda prolongada exige reiniciar a VM (o startup script reconfigura o Squid do zero no boot).
