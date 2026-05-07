---
tipo: adr
titulo: ADR-0001 — Migração da integração DBT de servidor RPC para CLI direta
status: aceito
data: 2025-03-18
origem: https://github.com/basedosdados/pipelines/wiki/Novo-Flow-do-DBT
---
# ADR-0001 — Migração da integração DBT de servidor RPC para CLI direta

## Contexto

A integração do **DBT (Data Build Tool)** com as pipelines do Prefect era feita através de um **servidor DBT-RPC** em execução contínua. As pipelines se conectavam ao servidor através de um cliente intermediário, e toda a comunicação entre o orquestrador e o DBT passava por essa camada RPC.

### Arquitetura anterior

```mermaid
graph TD
    A[Pipeline Prefect] --> B[Cliente DBT RPC]
    B --> C[Servidor DBT RPC]
    C --> D[Modelos DBT]
    C --> E[BigQuery]

    style C fill:#f96,stroke:#333,stroke-width:2px
    style B fill:#69f,stroke:#333,stroke-width:2px
```

Esse desenho trazia atrito operacional:

- Servidor DBT-RPC sempre-ligado, com seu próprio ciclo de deploy e monitoramento.
- Servidor opera sobre uma versão fixa do projeto, dificultando teste de modelos em branches em desenvolvimento.
- Necessidade de manter arquivos de credenciais persistidos em disco no servidor RPC.
- Logs e resultados ficavam no servidor RPC, não no pipeline que originou a chamada — debugging indireto.

## Decisão

Eliminar a camada intermediária RPC e invocar o DBT **diretamente via CLI** dentro do flow do Prefect, usando `dbtRunner`.

### Arquitetura nova

```mermaid
graph TD
    A[Pipeline Prefect] --> B[dbtRunner CLI]
    B --> C[Modelos DBT]
    B --> D[BigQuery]

    style B fill:#69f,stroke:#333,stroke-width:2px
```

### Fluxo de execução

```mermaid
sequenceDiagram
    participant U as Usuário
    participant P as Pipeline Prefect
    participant G as Git Repo
    participant D as DBT CLI
    participant E as Env Vars
    participant B as BigQuery
    U->>P: Executa Flow (opcionalmente com parâmetro branch)
    P->>G: Clona repositório DBT na branch especificada
    P->>E: Verifica variáveis de ambiente para credenciais
    P->>P: Configura profiles.yml
    P->>D: Instala dependências
    P->>D: Executa comando DBT
    D->>B: Processa modelos
    B-->>D: Retorna resultados
    D-->>P: Processa logs e resultados
    P-->>U: Exibe logs detalhados
```

Etapas:

1. Usuário inicia o flow Prefect, podendo especificar uma branch.
2. O repositório DBT é clonado localmente na branch especificada.
3. Credenciais são resolvidas via variáveis de ambiente.
4. `profiles.yml` é configurado (credenciais de ambiente ou arquivo personalizado).
5. Dependências DBT instaladas localmente.
6. Comandos DBT executados diretamente via CLI.
7. Logs e resultados processados e apresentados ao usuário.

### Autenticação e credenciais

```mermaid
graph TD
    A[Início do Flow] --> B{custom_keyfile_path fornecido?}
    B -->|Sim| C[Encontra profiles.yml]
    B -->|Não| F[Verifica variável de ambiente]
    C --> D[Atualiza keyfile no profiles.yml]
    F --> G{Usar credenciais de ambiente?}
    G -->|Sim| H[Atualiza profiles.yml para usar OAuth]
    G -->|Não| I[Mantém configuração original]
    D --> E[Instala dependências DBT]
    H --> E
    I --> E
    E --> J[Executa comando DBT]
```

Opções suportadas:

- `custom_keyfile_path` — caminho para credenciais locais (uso de desenvolvimento).
- `_update_profiles_for_env_credentials` — configura `profiles.yml` para usar OAuth com variáveis de ambiente.
- `use_env_credentials` (padrão: `True`) — controla se a autenticação usa credenciais de ambiente.
- O sistema detecta automaticamente o tipo de autenticação a aplicar.

### Tratamento de erros

```mermaid
graph TD
    A[Execução do comando DBT] --> B{Comando bem-sucedido?}
    B -->|Sim| C[Registra sucesso]
    B -->|Não| D[Extrai mensagens de erro]
    D --> E[Processa arquivo de log]
    D --> F[Verifica erros de compilação]
    D --> G[Verifica erros nos modelos]
    E --> H[Formata mensagens de erro]
    F --> H
    G --> H
    H --> I[Exibe erro detalhado no Prefect UI]
```

Características:

- Extração de erros de múltiplas fontes no resultado do DBT.
- Processamento do arquivo de log para obter informações detalhadas.
- Mensagens formatadas, erros numerados e em linhas separadas.
- Inclusão de contexto adicional (comando executado, diretório de trabalho).
- Exibição na interface do Prefect.

## Consequências

### Positivas

- **Infraestrutura significativamente mais simples**: deixa de existir um componente sempre-ligado para manter, deployar e monitorar.
- **Flexibilidade de branch**: o flow pode invocar o DBT contra qualquer branch/versão do projeto, viabilizando testes de modelos em desenvolvimento.
- **Logs e resultados no próprio pipeline**: o output fica acoplado à execução do flow, simplificando debugging e auditoria.
- **Maior segurança**: elimina a necessidade de manter arquivos de credenciais persistidos em disco no servidor RPC; suporte a OAuth via env vars.
- **Tratamento de erros melhorado**: erros extraídos de múltiplas fontes (resultado do comando, arquivo de log, erros de compilação, erros nos modelos), com formatação clara na UI do Prefect.

### Negativas

- *(não documentadas no material original. Candidatas a investigar: maior tempo de cold-start por execução já que o DBT é invocado do zero a cada flow run; clone do repo Git a cada execução; menor reutilização de cache entre runs.)*

### Neutras

- Cada flow do Prefect que toca DBT precisa ser ajustado para usar o `dbtRunner` no lugar do antigo cliente RPC.
- Adoção de novos parâmetros de flow (`custom_keyfile_path`, `use_env_credentials`, branch).

## Alternativas consideradas

O material original não documenta alternativas formalmente consideradas além da migração de RPC para CLI. Possíveis alternativas que poderiam ter sido avaliadas (e não foram registradas):

- Manter o RPC e endurecer monitoramento/deploy.
- Usar uma solução de orquestração nativa do DBT (DBT Cloud).

## Status

Aceito e implementado. Última revisão da documentação original em 2025-03-18 por Gabriel Pisa.
