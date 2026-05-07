bjetivo é criar um manual completo de todas as operações, processos, ferramentas e atividades essenciais da equipe dados







1. Dicionários
  1. dicionários de pipleines precisam ter código autoamtizado para gerar o dicionário
  2. 
2. Definir padrões das etapas de pipleines que fazemos
  1. Ter um retrato fino dos módulos, validações é táticas de abordagens a certos problemas
  2. 



1. uma agenda de um padrão de design de pipelines
    1. “”Refactoring may help out in the long run, but it will definitely slow down the current task. So, you look for quick patches to work around any problems you encounter. This just creates more complexity, which then requires more patches.”  A philosophy of Software Design
    2. 



1. Trtaemnto de dados
  1. Bases de dados geoespaciais e suas especificidades;
    1. Validações de polígonos
    2. bbox



1. Definir padrões para registro de Issues e Backlog
  1. Muitas issues ficam abertas sem conclusões;
    1. Apesar da issue ter sido feita, ela permanece aberta sem um comentário de fechamento para encerrar de fato a tarefa
  2. Muitas issues não tem descrição clara;
  3. 



1. Processos de Correção de Bugs
  1. Formas de Relatos de Bugs
    1. Fórum do Discord
    2. Email da BD
  2. Problemas
    1. Quando recebemos um email, temos o hábito de copiar pro discord e depois criar uma issue
    2. Automação para criar issue dirertamente a partir de uma tag de relato de erro;



Processos de Correção de Erros 



1. Erros em bases anuais grande
  1. Ex. br_me_rais
  2. Essas bases apresentam diversos problemas e não é comum que dúvidas surjam. Quando um erro relacionado a elas for relatado é fundamental verificar os guias de uso e os readmes dos conjuntos para ver se o problema relatado já não foi documentado e, na verdade, é problema da base em si e não do tratamento da BD.







Central de Observabilidade de Dados

A central de acompanhamento da equipe dados fica no dashboard Central da Equipe Dados no Metabase. Este dashboard centraliza um conjunto de indicadores sobre a plataforma estruturado em 5 seções



O sonho de principe seria ter um

1. Pontos ausentes
  1. O Painel de observabilidade deveria ter estatísticas do serviços consumidos ao longo do processo de dados;
    1. Atualmente, não temos apenas metadados do BQ basedosdados
      1. Estão ausentes:
        1. basedosdados-dev
        2. basedosdados-staging
    2. Não temos estatísticas dos buckets
      1. basedosdados-dev
      2. basedosdados
  2. Custos
    1. Não temos estatísticas de custo que nos permitam entender custos detalhados a nível de pipelines;



* Estrutura do Dashboard
Página Panorama de Dados
    * Seção de Estatísticas Gerais do Big Query de Produção (basedosdados)
Volume de Dados Armazenados (TB)
          * O que a métrica mede?
            *  Soma da quantidade total de Terrabytes de dados armazenados no Big Query basedosdados 
          * Utiliza tabela bigquery_tables construida com a pipeline  br_bd_metadados.bigquery_tables que compila dados do INFORMATION_SCHEMA - tabela default com metadados do BQ. 
Quantidade de Datasets no BQ
          * O que a métrica mede?
            *  Contagem da quantidade de datasets no BQ basedosdados
          * Utiliza tabela bigquery_tables construida com a pipeline  br_bd_metadados.bigquery_tables que compila dados do INFORMATION_SCHEMA - tabela default com metadados do BQ. 
Quantidade de Tabelas no BQ
          * O que a métrica mede?
            *  Contagem da quantidade de tabelas no BQ basedosdados
          * Utiliza tabela bigquery_tables construida com a pipeline  br_bd_metadados.bigquery_tables que compila dados do INFORMATION_SCHEMA - tabela default com metadados do BQ. 
    * Seção de estatísticas Gerais do Banco de Produção da  API de Metadados
Quantidade de Datasets
        * O que a métrica mede?
          * Contagem da quantidade de Datasets com Tabelas registradas no banco de metadados de prod. 
        * Utiliza a tabela tables e conta o número de valores únicos de dataset_ids
Quantidade de Tabelas
        * O que a métrica mede?
          * Contagem da quantidade de tabelas  registradas no banco de metadados de prod. 
        * Utiliza a tabela tables e conta o número de valores únicos de table_id
Quantidade de Tabelas com Pipelines
        * O que a métrica mede?
          * Contagem da quantidade de Datasets com Tabelas registradas no banco de metadados de prod. 
        * Utiliza a tabela tables e conta o número de valores únicos de dataset_ids
Quantidade de Tabelas por Status de Publicação
        * Permite termos uma visualização das tabelas que estão ativas e não estão
    * Seção de Acompanhamento da atualização de tabelas
Disclaimer
        * Lógica de cálculo geral de todas as métricas de atualização de tabelas
          * Os metadados utilizados são o latest (timestamp da última atualização bem-sucedida - última atualização na base dos dados), o frequency (frequência de atualização das tabelas) e o slug da entidade (unidade de tempo como day, week, month, quarter ou year). É gerado o date_diff, que representa o "idade" real dos dados em dias em relação à data atual, e o frequency_days, que é o produto do multiplicador da frequência pela base de dias da unidade (ex: 1 mês = 30 dias). O resultado
        * Lógica do status de atenção das pipelines
          * É comum que as tabelas não tenham cronogramas fixos de atualização o que gera falsos positivos para dados desatualizados. Para evitar este problema, foi criado um status de “em observação” que considera um intervalo de dias 
          * A lógica do status_atencao funciona como uma régua de triagem com "zona de escape", desenhada para acomodar a natureza volátil de fontes de dados públicas. O status é atribuído em três níveis: se o atraso acumulado for menor ou igual à frequência esperada, a tabela é considerada "Atualizada". Ao ultrapassar esse limite, a query aplica uma camada de tolerance_days baseada na periodicidade (ex: 7 dias para mensais, 90 dias para anuais); se o atraso estiver dentro dessa janela extra, a tabela entra "Em observação", sinalizando um atraso que, embora existente, ainda é aceitável. Somente quando o tempo de atraso estoura a soma da frequência com a tolerância é que o status muda para "Desatualizado", categorizando o item como um atraso crítico que demanda investigação ou manutenção imediata no pipeline.
        * Lógica de contagem de pipelines e tabelas semi automatizadas
          * A forma de identificação de tabelas com pipelines consiste em filtrar linhas da tabela "public"."table" que contenham valores de pipeline_id not null; Para tabelas semi automatizadas,  a lógica muda para contar null; 
        * Nomenclatura
          * Pipelines → tabelas que possuem automações com prefect
          * Tabelas semi automatizadas → tabelas que possuem código de ELT/ETL registrado no repositório mas não possuem pipelines de automação. Dependem de trabalho humano para serem atualizadas;
Proporção de atualização das Tabelas  por Status e Tabelas em observação e desatualizadas
        * O que a métrica mede?
          * Mede a proporção de todas as tabelas da BD por status de atualização e mostra as tabelas que contém status de desatualizadas e em observação
        * Para detalhes sobre a fórmula de cálculo leia a seção de Disclaimer acima
Proporção de atualização das pipelines por Status e Tabelas com pipelines Em observação e desatualizadas
        * Mede a proporção de  tabelas com pipleines da BD por status de atualização e mostra as tabelas que contém status de desatualizadas e em observação
        * Para detalhes sobre a fórmula de cálculo leia a seção de Disclaimer acima
Proporção de atualização das tabelas semi automatizadas por Status e tabelas semi automatizadas Em observação desatualizadas
        * Mede a proporção de  tabelas semi automatizadas da BD por status de atualização e mostra as tabelas que contém status de desatualizadas e em observação
        * Para detalhes sobre a fórmula de cálculo leia a seção de Disclaimer acima
    * Seção sobre o prefect
Quantidade  de Flows com Schedule Ativa e flows com schedule ativa
        * Mede a quantidade de flows que possuem uma schedule ativa na API do prefect. É construido com o tabela br_bd_metadados.prefect_flows que é extraida diariamente da API do prefect
      * TODO: Inserir acompanhamento da feature de desativação dos flows
Página de Qualidade de Metadados
    * Avaliar a partir dos metadados que já estão presentes
    * Qualidade dos Conjuntos de Dados
      * % de preenchimento dos metadados?
    * Qualidade das Tabelas
      * % de preenchimento dos metadados?
        * Em específico, metadados de atualização e frequência de atualização
        * Validação de compatibilidade de metadados com o Big Query
    * Prefect  -  a api do prefect resulta em campos nulos quando o flow não tem schedule;
      * basta consertar o erro relacionado ao parsing do datetime p/ 
Página Custos
Disclaimer sobre o motívo das séries estarem em USD e BRL e sobre  a perca de dados de 2025
      * Em janeiro de 2025 a conta de faturamento foi modificada para termos as cobranças feitas em BRL ao invés de USD. A pessoa que trocou a conta esqueceu de, após criar a nova, ativar a exportação das tabelas de custos com serviços no GCP. Como resultado, perdemos as tabelas de custo detalhadas para o período de fevereiro de 2025 até setembro de 2025. 
        * Para manter as séries históricas para custos agregadados, foi feito um compilado de dados a partir do painel de billing da GCP e inserido no Big Query na tabela gcp_billing_export_cost_panel_01620F_C10520_CCE6DC; Como resultado, as séries históricas de custos  agregados anteriores a 2025 e entre janeiro de 2025 e setembro de 2025 estão corrigidas e expostas no painel do Metabase; 
        * Contexto do problema pode ser visto na issue
      * Custos em USD
        * A nova conta de faturamento, debitada em reais, contém a taxa de câmbio que a GCP utilizou para converter a fatura de USD para BRL. Isto nos permite ter as séries pós setembro de 2025 em USD ou BRL, com valores exatos.
        * As séries pré janeiro 2025 eram debitadas em USD e não tinham a informação do câmbio utilizado. É possível consertar isto de n formas, como: assumindo uma taxa de câmbio média p/ o período ou taxa de câmbio média mensal. Estes ajustes não foram feitos e as séries foram mantidas em USD.
    * Séries históricas de Custo com a GCP
Custo Mensal com Todos Serviços da GCP (USD)
        * O que a métrica mede?
          * Soma total dos custos com toda infraestrutura da GCP consumida pela Base dos Dados em USD
        * Utiliza tabelas de custos exportadas pelo billing da GCP no conjunto br_bd_indicadores no projeto basedosdados
Custo Mensal com Principais Serviços da GCP (USD)
        * O que a métrica mede?
          * Soma total dos custos com toda infraestrutura da GCP agrupados pelos serviços com maior volume de gastos em USD
        * Verificamos quais serviços tem maior valor gasto e agrupamos todos os demais na categorias outros. Detalhes podem ser vistos na consulta do Metabase
    * Métricas de Custos Mensais e Semestrais
Disclaimer
Todoas esses métricas são calculadas somente a partir de 2026;
Gasto Mensal com Principais Serviços da GCP (BRL)
        * O que a métrica mede?
          * Custo mensal com os principais serviços em BRL no semestre atual
        * Utiliza tabela gcp_billing_export_resource_v1_01620F_C10520_CCE6DC que contém os custos com a nova conta de faturamento em BRL e USD;
Custo total de serviços da GCP por Semestre
        * O que a métrica mede?
          * Soma do custo total de serviços em BRL por semestre a partir de 2026
        * Utiliza tabela gcp_billing_export_resource_v1_01620F_C10520_CCE6DC que contém os custos com a nova conta de faturamento em BRL e USD;
Custo por Serviço da GCP por Semestre
        * O que a métrica mede?
        * Utiliza tabela gcp_billing_export_resource_v1_01620F_C10520_CCE6DC que contém os custos com a nova conta de faturamento em BRL e USD;



* TODO: Refletir sobre como representar essas métricas no metabase e como criar processos de check;
  * Tabelas que contém nomes de colunas diferentes do Big Query;
  * Tabelas que não possuem metadados de última atualização na fonte original, última verificação e última atualização na BD
1. Acesso aos dados
* Quantidade de acessos mensais por conjunto
* Quantidade de acesso mensais por conjunto bd pro





Fluxo de trabalho: abertura de PRs e Commits

* material de referÊncia para importância da descrição dos commits
  * O que a material nos informa sobre princípios para boas mensagens de commits?

Métricas de Consulta aos dados



* Logs de auditoria do Big Query - Cloud Audit Logs
  * Os logs de auditoria registram eventos de interação com o ambiente do Big Query. Na BD, temos os logs configurados para o projeto
  * Os logs foram configurados para serem exportados a cada hora na tabela  cloudaudit_googleapis_com_data_access que fica no projeto basedosdados do BQ.
* Limitações
  * Quando o conjunto de dados tem política de permissionamento de linhas e/ou é um conjunto aberto os logs de auditoria não coletam informações de identificação dos usuários. Dados como email, IP, query executada não são registrados pelos Logs de Auditoria no projeto basedosdados, somente  projeto do usuário que realizou a consulta. 
  * Referência da documentação da GCP: 
    * Data Access Audit Logs
“Publicly available resources that have the Identity and Access Management policies allAuthenticatedUsers or allUsers don’t generate audit logs”
  * Documentação da GCP sobre como settar a exportação de logs do cloudaudit para Big Query



Como utilizar os logs para gerar métricas sobre consultas executadas na BD?

1. Fluxo de Filtragem e Processamento 

    Para gerar métricas confiáveis, os logs brutos devem passar por um "funil" de tratamento. O BigQuery registra eventos de infraestrutura, e não apenas consultas. Portanto, a filtragem precisa ocorrer em 3 etapas:

    Etapa 1: Filtragem de Evento (O que aconteceu?)

    * Variável: protopayload_auditlog.methodName
    * Filtro: 'jobservice.insert' OR 'jobservice.query'
    * Por que: Embora jobservice.insert registre a tentativa de envio, apenas jobcompleted garante que o processamento ocorreu e gerou estatísticas de uso.

    Etapa 2: Expansão de Estrutura (Achatamento)

    * Variável: protopayload_auditlog.authorizationInfo
    * Operação: UNNEST()
    * Por que: Este campo é um ARRAY. Uma única query pode conter múltiplos objetos de autorização (ex: verificar acesso à tabela E verificar acesso à Row Policy). O UNNEST transforma esse array em linhas individuais para análise.

    Etapa 3: Filtragem de Escopo (O que foi acessado?)

    * Variável: auth.permission (campo gerado pelo unnest)
    * Filtro: bigquery.tables.getData OU bigquery.rowAccessPolicies.getFilteredData
    * Por que: Descarta eventos administrativos (como listar tabelas ou ver metadados) e foca apenas na leitura de dados.
2. Quantidades de consultas feitas em tabelas sem permissionamento

  As tabelas públicas ou sem restrição de linha (Row Level Security - RLS) possuem um comportamento linear no log.

  * Identificação: O recurso (resource) termina diretamente no nome da tabela e o valor da variável auth.permission é bigquery.tables.getData
  * Comportamento do Log:
    * Gera 1 evento de autorização por tabela consultada.
    * A permissão registrada é bigquery.tables.getData
3. Métricas para Tabelas Com Permissionamento (BD Pro)

  Tabelas com Row Access Policies (como as do conjunto br_bd_pro) geram múltiplos eventos de verificação para uma única consulta. É aqui que reside a complexidade da diferenciação entre usuários BD Pro e Públicos.

* Identificação: O recurso (resource) possui sufixos como /rowAccessPolicies/bdpro_filter ou /rowAccessPolicies/allusers_filter.
* Comportamento do Log:
  * Gera 3 ou mais eventos por tabela (Verificação da Tabela Base + Verificação de cada Policy).
* A Lógica do "Granted":
  * true: Acesso concedido através desta política específica.
  * false: Acesso negado para esta política específica (mas pode ter sido concedido por outra).
  * NULL: A política foi avaliada, mas não foi utilizada para conceder o acesso (Usuário entrou por outra rota)



Tipo de Usuário	Permission	Recurso (Policy)	Granted Status	Interpretação
BD Pro	bigquery.rowAccessPolicies.getFilteredData	bdpro_filter	TRUE	Contabilizar como um acesso de usuário BD PRO
Público	bigquery.rowAccessPolicies.getFilteredData	allusers_filter	TRUE	Contabilizar como acesso público
Público	bigquery.tables.getData	Null	TRUE	Contabilizar como acesso público
				


Consulta de SQL  para gerar ambas as métricas

with t2 as (
select 
    EXTRACT(year FROM timestamp) AS ano,
    EXTRACT(month FROM timestamp) AS mes,
logName,
protopayload_auditlog.methodName as name,
auth.permission,
protopayload_auditlog.status.code,
protopayload_auditlog.status.message,
insertId,
auth.granted,
SPLIT(auth.resource, '/')[SAFE_OFFSET(3)] as id_conjunto,
SPLIT(auth.resource, '/')[SAFE_OFFSET(5)] as id_tabela,
SPLIT(auth.resource, '/')[SAFE_OFFSET(7)] as politica_permissionamento,
CASE 
        WHEN REGEXP_CONTAINS(lower(protopayload_auditlog.requestMetadata.callerSuppliedUserAgent), 'python') then 'python'
        WHEN REGEXP_CONTAINS(lower(protopayload_auditlog.requestMetadata.callerSuppliedUserAgent), 'rstudio|bigrquery' ) then 'R'
        ELSE 'bigquery'
    END AS tipo_conexao,
from `basedosdados.logs.cloudaudit_googleapis_com_data_access`,
UNNEST(protopayload_auditlog.authorizationInfo) AS auth
where (protopayload_auditlog.methodName = 'jobservice.insert' OR protopayload_auditlog.methodName = 'jobservice.query') and  EXTRACT(year FROM timestamp) = 2025 and  EXTRACT(month FROM timestamp) = 1 and
auth.permission IN (
    'bigquery.tables.getData', 
    'bigquery.rowAccessPolicies.getFilteredData'
  ))

select
ano,
mes,
id_conjunto,
COUNTIF(politica_permissionamento is not null and granted = true) as quantidade_acessos_bd_profilter,
COUNTIF(politica_permissionamento is null) as quantidade_acessos_allusers,
COUNTIF(politica_permissionamento is null and permission = 'bigquery.tables.getData') as quantidadade_total_acessos
--- note que ambos quantidade_acessos_allusers e quantidadade_total_acessos
from t2
group by all
order by 1,2
desc






Guia de preenchimento de metadados

