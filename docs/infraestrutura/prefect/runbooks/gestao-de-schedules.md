---
tipo: runbook
titulo: Runbook — Schedule de flow não desativa ou não sincroniza
trigger: Flows não sendo desativados automaticamente após falhas consecutivas, ou automação e sync retornando erros (401, 500)
---
# Runbook — Schedule de flow não desativa ou não sincroniza

## Sintoma

Um destes, observado no Prefect 3, no Django Admin ou nos logs do backend no Grafana:

- um flow falhou duas vezes seguidas (ou falhou no `run_dbt`) e continua com o schedule ativo;
- o webhook `flow-failed` ou o sync do CI retorna `401`;
- um flow aparece ativo no Django Admin e pausado no Prefect 3, ou o contrário;
- o log `Sync complete` não aparece depois de um deploy.

Como o sistema funciona (condições de desativação, endpoints, automação, campos do Django Admin):
[referência de gestão de schedules](../referencia/gestao-de-schedules.md).

## Impacto

Um flow quebrado que não é desativado continua rodando e falhando a cada execução agendada, gastando
pods e cota. Um flow que fica pausado por divergência deixa a tabela desatualizada sem alerta.

## Diagnóstico

1. No Grafana, procure `webhook received` no horário da falha (consulta "Webhook recebido e
   ações" da referência). Sem registro, a automação não disparou ou o webhook falhou.
2. Procure `Unauthorized` com `flow-failed` (consulta "Erros de autenticação"). Se aparecer, a
   chave do bloco está errada.
3. Depois de um deploy, procure `Sync complete`. Sem registro, veja o log da action
   `cd-prefect3.yaml` no GitHub: o passo de sync falha em silêncio (`|| echo`).
4. Para achar divergências entre o banco e o Prefect 3, rode no shell do pod do backend:

    ```python
    from backend.apps.admin_data_tools.models import DisabledFlowSchedule
    from backend.apps.admin_data_tools._prefect3_client import Prefect3Client

    client = Prefect3Client()
    for dep in client.iter_deployments():
        try:
            record = DisabledFlowSchedule.objects.get(deployment_id=dep['id'])
            prefect_paused = dep.get('paused', False)
            db_active = record.is_schedule_active
            if prefect_paused == db_active:  # estado invertido = divergência
                print(f'DIVERGÊNCIA: {record.flow_name} | DB active={db_active} | Prefect paused={prefect_paused}')
        except DisabledFlowSchedule.DoesNotExist:
            print(f'SEM REGISTRO: {dep["name"]}')
    ```

## Resolução

### Caso A: o webhook retorna 401

1. Confira a `PREFECT3_API_KEY` no sealed secret do backend.
2. Confira o header de autenticação do bloco `flow-failed-prod` no Prefect 3. As duas chaves precisam
   ser iguais.

### Caso B: o flow falhou, mas não foi desativado

1. Se o diagnóstico mostrou que a automação não disparou ou o webhook deu erro, corrija a causa
   (Caso A, se for autenticação).
2. Se a automação ficou fora do ar por um período, force a desativação dos flows com falhas
   acumuladas, no shell do pod do backend:

    ```python
    from backend.apps.admin_data_tools.models import DisabledFlowSchedule
    from backend.apps.admin_data_tools.views import _is_consecutive_failure, _is_dbt_failure
    from backend.apps.admin_data_tools._prefect3_client import Prefect3Client
    from datetime import datetime, timezone

    client = Prefect3Client()
    for record in DisabledFlowSchedule.objects.filter(is_schedule_active=True):
        runs = client.get_recent_completed_runs(record.deployment_id, limit=2)
        should_disable = _is_consecutive_failure(runs, record.reactivated_at)
        if not should_disable and runs:
            task_runs = client.get_failed_task_runs(runs[0]['id'])
            should_disable = _is_dbt_failure(task_runs, runs[0]['start_time'], record.reactivated_at)
        if should_disable:
            client.set_paused(record.deployment_id, paused=True)
            record.is_schedule_active = False
            record.reactivated_at = None
            record.disabled_at = datetime.now(tz=timezone.utc)
            record.save(update_fields=['is_schedule_active', 'reactivated_at', 'disabled_at'])
            print(f'Desativado: {record.flow_name}')
    ```

### Caso C: o sync do CI retorna 401

1. O secret `PREFECT3_AUTH_TOKEN` do repositório `pipelines` está diferente da `PREFECT3_API_KEY`
   do pod.
2. Sele de novo o secret no IAC com a chave correta.

### Caso D: o banco e o Prefect 3 divergem

1. Rode o sync à mão:

    ```bash
    curl -X POST https://backend.basedosdados.org/admin-tools/sync-deployments/ \
      -H "Authorization: Bearer $PREFECT3_AUTH_TOKEN"
    ```

2. O sync aplica o estado do banco no Prefect 3. Se o banco estiver errado, corrija o flow no
   Django Admin antes (marque ou desmarque `is_schedule_active` e, ao reativar, preencha
   `reactivated_at`).

### Caso E: flows que devem ficar pausados voltaram a rodar depois de um deploy

1. Rode o script que mantém a lista `SHOULD_BE_PAUSED`:

    ```bash
    PREFECT_API_KEY=<chave-compartilhada> python pipelines/tmp/prefect_pause_flows.py
    ```

2. Sincronize o banco para refletir as pausas aplicadas à mão (Caso D).

## Após a resolução

- Confira no Grafana se o próximo `webhook received` ou `Sync complete` aparece sem erro.
- Se a causa foi uma chave trocada, avise quem gerencia os secrets do IAC.
- Se a divergência se repetir, abra uma issue no repositório do backend.

## Ver também

- [Referência — Gestão de Schedules](../referencia/gestao-de-schedules.md) — como o sistema
  funciona, endpoints e campos.
- [Como configurar a automação Flow Failed Webhook](../como-fazer/flow-failed-webhook.md)
