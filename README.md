# Team1 GitOps Repository

Этот репозиторий содержит Kubernetes манифесты для team1 HR Assistant приложения.

## Структура

```
team1-gitops/
├── production/          # Production окружение
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
└── README.md
```

## ArgoCD Configuration

**Repository URL:** `https://github.com/CSU-ITMO-2025-2/team1-gitops`
**Path:** `production`
**Target Revision:** `main`
**Namespace:** `team1-ns`

## Deployment Strategy

Используется **Argo Rollouts** с Canary deployment стратегией для core-api:
- 20% трафика → пауза 2 минуты
- 50% трафика → пауза 2 минуты
- 100% трафика (полный деплой)

## Как обновить версию

1. Изменить `image.tag` в `production/values.yaml`
2. Закоммитить и запушить в `main`
3. ArgoCD автоматически начнёт Canary rollout

## Мониторинг деплоя

```bash
# Смотреть статус Rollout
kubectl argo rollouts get rollout team1-core-api -n team1-ns

# Ускорить деплой (пропустить паузы)
kubectl argo rollouts promote team1-core-api -n team1-ns

# Откатить деплой
kubectl argo rollouts abort team1-core-api -n team1-ns
```
