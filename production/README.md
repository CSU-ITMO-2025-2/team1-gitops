# Production Environment

Helm chart для production окружения team1 HR Assistant.

## Компоненты

- **core-api** (Rollout) - основное API с Canary deployment
- **frontend** - web интерфейс
- **workers** - обработчики задач (resume, job-descr, cv-recommendations)
- **postgres** - база данных
- **rabbitmq** - очередь сообщений

## Secrets

Используются существующие Kubernetes Secret:
- `hr-assist-secrets` - OPENAI_API_KEY, KC_CLIENT_SECRET, RABBITMQ_DEFAULT_PASS, POSTGRES_PASSWORD

## Изменение конфигурации

Отредактируйте `values.yaml`:
- `image.tag` - версия Docker images
- `replicas` - количество реплик
- `resources` - CPU/Memory лимиты

После изменений закоммитьте в `main` - ArgoCD автоматически применит.
