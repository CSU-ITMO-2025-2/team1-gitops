# Team1 GitOps Infrastructure

> Production-ready Kubernetes deployment with ArgoCD, Argo Rollouts, KEDA autoscaling, and HashiCorp Vault integration

## 📋 Оглавление

- [Архитектура](#архитектура)
- [Компоненты](#компоненты)
- [Управление секретами](#управление-секретами)
- [Argo Rollouts (Canary Deployment)](#argo-rollouts-canary-deployment)
- [Автомасштабирование (KEDA)](#автомасштабирование-keda)
- [Проблемы и решения](#проблемы-и-решения)
- [CI/CD Pipeline](#cicd-pipeline)
- [Как работать с проектом](#как-работать-с-проектом)

---

## 🏗️ Архитектура

```
GitHub (team1-gitops)

ArgoCD (GitOps controller)

Kubernetes Cluster (team1-ns)
    ├── core-api (Argo Rollout + canary)
    ├── frontend (Deployment + HPA)
    ├── workers × 3 (Deployment + KEDA)
    ├── RabbitMQ (Deployment)
    └── PostgreSQL (External)
```

### Принципы:

1. **GitOps**: Все изменения через Git → ArgoCD автоматически синхронизирует
2. **Declarative**: Инфраструктура как код (Infrastructure as Code)
3. **Progressive Delivery**: Безопасные обновления через Argo Rollouts
4. **Auto-scaling**: HPA для frontend, KEDA для workers
5. **Security**: Секреты через HashiCorp Vault + External Secrets Operator

---

## 📦 Компоненты

| Сервис | Тип | Автомасштабирование | Описание |
|--------|-----|---------------------|----------|
| `core-api` | **Rollout** | Manual (1 replica) | REST API, Argo Rollouts с canary |
| `frontend` | Deployment | HPA (1-5) | Streamlit UI, масштабирование по CPU |
| `job-descr-worker` | Deployment | KEDA (0-5) | Обработка описаний вакансий, очередь RabbitMQ |
| `question-worker` | Deployment | KEDA (0-5) | Генерация вопросов, очередь RabbitMQ |
| `resume-worker` | Deployment | KEDA (0-5) | Оценка резюме, очередь RabbitMQ |
| `rabbitmq` | Deployment | Static (1) | Message broker |
| `postgres` | External | - | База данных (79.174.89.43:18809) |

### Образы:

Все образы хранятся в **GitHub Container Registry** (GHCR):
```
ghcr.io/csu-itmo-2025-2/team1-core-api:sha-79af7ba
ghcr.io/csu-itmo-2025-2/team1-frontend:sha-79af7ba
ghcr.io/csu-itmo-2025-2/team1-*-worker:sha-79af7ba
```

Автоматическое обновление тегов через GitHub Actions при мерже в `main`.

---

## 🔐 Управление секретами

### Архитектура секретов:

```
HashiCorp Vault (vault.kubepractice.ru)
     (token: yRwIqKW40ggB)
SecretStore (vault-team1)
     (syncs every 1m)
ExternalSecret (создаёт K8s Secret)
    
Pod (монтирует как env переменные)
```

### Как это работает:

#### 1. **SecretStore**:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-team1
  namespace: team1-ns
spec:
  provider:
    vault:
      server: "https://vault.kubepractice.ru"
      path: "team1"
      version: "v2"
      auth:
        tokenSecretRef:
          name: vault-credentials
          key: token
```

**Назначение**: Подключение к Vault, хранит токен аутентификации.

#### 2. **ExternalSecret** :

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: team1-project-core-api-secrets
spec:
  refreshInterval: 1m  # Синхронизация каждую минуту
  secretStoreRef:
    name: vault-team1
  target:
    name: team1-project-core-api-secrets  # Имя создаваемого Secret
  data:
    - secretKey: OPENAI_API_KEY
      remoteRef:
        key: team1/core-api
        property: OPENAI_API_KEY
```

**Что происходит**:
1. External Secrets Operator читает этот манифест
2. Подключается к Vault через `vault-team1` SecretStore
3. Забирает значение из `team1/core-api` → `OPENAI_API_KEY`
4. Создаёт Kubernetes Secret `team1-project-core-api-secrets`
5. Обновляет каждую минуту (если в Vault изменилось)

#### 3. **Pod монтирует Secret**:

```yaml
containers:
  - name: core-api
    envFrom:
      - secretRef:
          name: team1-project-core-api-secrets
```

### Какие секреты у нас есть:

| ExternalSecret | Vault Path | Ключи | Используется |
|----------------|------------|-------|--------------|
| `postgres-secrets` | `team1/postgres` | `POSTGRES_PASSWORD` | PostgreSQL |
| `rabbitmq-secrets` | `team1/rabbitmq` | `RABBITMQ_DEFAULT_USER`, `RABBITMQ_DEFAULT_PASS` | RabbitMQ + KEDA |
| `core-api-secrets` | `team1/core-api` | `OPENAI_API_KEY`, `POSTGRES_PASSWORD`, `RABBITMQ_DEFAULT_PASS` | core-api |
| `frontend-secrets` | `team1/frontend` | `OPENAI_API_KEY`, `KC_CLIENT_SECRET` | frontend |
| `workers-secrets` | `team1/workers` | `OPENAI_API_KEY`, `POSTGRES_PASSWORD`, `RABBITMQ_DEFAULT_PASS` | workers |



### Проверка статуса:

```bash
# Проверить SecretStore
kubectl get secretstore vault-team1 -n team1-ns

# Проверить ExternalSecrets
kubectl get externalsecrets -n team1-ns

# Проверить созданные Secrets
kubectl get secrets -n team1-ns | grep team1-project
```

---

## 🚀 Argo Rollouts (Canary Deployment)

### Сравнение

**Обычный Deployment**: При обновлении образа сразу заменяет все поды → если баг, все пользователи видят ошибку.

**Argo Rollout**: Постепенно переводит трафик на новую версию, можно откатиться на любом шаге.

### Наша стратегия (canary):

```yaml
strategy:
  canary:
    steps:
      - setWeight: 20        # Шаг 1: 20% трафика на новую версию
      - pause: {duration: 2m} # Пауза 2 минуты (смотрим метрики)
      - setWeight: 50        # Шаг 2: 50% трафика
      - pause: {duration: 2m} # Пауза 2 минуты
      - setWeight: 100       # Шаг 3: 100% на новую версию
```

### Как это работает:

**Пример**: Обновление `core-api` с версии `sha-79af7ba` на `sha-abc1234`

1. **0 мин**: Создаётся 1 новый под (новая версия)
   - Старая версия: 80% трафика
   - Новая версия: 20% трафика (канарейка)

2. **2 мин**: Проверяем метрики (error rate, latency)
   - ✅ Всё ОК? Продолжаем
   - ❌ Ошибки? Откатываемся: `kubectl argo rollouts abort team1-project-core-api -n team1-ns`

3. **4 мин**: Трафик 50/50

4. **6 мин**: 100% на новой версии, старый под удаляется

### Команды для проверко:

```bash
# Статус Rollout
kubectl argo rollouts get rollout team1-project-core-api -n team1-ns

# Промотировать на следующий шаг (пропустить паузу)
kubectl argo rollouts promote team1-project-core-api -n team1-ns

# Откатиться к предыдущей версии
kubectl argo rollouts abort team1-project-core-api -n team1-ns
kubectl argo rollouts undo team1-project-core-api -n team1-ns

# История rollout
kubectl argo rollouts history team1-project-core-api -n team1-ns
```

## 🔄 CI/CD Pipeline

### GitHub Actions Workflow

**Файл**: `.github/workflows/ci.yml` (репозиторий `team1`)

**Триггер**: Push в ветку `main`

**Шаги**:

1. **Test**: Запуск тестов `core_api` (pytest)
2. **Build**: Сборка Docker образов для всех сервисов
3. **Push**: Загрузка образов в GHCR с тегом `sha-<commit>`
4. **Update GitOps**: Автоматическое создание PR в `team1-gitops` с обновлением тегов

### Автообновление образов

После мержа PR в `team1` → автоматически создаётся PR в `team1-gitops`:

```yaml
# production/values.yaml
image:
  coreApi:
    repository: ghcr.io/csu-itmo-2025-2/team1-core-api
    tag: sha-79af7ba  # ← Обновляется автоматически
```

ArgoCD видит изменения в Git → синхронизирует кластер.
