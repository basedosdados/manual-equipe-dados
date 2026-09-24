# Usar o proxy com IP brasileiro

> Contexto e decisão arquitetural completos em [ADR-0005 — VM proxy com IP brasileiro](../../infraestrutura/adr/0005-vm-proxy-ip-brasileiro.md). Referência técnica de como o Squid funciona: [VM proxy com IP brasileiro (Explicação)](../../infraestrutura/prefect/explicacao/vm-proxy-ip-brasileiro.md). Esta página cobre só o dia a dia de quem escreve flows.

## Quando isso importa

Algumas fontes de dados bloqueiam requisições vindas de fora do Brasil. O cluster GKE roda em `us-central1`, então toda saída normal de um pod tem IP americano. O sintoma típico: a fonte funciona perfeitamente rodando local (no Brasil), mas o mesmo flow em produção falha com algo como `ConnectionError: RemoteDisconnected` ou timeout — sem 403 explícito, sem mudança nenhuma de código, só de onde a requisição está saindo.

## Como usar

A variável `BRASIL_PROXY_URL` já chega automaticamente em todo pod do work pool (via o secret `gcp-credentials`, compartilhado) — **não precisa mexer em `flows.py` nem na configuração da deployment**. Usa-se direto os helpers de `pipelines/utils/utils.py`:

```python
from pipelines.utils.utils import brasil_proxy_dict, brasil_proxy_url

# requests
response = requests.get(url, proxies=brasil_proxy_dict(), timeout=30)

# httpx
async with httpx.AsyncClient(proxy=brasil_proxy_url()) as client:
    response = await client.get(url)
```

- Os dois helpers retornam `None` quando `BRASIL_PROXY_URL` não está definida (dev local, por exemplo) — a chamada sai direto, sem proxy, sem precisar de `if`/`else` no código de quem chama.
- `brasil_proxy_dict()` já devolve `{"http": url, "https": url}` pronto pro `proxies=` do `requests`. `brasil_proxy_url()` devolve só a URL, pro `proxy=` do `httpx`.
- Exemplo de referência real: `pipelines/datasets/br_rf_cnpj/utils.py` ([basedosdados/pipelines#2094](https://github.com/basedosdados/pipelines/pull/2094)).

## O que nunca fazer

**Nunca** setar `HTTP_PROXY`/`HTTPS_PROXY` como variável de ambiente global do processo. Isso desviaria tráfego que não tem nada a ver com o bloqueio — upload pro GCS, query no BigQuery, comunicação com a API do Prefect, leitura de segredos do Vault — pela VM do proxy, que é pequena e não foi dimensionada pra isso. Passar o proxy **só explicitamente**, nas chamadas de rede que de fato precisam dele.

## Checklist — quando uma fonte começar a falhar só em produção

1. Confirmar nos logs do Prefect que é mesmo bloqueio de conexão (não outro tipo de erro) — geralmente aparece como `ConnectionError`/`RemoteDisconnected`/timeout, consistente entre execuções, mas funcionando normalmente fora do cluster.
2. Aplicar os helpers acima nas chamadas de rede daquela fonte específica.
3. Testar o flow em `dev` antes de subir pra produção.

## Pontos de atenção

- A VM é uma instância única (`e2-small`) — evite rotear downloads massivos ou muito paralelos por ela sem necessidade real. Detalhamento de capacidade/concorrência e o diagrama de como o Squid funciona: [VM proxy com IP brasileiro (Explicação)](../../infraestrutura/prefect/explicacao/vm-proxy-ip-brasileiro.md).
- Se o proxy cair, todo flow que depende dele falha até a VM voltar. Se isso acontecer, acionar quem cuida da infraestrutura.
