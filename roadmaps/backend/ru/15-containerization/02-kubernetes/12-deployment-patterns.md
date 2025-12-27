# Паттерны деплоя в Kubernetes

## Введение в стратегии деплоя

Стратегия деплоя определяет, как приложение обновляется в production-окружении. Правильный выбор стратегии критически важен для обеспечения:

- **Zero-downtime** — пользователи не замечают обновления
- **Быстрого rollback** — возможность откатиться при проблемах
- **Контролируемого риска** — минимизация влияния багов на пользователей
- **Тестирования в production** — проверка на реальном трафике

### Обзор основных паттернов

| Паттерн | Downtime | Rollback | Сложность | Ресурсы |
|---------|----------|----------|-----------|---------|
| Recreate | Да | Медленный | Низкая | 1x |
| Rolling Update | Нет | Быстрый | Низкая | 1x-2x |
| Blue-Green | Нет | Мгновенный | Средняя | 2x |
| Canary | Нет | Быстрый | Высокая | 1x+ |
| A/B Testing | Нет | Быстрый | Высокая | 1x+ |
| Shadow | Нет | Мгновенный | Очень высокая | 2x |

---

## Rolling Update — постепенное обновление

Rolling Update — встроенная стратегия Kubernetes, которая постепенно заменяет старые Pod'ы новыми. Это стратегия по умолчанию для Deployment.

### Принцип работы

```
Начальное состояние:  [v1] [v1] [v1] [v1]
Шаг 1:                [v1] [v1] [v1] [v2]
Шаг 2:                [v1] [v1] [v2] [v2]
Шаг 3:                [v1] [v2] [v2] [v2]
Шаг 4:                [v2] [v2] [v2] [v2]
```

### Конфигурация Rolling Update

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  labels:
    app: web-app
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      # Максимальное количество Pod'ов сверх replicas
      maxSurge: 1          # или "25%"
      # Максимальное количество недоступных Pod'ов
      maxUnavailable: 0    # или "25%"
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web-app
        image: myapp:v2
        ports:
        - containerPort: 8080
        # Критически важны для Rolling Update
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 10
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
```

### Параметры maxSurge и maxUnavailable

```yaml
# Вариант 1: Консервативный (zero-downtime гарантирован)
strategy:
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
# Результат: 4 -> 5 -> 5 -> 5 -> 4 Pod'ов
# Медленнее, но всегда 100% capacity

# Вариант 2: Агрессивный (быстрее, но может быть degradation)
strategy:
  rollingUpdate:
    maxSurge: 2
    maxUnavailable: 1
# Результат: Быстрое обновление, но временно 75% capacity

# Вариант 3: Процентные значения
strategy:
  rollingUpdate:
    maxSurge: 25%
    maxUnavailable: 25%
# Масштабируется с количеством реплик
```

### Контроль скорости обновления

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: slow-rollout-app
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  # Минимальное время, которое Pod должен быть Ready
  # перед тем, как считаться доступным
  minReadySeconds: 30
  # Время ожидания прогресса деплоя
  progressDeadlineSeconds: 600
  template:
    spec:
      containers:
      - name: app
        image: myapp:v2
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          successThreshold: 3  # 3 успешных проверки подряд
```

### Мониторинг Rolling Update

```bash
# Наблюдение за процессом обновления
kubectl rollout status deployment/web-app

# Просмотр истории
kubectl rollout history deployment/web-app

# Детали конкретной ревизии
kubectl rollout history deployment/web-app --revision=2

# Пауза и возобновление rollout
kubectl rollout pause deployment/web-app
kubectl rollout resume deployment/web-app

# Откат к предыдущей версии
kubectl rollout undo deployment/web-app

# Откат к конкретной ревизии
kubectl rollout undo deployment/web-app --to-revision=3
```

---

## Recreate — полное пересоздание

Стратегия Recreate полностью останавливает все Pod'ы старой версии перед созданием новых. Это приводит к downtime, но гарантирует, что только одна версия работает в любой момент времени.

### Когда использовать Recreate

- База данных или stateful-приложения, которые не поддерживают несколько версий одновременно
- Миграции схемы данных, несовместимые с предыдущей версией
- Приложения с лицензией на одну инстанцию
- Dev/staging окружения, где downtime допустим

### Конфигурация Recreate

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: legacy-database-app
spec:
  replicas: 3
  strategy:
    type: Recreate  # Просто указываем тип
  selector:
    matchLabels:
      app: legacy-database-app
  template:
    metadata:
      labels:
        app: legacy-database-app
    spec:
      containers:
      - name: app
        image: legacy-app:v2
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: data
          mountPath: /var/lib/data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: app-data-pvc
```

### Процесс Recreate

```
Шаг 1: Terminate all old Pods
[v1] [v1] [v1] -> [   ] [   ] [   ]
                  (downtime начинается)

Шаг 2: Create all new Pods
[   ] [   ] [   ] -> [v2] [v2] [v2]
                     (downtime заканчивается)
```

### Recreate с maintenance page

```yaml
# Job для показа maintenance page во время обновления
apiVersion: batch/v1
kind: Job
metadata:
  name: maintenance-mode
spec:
  template:
    spec:
      containers:
      - name: maintenance
        image: nginx:alpine
        command: ["/bin/sh", "-c"]
        args:
        - |
          echo "Updating... Please wait" > /usr/share/nginx/html/index.html
          nginx -g 'daemon off;'
      restartPolicy: Never
---
# Ingress переключается на maintenance pod во время обновления
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    # Можно использовать default backend для maintenance
    nginx.ingress.kubernetes.io/default-backend: maintenance-svc
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
```

---

## Blue-Green Deployment

Blue-Green deployment поддерживает два идентичных production-окружения. Одно (Blue) обслуживает трафик, второе (Green) используется для деплоя новой версии. После проверки трафик переключается.

### Архитектура Blue-Green

```
                    ┌─────────────┐
                    │   Ingress/  │
                    │   Service   │
                    └──────┬──────┘
                           │
              ┌────────────┴────────────┐
              │                         │
              ▼                         ▼
     ┌─────────────────┐       ┌─────────────────┐
     │  Blue (active)  │       │ Green (standby) │
     │    version 1    │       │    version 2    │
     │  [v1] [v1] [v1] │       │  [v2] [v2] [v2] │
     └─────────────────┘       └─────────────────┘
```

### Реализация Blue-Green с Service selector

```yaml
# Blue Deployment (текущая production версия)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-blue
  labels:
    app: myapp
    version: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
      - name: app
        image: myapp:1.0.0
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
---
# Green Deployment (новая версия)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-green
  labels:
    app: myapp
    version: green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
      - name: app
        image: myapp:2.0.0
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
---
# Service, направляющий трафик на активную версию
apiVersion: v1
kind: Service
metadata:
  name: app-service
spec:
  selector:
    app: myapp
    version: blue  # Переключаем на green для switch
  ports:
  - port: 80
    targetPort: 8080
---
# Отдельные сервисы для тестирования каждой версии
apiVersion: v1
kind: Service
metadata:
  name: app-blue-test
spec:
  selector:
    app: myapp
    version: blue
  ports:
  - port: 80
    targetPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: app-green-test
spec:
  selector:
    app: myapp
    version: green
  ports:
  - port: 80
    targetPort: 8080
```

### Скрипт переключения Blue-Green

```bash
#!/bin/bash
# blue-green-switch.sh

CURRENT_VERSION=$(kubectl get svc app-service -o jsonpath='{.spec.selector.version}')

if [ "$CURRENT_VERSION" == "blue" ]; then
    NEW_VERSION="green"
else
    NEW_VERSION="blue"
fi

echo "Switching from $CURRENT_VERSION to $NEW_VERSION"

# Проверка готовности новой версии
echo "Checking $NEW_VERSION deployment readiness..."
kubectl rollout status deployment/app-$NEW_VERSION

# Переключение трафика
kubectl patch svc app-service -p "{\"spec\":{\"selector\":{\"version\":\"$NEW_VERSION\"}}}"

echo "Traffic switched to $NEW_VERSION"
echo "Testing new version..."

# Простой smoke test
for i in {1..5}; do
    STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://app.example.com/health)
    if [ "$STATUS" != "200" ]; then
        echo "Health check failed! Rolling back..."
        kubectl patch svc app-service -p "{\"spec\":{\"selector\":{\"version\":\"$CURRENT_VERSION\"}}}"
        exit 1
    fi
    sleep 2
done

echo "Deployment successful!"
```

### Blue-Green с Ingress (Nginx)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/canary: "false"
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service  # Основной сервис
            port:
              number: 80
---
# Ingress для тестирования green версии
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-green-ingress
spec:
  rules:
  - host: green.app.example.com  # Тестовый домен
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-green-test
            port:
              number: 80
```

### Blue-Green с Argo Rollouts

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: app-rollout
spec:
  replicas: 3
  revisionHistoryLimit: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: myapp:1.0.0
        ports:
        - containerPort: 8080
  strategy:
    blueGreen:
      # Сервис для активной версии
      activeService: app-active
      # Сервис для preview (новой) версии
      previewService: app-preview
      # Автоматическое переключение после проверки
      autoPromotionEnabled: false
      # Время на ручную проверку
      autoPromotionSeconds: 30
      # Масштабировать preview до переключения
      previewReplicaCount: 3
      # Откатить при отмене
      abortScaleDownDelaySeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  name: app-active
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: app-preview
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
```

---

## Canary Deployment — канареечные релизы

Canary deployment направляет небольшой процент трафика на новую версию, постепенно увеличивая его при успешных проверках. Названо в честь канареек, которых использовали шахтёры для обнаружения опасных газов.

### Принцип работы Canary

```
Этап 1 (5% трафика):
Production: [v1] [v1] [v1] [v1] [v1] [v1] [v1] [v1] [v1] [v2]
                                                         ↑ canary

Этап 2 (25% трафика):
Production: [v1] [v1] [v1] [v1] [v1] [v1] [v2] [v2] [v2] [v2]

Этап 3 (100% трафика):
Production: [v2] [v2] [v2] [v2] [v2] [v2] [v2] [v2] [v2] [v2]
```

### Базовый Canary с разным количеством реплик

```yaml
# Stable версия (90% трафика при равном весе)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: myapp
      version: stable
  template:
    metadata:
      labels:
        app: myapp
        version: stable
    spec:
      containers:
      - name: app
        image: myapp:1.0.0
        ports:
        - containerPort: 8080
---
# Canary версия (10% трафика)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
      version: canary
  template:
    metadata:
      labels:
        app: myapp
        version: canary
    spec:
      containers:
      - name: app
        image: myapp:2.0.0
        ports:
        - containerPort: 8080
---
# Service выбирает оба deployment'а
apiVersion: v1
kind: Service
metadata:
  name: app-service
spec:
  selector:
    app: myapp  # Без version - выбирает оба
  ports:
  - port: 80
    targetPort: 8080
```

### Canary с Nginx Ingress (weighted routing)

```yaml
# Основной Ingress (stable)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress-stable
spec:
  ingressClassName: nginx
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-stable-service
            port:
              number: 80
---
# Canary Ingress с аннотациями
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress-canary
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    # Процент трафика на canary
    nginx.ingress.kubernetes.io/canary-weight: "10"
spec:
  ingressClassName: nginx
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-canary-service
            port:
              number: 80
```

### Canary с маршрутизацией по заголовкам

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress-canary-header
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    # Направлять на canary если есть header X-Canary: always
    nginx.ingress.kubernetes.io/canary-by-header: "X-Canary"
    nginx.ingress.kubernetes.io/canary-by-header-value: "always"
spec:
  ingressClassName: nginx
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-canary-service
            port:
              number: 80
```

### Canary с маршрутизацией по cookie

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress-canary-cookie
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    # Направлять на canary если cookie canary=true
    nginx.ingress.kubernetes.io/canary-by-cookie: "canary"
spec:
  ingressClassName: nginx
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-canary-service
            port:
              number: 80
```

### Canary с Argo Rollouts

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: app-rollout
spec:
  replicas: 10
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: myapp:1.0.0
        ports:
        - containerPort: 8080
  strategy:
    canary:
      # Шаги canary развёртывания
      steps:
      # Шаг 1: 5% трафика
      - setWeight: 5
      # Пауза для ручной проверки
      - pause: {duration: 1m}

      # Шаг 2: 20% трафика
      - setWeight: 20
      - pause: {duration: 2m}

      # Шаг 3: 50% трафика
      - setWeight: 50
      - pause: {duration: 5m}

      # Шаг 4: 80% трафика
      - setWeight: 80
      - pause: {duration: 5m}

      # Финальный шаг - полный rollout
      # (автоматически 100%)

      # Сервисы для traffic splitting
      canaryService: app-canary
      stableService: app-stable

      # Traffic routing через Nginx, Istio, или другой
      trafficRouting:
        nginx:
          stableIngress: app-ingress
          additionalIngressAnnotations:
            canary-by-header: X-Canary
            canary-by-header-value: always
```

### Canary с автоматическим анализом метрик

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: app-rollout-with-analysis
spec:
  replicas: 10
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: myapp:1.0.0
  strategy:
    canary:
      steps:
      - setWeight: 20
      - pause: {duration: 1m}

      # Запуск анализа метрик
      - analysis:
          templates:
          - templateName: success-rate
          args:
          - name: service-name
            value: app-canary

      - setWeight: 50
      - pause: {duration: 2m}

      - analysis:
          templates:
          - templateName: success-rate
          - templateName: latency-check
          args:
          - name: service-name
            value: app-canary

      - setWeight: 100
---
# AnalysisTemplate для проверки success rate
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  args:
  - name: service-name
  metrics:
  - name: success-rate
    # Интервал проверки
    interval: 30s
    # Количество успешных проверок для прохождения
    successCondition: result[0] >= 0.95
    # Количество неуспешных для провала
    failureLimit: 3
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          sum(rate(http_requests_total{service="{{args.service-name}}",status=~"2.."}[5m]))
          /
          sum(rate(http_requests_total{service="{{args.service-name}}"}[5m]))
---
# AnalysisTemplate для проверки latency
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: latency-check
spec:
  args:
  - name: service-name
  metrics:
  - name: latency-p99
    interval: 30s
    # P99 latency должна быть меньше 500ms
    successCondition: result[0] < 500
    failureLimit: 3
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket{service="{{args.service-name}}"}[5m])) by (le)
          ) * 1000
```

---

## A/B Testing в Kubernetes

A/B Testing направляет пользователей на разные версии приложения на основе определённых критериев (заголовки, cookies, геолокация, user ID) для сравнения метрик.

### Отличие от Canary

| Аспект | Canary | A/B Testing |
|--------|--------|-------------|
| Цель | Проверка стабильности | Сравнение бизнес-метрик |
| Распределение | По весу (%) | По правилам (user segment) |
| Метрики | Ошибки, latency | Конверсия, engagement |
| Длительность | Минуты-часы | Дни-недели |

### A/B Testing с Istio

```yaml
# VirtualService для A/B роутинга
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: app-ab-test
spec:
  hosts:
  - app.example.com
  http:
  # Правило 1: Premium пользователи получают версию B
  - match:
    - headers:
        x-user-tier:
          exact: "premium"
    route:
    - destination:
        host: app-service
        subset: version-b

  # Правило 2: Пользователи из определённых регионов
  - match:
    - headers:
        x-user-region:
          regex: "^(US|CA|UK)$"
    route:
    - destination:
        host: app-service
        subset: version-b

  # Правило 3: По user ID (чётные -> A, нечётные -> B)
  - match:
    - headers:
        x-user-id:
          regex: ".*[13579]$"  # Нечётные ID
    route:
    - destination:
        host: app-service
        subset: version-b

  # Default: версия A
  - route:
    - destination:
        host: app-service
        subset: version-a
---
# DestinationRule определяет subsets
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: app-destination
spec:
  host: app-service
  subsets:
  - name: version-a
    labels:
      version: a
  - name: version-b
    labels:
      version: b
```

### A/B Testing с Nginx Ingress

```yaml
# Lua snippet для сложной логики маршрутизации
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-ab-test-config
data:
  ab-test.lua: |
    local user_id = ngx.var.http_x_user_id or ""
    local hash = ngx.crc32_long(user_id)
    local bucket = hash % 100

    -- 30% пользователей на версию B
    if bucket < 30 then
      ngx.var.proxy_upstream_name = "app-version-b"
    else
      ngx.var.proxy_upstream_name = "app-version-a"
    end
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/configuration-snippet: |
      set $proxy_upstream_name "";
      access_by_lua_file /etc/nginx/lua/ab-test.lua;
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
```

### A/B Testing с Argo Rollouts и Experiments

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Experiment
metadata:
  name: homepage-experiment
spec:
  # Длительность эксперимента
  duration: 7d

  # Определение вариантов
  templates:
  - name: control
    replicas: 3
    selector:
      matchLabels:
        app: homepage
        variant: control
    template:
      metadata:
        labels:
          app: homepage
          variant: control
      spec:
        containers:
        - name: app
          image: homepage:v1-control

  - name: experiment
    replicas: 3
    selector:
      matchLabels:
        app: homepage
        variant: experiment
    template:
      metadata:
        labels:
          app: homepage
          variant: experiment
      spec:
        containers:
        - name: app
          image: homepage:v1-experiment

  # Анализ для сравнения вариантов
  analyses:
  - name: conversion-rate-comparison
    templateName: ab-test-analysis
    args:
    - name: control-service
      value: homepage-control
    - name: experiment-service
      value: homepage-experiment
---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: ab-test-analysis
spec:
  args:
  - name: control-service
  - name: experiment-service
  metrics:
  - name: conversion-rate-diff
    interval: 1h
    # Experiment должен быть не хуже control на 5%
    successCondition: result[0] >= -0.05
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          (
            sum(rate(purchases_total{service="{{args.experiment-service}}"}[1h]))
            /
            sum(rate(page_views_total{service="{{args.experiment-service}}"}[1h]))
          )
          -
          (
            sum(rate(purchases_total{service="{{args.control-service}}"}[1h]))
            /
            sum(rate(page_views_total{service="{{args.control-service}}"}[1h]))
          )
```

---

## Shadow/Dark Launching

Shadow deployment (также известный как Dark Launching или Traffic Mirroring) копирует production трафик на новую версию без влияния на пользователей. Ответы новой версии отбрасываются.

### Применение Shadow Deployment

- Тестирование производительности под реальной нагрузкой
- Проверка совместимости с production данными
- Обнаружение багов без риска для пользователей
- Warming up кэшей перед переключением

### Shadow с Istio

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: app-shadow
spec:
  hosts:
  - app.example.com
  http:
  - route:
    # Основной трафик на stable версию
    - destination:
        host: app-stable
        port:
          number: 80
    # Зеркалирование на shadow версию
    mirror:
      host: app-shadow
      port:
        number: 80
    # Процент трафика для зеркалирования
    mirrorPercentage:
      value: 100.0
---
# Deployment для shadow версии
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-shadow
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: shadow
  template:
    metadata:
      labels:
        app: myapp
        version: shadow
    spec:
      containers:
      - name: app
        image: myapp:2.0.0-shadow
        ports:
        - containerPort: 8080
        env:
        # Маркер для логирования
        - name: SHADOW_MODE
          value: "true"
        # Отключение side effects
        - name: DISABLE_WRITES
          value: "true"
---
apiVersion: v1
kind: Service
metadata:
  name: app-shadow
spec:
  selector:
    app: myapp
    version: shadow
  ports:
  - port: 80
    targetPort: 8080
```

### Shadow с Nginx (request mirroring)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-mirror-config
data:
  nginx.conf: |
    upstream production {
        server app-stable:80;
    }

    upstream shadow {
        server app-shadow:80;
    }

    server {
        listen 80;

        location / {
            # Mirror запросы на shadow
            mirror /mirror;
            mirror_request_body on;

            # Основной запрос
            proxy_pass http://production;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }

        location = /mirror {
            internal;

            # Shadow запрос (ответ игнорируется)
            proxy_pass http://shadow$request_uri;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Shadow-Request "true";

            # Не ждать ответа
            proxy_connect_timeout 1s;
            proxy_read_timeout 1s;
        }
    }
```

### Сравнение ответов Shadow и Production

```yaml
# Deployment для сравнения ответов
apiVersion: apps/v1
kind: Deployment
metadata:
  name: response-comparator
spec:
  replicas: 1
  selector:
    matchLabels:
      app: response-comparator
  template:
    metadata:
      labels:
        app: response-comparator
    spec:
      containers:
      - name: comparator
        image: custom-comparator:latest
        env:
        - name: PRODUCTION_URL
          value: "http://app-stable"
        - name: SHADOW_URL
          value: "http://app-shadow"
        - name: COMPARISON_SAMPLE_RATE
          value: "0.1"  # Сравнивать 10% запросов
        ports:
        - containerPort: 8080
---
# ConfigMap с логикой сравнения
apiVersion: v1
kind: ConfigMap
metadata:
  name: comparator-rules
data:
  rules.yaml: |
    comparison_rules:
      # Игнорировать timestamps при сравнении
      ignore_fields:
        - "timestamp"
        - "request_id"
        - "generated_at"

      # Допустимые отклонения для числовых полей
      numeric_tolerance:
        - field: "price"
          tolerance: 0.01
        - field: "count"
          tolerance: 0

      # Алерты при различиях
      alerts:
        - type: slack
          channel: "#shadow-testing"
          threshold: 0.01  # 1% различий
```

---

## Feature Flags интеграция

Feature Flags (Feature Toggles) позволяют включать/выключать функциональность без деплоя нового кода. В комбинации с Kubernetes это мощный инструмент для постепенного релиза.

### Архитектура с Feature Flags

```
┌─────────────┐     ┌──────────────────┐     ┌─────────────┐
│   Client    │────▶│   Application    │◀───▶│ Feature Flag│
│             │     │                  │     │   Service   │
└─────────────┘     └──────────────────┘     └─────────────┘
                            │                       │
                            │                       │
                    ┌───────▼───────┐       ┌───────▼───────┐
                    │   Feature A   │       │   ConfigMap/  │
                    │   (enabled)   │       │   External    │
                    └───────────────┘       └───────────────┘
```

### Feature Flags через ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: feature-flags
data:
  features.json: |
    {
      "new_checkout_flow": {
        "enabled": true,
        "percentage": 25,
        "user_whitelist": ["user123", "user456"],
        "regions": ["US", "CA"]
      },
      "dark_mode": {
        "enabled": true,
        "percentage": 100
      },
      "experimental_search": {
        "enabled": false,
        "percentage": 0
      },
      "premium_features": {
        "enabled": true,
        "tiers": ["premium", "enterprise"]
      }
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
      annotations:
        # Автоматический reload при изменении ConfigMap
        checksum/config: "{{ sha256sum .Values.featureFlags }}"
    spec:
      containers:
      - name: app
        image: myapp:latest
        volumeMounts:
        - name: feature-flags
          mountPath: /etc/config
          readOnly: true
        env:
        - name: FEATURE_FLAGS_PATH
          value: /etc/config/features.json
      volumes:
      - name: feature-flags
        configMap:
          name: feature-flags
```

### Feature Flags с Hot Reload (Reloader)

```yaml
# Установка Reloader для автоматического перезапуска
# при изменении ConfigMap
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
  annotations:
    # Reloader будет следить за этим ConfigMap
    configmap.reloader.stakater.com/reload: "feature-flags"
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest
        volumeMounts:
        - name: feature-flags
          mountPath: /etc/config
      volumes:
      - name: feature-flags
        configMap:
          name: feature-flags
```

### Интеграция с LaunchDarkly

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest
        env:
        # LaunchDarkly SDK key
        - name: LAUNCHDARKLY_SDK_KEY
          valueFrom:
            secretKeyRef:
              name: launchdarkly-secret
              key: sdk-key
        - name: LAUNCHDARKLY_BASE_URI
          value: "https://sdk.launchdarkly.com"
---
apiVersion: v1
kind: Secret
metadata:
  name: launchdarkly-secret
type: Opaque
stringData:
  sdk-key: "sdk-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```

### Пример кода с Feature Flags (Python)

```python
# feature_flags.py
import json
import os
from functools import wraps
from flask import request, g
import hashlib

class FeatureFlags:
    def __init__(self, config_path: str):
        self.config_path = config_path
        self._load_config()

    def _load_config(self):
        with open(self.config_path) as f:
            self.flags = json.load(f)

    def is_enabled(self, feature: str, user_id: str = None,
                   region: str = None, tier: str = None) -> bool:
        if feature not in self.flags:
            return False

        flag = self.flags[feature]

        if not flag.get('enabled', False):
            return False

        # Проверка whitelist
        if user_id and user_id in flag.get('user_whitelist', []):
            return True

        # Проверка региона
        if region and 'regions' in flag:
            if region not in flag['regions']:
                return False

        # Проверка tier
        if tier and 'tiers' in flag:
            if tier not in flag['tiers']:
                return False

        # Percentage rollout (consistent hashing)
        percentage = flag.get('percentage', 100)
        if user_id:
            hash_val = int(hashlib.md5(
                f"{feature}:{user_id}".encode()
            ).hexdigest(), 16) % 100
            return hash_val < percentage

        return True

# Flask decorator
feature_flags = FeatureFlags(os.environ['FEATURE_FLAGS_PATH'])

def feature_flag(feature: str):
    def decorator(f):
        @wraps(f)
        def wrapped(*args, **kwargs):
            user_id = getattr(g, 'user_id', None)
            region = request.headers.get('X-User-Region')
            tier = getattr(g, 'user_tier', None)

            if feature_flags.is_enabled(feature, user_id, region, tier):
                return f(*args, **kwargs)
            else:
                # Fallback или 404
                return {"error": "Feature not available"}, 404
        return wrapped
    return decorator

# Использование
@app.route('/checkout')
@feature_flag('new_checkout_flow')
def new_checkout():
    return render_template('checkout_v2.html')
```

### Feature Flags с Argo Rollouts Analysis

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: app-with-feature-flags
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest
        env:
        - name: FEATURE_NEW_CHECKOUT
          value: "true"  # Включено в canary
  strategy:
    canary:
      steps:
      - setWeight: 10
      # Анализ метрик с включённым feature flag
      - analysis:
          templates:
          - templateName: feature-flag-analysis
          args:
          - name: feature-name
            value: new_checkout
      - setWeight: 50
      - pause: {duration: 30m}
      - setWeight: 100
---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: feature-flag-analysis
spec:
  args:
  - name: feature-name
  metrics:
  - name: feature-conversion-rate
    interval: 5m
    successCondition: result[0] >= 0.95  # Не хуже 95% от baseline
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          sum(rate(checkout_completed_total{feature="{{args.feature-name}}"}[10m]))
          /
          sum(rate(checkout_started_total{feature="{{args.feature-name}}"}[10m]))
```

---

## GitOps подходы

GitOps — методология, где Git репозиторий является единственным источником истины для инфраструктуры и конфигурации приложений. Изменения применяются автоматически при коммитах.

### Принципы GitOps

1. **Declarative** — всё описано декларативно в Git
2. **Versioned and Immutable** — история изменений сохраняется
3. **Pulled Automatically** — изменения применяются автоматически
4. **Continuously Reconciled** — система постоянно синхронизируется

### Argo CD

Argo CD — декларативный GitOps CD инструмент для Kubernetes.

#### Установка Argo CD

```bash
# Создание namespace
kubectl create namespace argocd

# Установка Argo CD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Получение initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Port-forward для доступа к UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

#### Application CRD

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default

  # Источник манифестов
  source:
    repoURL: https://github.com/myorg/myapp-k8s-manifests.git
    targetRevision: HEAD
    path: overlays/production

    # Для Helm charts
    # helm:
    #   valueFiles:
    #   - values-production.yaml

    # Для Kustomize
    # kustomize:
    #   images:
    #   - myapp=myregistry/myapp:v1.2.3

  # Куда деплоить
  destination:
    server: https://kubernetes.default.svc
    namespace: production

  # Политика синхронизации
  syncPolicy:
    automated:
      # Автоматически синхронизировать при изменениях в Git
      prune: true
      # Автоматически исправлять drift
      selfHeal: true
      # Разрешить пустые ресурсы
      allowEmpty: false
    syncOptions:
    - CreateNamespace=true
    - PrunePropagationPolicy=foreground
    - PruneLast=true

    # Retry при ошибках
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

#### ApplicationSet для множества окружений

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp-environments
  namespace: argocd
spec:
  generators:
  # Генератор на основе списка
  - list:
      elements:
      - environment: development
        namespace: dev
        revision: develop
      - environment: staging
        namespace: staging
        revision: release
      - environment: production
        namespace: production
        revision: main

  template:
    metadata:
      name: 'myapp-{{environment}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/myapp-k8s-manifests.git
        targetRevision: '{{revision}}'
        path: 'overlays/{{environment}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

#### Argo CD с Argo Rollouts

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-with-rollouts
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/myapp.git
    targetRevision: HEAD
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
  # Игнорировать изменения в rollout status
  ignoreDifferences:
  - group: argoproj.io
    kind: Rollout
    jsonPointers:
    - /status
```

### Flux CD

Flux — GitOps оператор от CNCF.

#### Установка Flux

```bash
# Установка Flux CLI
curl -s https://fluxcd.io/install.sh | sudo bash

# Bootstrap Flux в кластер
flux bootstrap github \
  --owner=myorg \
  --repository=fleet-infra \
  --branch=main \
  --path=clusters/production \
  --personal
```

#### GitRepository и Kustomization

```yaml
# Определение Git репозитория
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: myapp
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/myorg/myapp-k8s-manifests
  ref:
    branch: main
  secretRef:
    name: github-credentials
---
# Kustomization для применения манифестов
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: myapp
  namespace: flux-system
spec:
  interval: 5m
  path: ./overlays/production
  prune: true
  sourceRef:
    kind: GitRepository
    name: myapp
  healthChecks:
  - apiVersion: apps/v1
    kind: Deployment
    name: myapp
    namespace: production
  timeout: 2m
```

#### Flux с Helm

```yaml
# HelmRepository
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: bitnami
  namespace: flux-system
spec:
  interval: 1h
  url: https://charts.bitnami.com/bitnami
---
# HelmRelease
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: postgresql
  namespace: production
spec:
  interval: 5m
  chart:
    spec:
      chart: postgresql
      version: "12.x"
      sourceRef:
        kind: HelmRepository
        name: bitnami
        namespace: flux-system
  values:
    primary:
      persistence:
        size: 50Gi
    auth:
      postgresPassword: ${POSTGRES_PASSWORD}
  valuesFrom:
  - kind: Secret
    name: postgresql-values
    valuesKey: values.yaml
```

#### Image Automation с Flux

```yaml
# Сканирование container registry
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageRepository
metadata:
  name: myapp
  namespace: flux-system
spec:
  image: myregistry/myapp
  interval: 1m
  secretRef:
    name: registry-credentials
---
# Политика выбора тега
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImagePolicy
metadata:
  name: myapp
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: myapp
  policy:
    semver:
      range: ">=1.0.0"
    # или по времени
    # alphabetical:
    #   order: desc
---
# Автоматическое обновление манифестов
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageUpdateAutomation
metadata:
  name: myapp
  namespace: flux-system
spec:
  interval: 1m
  sourceRef:
    kind: GitRepository
    name: myapp
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        email: flux@myorg.com
        name: Flux
      messageTemplate: |
        Automated image update

        Automation: {{ .AutomationObject }}
        Images:
        {{ range .Updated.Images -}}
        - {{ . }}
        {{ end -}}
    push:
      branch: main
  update:
    path: ./overlays/production
    strategy: Setters
```

### Структура GitOps репозитория

```
fleet-infra/
├── clusters/
│   ├── production/
│   │   ├── flux-system/          # Flux bootstrap
│   │   ├── infrastructure.yaml   # Ссылка на infrastructure
│   │   └── apps.yaml             # Ссылка на приложения
│   └── staging/
│       ├── flux-system/
│       ├── infrastructure.yaml
│       └── apps.yaml
├── infrastructure/
│   ├── base/
│   │   ├── cert-manager/
│   │   ├── ingress-nginx/
│   │   └── monitoring/
│   └── overlays/
│       ├── production/
│       └── staging/
└── apps/
    ├── base/
    │   ├── myapp/
    │   │   ├── deployment.yaml
    │   │   ├── service.yaml
    │   │   └── kustomization.yaml
    │   └── api/
    └── overlays/
        ├── production/
        │   ├── myapp/
        │   │   ├── kustomization.yaml
        │   │   └── patch-replicas.yaml
        │   └── kustomization.yaml
        └── staging/
```

---

## Rollback стратегии

Rollback — критически важная операция для восстановления после неудачного деплоя. Kubernetes предоставляет несколько механизмов отката.

### Встроенный Rollback в Deployment

```bash
# Просмотр истории ревизий
kubectl rollout history deployment/myapp

# Детали конкретной ревизии
kubectl rollout history deployment/myapp --revision=3

# Откат к предыдущей ревизии
kubectl rollout undo deployment/myapp

# Откат к конкретной ревизии
kubectl rollout undo deployment/myapp --to-revision=3

# Проверка статуса отката
kubectl rollout status deployment/myapp
```

### Сохранение истории ревизий

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  # Количество сохраняемых ReplicaSet'ов для rollback
  revisionHistoryLimit: 10
  template:
    metadata:
      annotations:
        # Причина изменения (для истории)
        kubernetes.io/change-cause: "Update to v2.1.0 - new feature X"
```

### Автоматический Rollback при ошибках

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  # Время ожидания прогресса
  progressDeadlineSeconds: 300
  template:
    spec:
      containers:
      - name: app
        image: myapp:v2
        # Обязательные probes для автоматического rollback
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3
```

### Rollback в Argo Rollouts

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  strategy:
    canary:
      steps:
      - setWeight: 20
      - analysis:
          templates:
          - templateName: error-rate-check
      - setWeight: 50
      - pause: {duration: 5m}
      - setWeight: 100

      # Автоматический откат при провале анализа
      analysis:
        successfulRunHistoryLimit: 3
        unsuccessfulRunHistoryLimit: 3

      # Anti-affinity для rollback
      antiAffinity:
        requiredDuringSchedulingIgnoredDuringExecution: {}
---
# AnalysisTemplate с условием отката
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: error-rate-check
spec:
  metrics:
  - name: error-rate
    interval: 30s
    # При превышении 5% ошибок - rollback
    failureCondition: result[0] >= 0.05
    failureLimit: 3
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          sum(rate(http_requests_total{status=~"5.."}[5m]))
          /
          sum(rate(http_requests_total[5m]))
```

### Ручной Rollback в Argo Rollouts

```bash
# Отмена текущего rollout (откат)
kubectl argo rollouts abort myapp

# Откат к предыдущей версии
kubectl argo rollouts undo myapp

# Откат к конкретной ревизии
kubectl argo rollouts undo myapp --to-revision=3

# Retry после abort
kubectl argo rollouts retry rollout myapp
```

### Rollback в GitOps (Argo CD)

```bash
# Откат через Argo CD CLI
argocd app rollback myapp

# Откат к конкретной истории
argocd app history myapp
argocd app rollback myapp 5

# Синхронизация на конкретный коммит
argocd app sync myapp --revision abc123
```

### Rollback в GitOps (Flux)

```bash
# Git revert и push
git revert HEAD
git push

# Или через Flux suspend/resume
flux suspend kustomization myapp
# Исправить манифесты
flux resume kustomization myapp
```

### Автоматический Rollback с AlertManager

```yaml
# PrometheusRule для срабатывания rollback
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: rollback-triggers
spec:
  groups:
  - name: rollback
    rules:
    - alert: HighErrorRate
      expr: |
        sum(rate(http_requests_total{status=~"5.."}[5m])) by (deployment)
        /
        sum(rate(http_requests_total[5m])) by (deployment)
        > 0.10
      for: 2m
      labels:
        severity: critical
        action: rollback
      annotations:
        summary: "High error rate detected"
        runbook_url: "https://runbooks.example.com/rollback"
---
# AlertManager webhook для автоматического rollback
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
data:
  alertmanager.yml: |
    receivers:
    - name: rollback-webhook
      webhook_configs:
      - url: 'http://rollback-controller:8080/rollback'
        send_resolved: false

    route:
      receiver: default
      routes:
      - match:
          action: rollback
        receiver: rollback-webhook
```

### Rollback Controller

```yaml
# Простой контроллер для автоматического rollback
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rollback-controller
spec:
  replicas: 1
  selector:
    matchLabels:
      app: rollback-controller
  template:
    metadata:
      labels:
        app: rollback-controller
    spec:
      serviceAccountName: rollback-controller
      containers:
      - name: controller
        image: rollback-controller:latest
        ports:
        - containerPort: 8080
        env:
        - name: ROLLBACK_ENABLED
          value: "true"
        - name: DRY_RUN
          value: "false"
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: rollback-controller
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: rollback-controller
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "patch"]
- apiGroups: ["argoproj.io"]
  resources: ["rollouts"]
  verbs: ["get", "list", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: rollback-controller
subjects:
- kind: ServiceAccount
  name: rollback-controller
  namespace: default
roleRef:
  kind: ClusterRole
  name: rollback-controller
  apiGroup: rbac.authorization.k8s.io
```

---

## Сравнение и выбор стратегии

### Матрица выбора стратегии

| Требование | Рекомендуемая стратегия |
|------------|------------------------|
| Минимальное время деплоя | Rolling Update |
| Zero-downtime обязателен | Rolling Update, Blue-Green, Canary |
| Мгновенный rollback | Blue-Green |
| Минимум ресурсов | Rolling Update, Recreate |
| Тестирование на production трафике | Canary, Shadow |
| A/B тестирование бизнес-метрик | A/B Testing |
| Stateful приложения | Recreate |
| Микросервисы | Canary с service mesh |
| Инфраструктура как код | GitOps (Argo CD, Flux) |

### Комбинирование стратегий

```yaml
# Пример: GitOps + Canary + Feature Flags + Automatic Rollback
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/myorg/myapp.git
    path: k8s
  destination:
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
---
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:v2
        env:
        - name: FEATURE_NEW_UI
          value: "true"
  strategy:
    canary:
      steps:
      - setWeight: 5
      - analysis:
          templates:
          - templateName: comprehensive-check
      - setWeight: 25
      - pause: {duration: 10m}
      - analysis:
          templates:
          - templateName: comprehensive-check
      - setWeight: 50
      - pause: {duration: 30m}
      - setWeight: 100
---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: comprehensive-check
spec:
  metrics:
  # Проверка error rate
  - name: error-rate
    failureCondition: result[0] >= 0.01
    provider:
      prometheus:
        query: sum(rate(http_errors[5m]))/sum(rate(http_requests[5m]))

  # Проверка latency
  - name: latency-p99
    failureCondition: result[0] >= 500
    provider:
      prometheus:
        query: histogram_quantile(0.99, sum(rate(http_duration_bucket[5m])) by (le)) * 1000

  # Проверка CPU usage
  - name: cpu-usage
    failureCondition: result[0] >= 80
    provider:
      prometheus:
        query: avg(rate(container_cpu_usage_seconds_total{pod=~"myapp.*"}[5m])) * 100
```

---

## Лучшие практики

### 1. Всегда используйте Health Checks

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3
  successThreshold: 1

livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 10
  failureThreshold: 3

startupProbe:
  httpGet:
    path: /startup
    port: 8080
  initialDelaySeconds: 0
  periodSeconds: 5
  failureThreshold: 30
```

### 2. Graceful Shutdown

```yaml
spec:
  terminationGracePeriodSeconds: 60
  containers:
  - name: app
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "sleep 10 && kill -SIGTERM 1"]
```

### 3. Resource Limits

```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "200m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

### 4. Pod Disruption Budget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 2
  # или maxUnavailable: 1
  selector:
    matchLabels:
      app: myapp
```

### 5. Мониторинг и алерты

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
  - port: metrics
    interval: 15s
---
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: myapp-alerts
spec:
  groups:
  - name: deployment
    rules:
    - alert: DeploymentReplicasMismatch
      expr: |
        kube_deployment_spec_replicas != kube_deployment_status_available_replicas
      for: 10m
      labels:
        severity: warning
```

---

## Заключение

Выбор стратегии деплоя зависит от множества факторов:

1. **Rolling Update** — стандартный выбор для stateless приложений
2. **Recreate** — для stateful приложений и legacy систем
3. **Blue-Green** — когда нужен мгновенный rollback
4. **Canary** — для постепенного внедрения с проверкой метрик
5. **A/B Testing** — для бизнес-экспериментов
6. **Shadow** — для тестирования без риска
7. **Feature Flags** — для гибкого управления функциональностью
8. **GitOps** — для автоматизации и аудита

Комбинация нескольких подходов (например, GitOps + Canary + Feature Flags) обеспечивает максимальную безопасность и гибкость процесса деплоя.
