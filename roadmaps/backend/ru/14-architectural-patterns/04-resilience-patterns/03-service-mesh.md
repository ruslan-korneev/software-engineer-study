# Service Mesh

## Определение

**Service Mesh** — это выделенный инфраструктурный слой для управления коммуникацией между микросервисами. Он обеспечивает наблюдаемость, безопасность и отказоустойчивость межсервисного взаимодействия без изменения кода приложений.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Service Mesh Architecture                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    Control Plane                         │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐ │    │
│  │  │ Config   │  │ Service  │  │ Cert     │  │ Metrics  │ │    │
│  │  │ Manager  │  │ Discovery│  │ Authority│  │ Collector│ │    │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘ │    │
│  └─────────────────────────────────────────────────────────┘    │
│                              │                                   │
│                              ▼                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                     Data Plane                           │    │
│  │                                                          │    │
│  │  ┌───────────────┐         ┌───────────────┐            │    │
│  │  │ Service A     │         │ Service B     │            │    │
│  │  │ ┌───────────┐ │         │ ┌───────────┐ │            │    │
│  │  │ │   App     │ │◄───────►│ │   App     │ │            │    │
│  │  │ └─────┬─────┘ │         │ └─────┬─────┘ │            │    │
│  │  │       │       │         │       │       │            │    │
│  │  │ ┌─────▼─────┐ │         │ ┌─────▼─────┐ │            │    │
│  │  │ │  Sidecar  │ │◄───────►│ │  Sidecar  │ │            │    │
│  │  │ │  Proxy    │ │  mTLS   │ │  Proxy    │ │            │    │
│  │  │ └───────────┘ │         │ └───────────┘ │            │    │
│  │  └───────────────┘         └───────────────┘            │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

Service Mesh решает проблемы, связанные с микросервисной архитектурой:
- Обнаружение сервисов
- Балансировка нагрузки
- Шифрование трафика (mTLS)
- Наблюдаемость (metrics, tracing, logging)
- Управление трафиком (routing, canary, A/B)
- Отказоустойчивость (retries, circuit breaker, timeouts)

## Ключевые характеристики

### Архитектура: Data Plane + Control Plane

#### Data Plane (Плоскость данных)

Sidecar proxy, развёрнутый рядом с каждым сервисом:

```yaml
# Kubernetes Pod с sidecar proxy
apiVersion: v1
kind: Pod
metadata:
  name: my-service
  annotations:
    sidecar.istio.io/inject: "true"  # Автоматическая инъекция
spec:
  containers:
  - name: my-app
    image: my-app:v1
    ports:
    - containerPort: 8080
  # Sidecar добавляется автоматически:
  # - name: istio-proxy
  #   image: envoyproxy/envoy
  #   ports:
  #   - containerPort: 15001  # Inbound
  #   - containerPort: 15006  # Outbound
```

Функции Data Plane:
- Перехват всего входящего и исходящего трафика
- TLS/mTLS терминация и инициация
- Балансировка нагрузки
- Health checking
- Сбор метрик и трейсов

#### Control Plane (Плоскость управления)

Централизованное управление конфигурацией:

```
┌────────────────────────────────────────────────────────────┐
│                      Control Plane                          │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    │
│  │   Pilot     │    │   Citadel   │    │   Galley    │    │
│  │  (Config)   │    │   (Certs)   │    │  (Validate) │    │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘    │
│         │                   │                   │          │
│         └───────────────────┼───────────────────┘          │
│                             │                               │
│                             ▼                               │
│                    ┌──────────────┐                        │
│                    │   xDS API    │                        │
│                    │ (gRPC stream)│                        │
│                    └──────────────┘                        │
│                                                             │
└────────────────────────────────────────────────────────────┘
                              │
                              ▼ Push configuration
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
         [Sidecar 1]    [Sidecar 2]    [Sidecar N]
```

### Основные возможности

#### 1. Traffic Management

```yaml
# Istio VirtualService для canary deployment
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: my-service
spec:
  hosts:
  - my-service
  http:
  - match:
    - headers:
        x-canary:
          exact: "true"
    route:
    - destination:
        host: my-service
        subset: v2
  - route:
    - destination:
        host: my-service
        subset: v1
      weight: 90
    - destination:
        host: my-service
        subset: v2
      weight: 10
```

#### 2. Resilience (Отказоустойчивость)

```yaml
# Istio DestinationRule с resilience настройками
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: my-service
spec:
  host: my-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: UPGRADE
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000

    # Circuit Breaker
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50

    # Retries
    loadBalancer:
      simple: ROUND_ROBIN

  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

#### 3. Security (mTLS)

```yaml
# Istio PeerAuthentication для mTLS
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT  # Требовать mTLS для всех коммуникаций

---
# Authorization Policy
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: payment-service
  namespace: production
spec:
  selector:
    matchLabels:
      app: payment-service
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/production/sa/order-service"]
    to:
    - operation:
        methods: ["POST"]
        paths: ["/api/payments/*"]
```

#### 4. Observability (Наблюдаемость)

```
┌────────────────────────────────────────────────────────────┐
│                    Observability Stack                      │
├────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌──────────────┐                                         │
│   │   Metrics    │ ──► Prometheus ──► Grafana              │
│   │  (RED/USE)   │     (scrape)       (dashboards)         │
│   └──────────────┘                                         │
│                                                             │
│   ┌──────────────┐                                         │
│   │   Tracing    │ ──► Jaeger / Zipkin / Tempo             │
│   │ (spans/ctx)  │     (distributed tracing)               │
│   └──────────────┘                                         │
│                                                             │
│   ┌──────────────┐                                         │
│   │   Logging    │ ──► Fluentd ──► Elasticsearch/Loki      │
│   │ (access log) │     (collect)    (store/query)          │
│   └──────────────┘                                         │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

## Когда использовать

### Идеальные сценарии

1. **Большое количество микросервисов (10+)**
   - Сложно управлять конфигурацией в коде каждого сервиса
   - Нужна единая точка управления политиками

2. **Требования к безопасности**
   - Zero-trust networking
   - mTLS между всеми сервисами
   - Fine-grained access control

3. **Сложные deployment стратегии**
   - Canary releases
   - A/B тестирование
   - Traffic mirroring

4. **Полиглотная среда**
   - Сервисы на разных языках
   - Единые политики независимо от языка

5. **Требования к observability**
   - Distributed tracing из коробки
   - Единые метрики для всех сервисов

### Когда НЕ использовать

1. **Малое количество сервисов (<5)**
   - Overhead больше пользы
   - Проще реализовать в коде приложения

2. **Простые deployment модели**
   - Нет canary/blue-green
   - Один дата-центр

3. **Ограниченные ресурсы**
   - Sidecar потребляет CPU/RAM
   - Сложность operations

4. **Требования к минимальной latency**
   - Sidecar добавляет ~1-3ms latency

## Преимущества и недостатки

### Преимущества

| Преимущество | Описание |
|--------------|----------|
| **Единообразие** | Одинаковое поведение для всех сервисов |
| **Language-agnostic** | Не зависит от языка программирования |
| **Разделение ответственности** | Разработчики фокусируются на бизнес-логике |
| **mTLS из коробки** | Шифрование без изменения кода |
| **Observability** | Metrics, traces, logs автоматически |
| **Гибкое управление трафиком** | Canary, A/B, fault injection |
| **Централизованное управление** | Политики из одного места |

### Недостатки

| Недостаток | Описание |
|------------|----------|
| **Сложность** | Крутая кривая обучения |
| **Resource overhead** | Sidecar на каждый pod |
| **Latency** | Дополнительный hop через proxy |
| **Debugging** | Сложнее понять проблемы |
| **Зависимость** | Vendor lock-in или OSS complexity |
| **Управление** | Требует DevOps/Platform expertise |

## Примеры реализации

### Istio: Базовая конфигурация

```yaml
# 1. Установка Istio
# istioctl install --set profile=production

# 2. Включение sidecar injection для namespace
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    istio-injection: enabled

---
# 3. Gateway для входящего трафика
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: main-gateway
  namespace: production
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: main-tls-cert
    hosts:
    - "api.example.com"

---
# 4. VirtualService для routing
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api-routes
  namespace: production
spec:
  hosts:
  - "api.example.com"
  gateways:
  - main-gateway
  http:
  - match:
    - uri:
        prefix: /api/orders
    route:
    - destination:
        host: order-service
        port:
          number: 8080
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: connect-failure,refused-stream,503
    timeout: 10s

  - match:
    - uri:
        prefix: /api/users
    route:
    - destination:
        host: user-service
        port:
          number: 8080
```

### Linkerd: Простая конфигурация

```yaml
# 1. Установка Linkerd
# linkerd install | kubectl apply -f -

# 2. Аннотация для injection
apiVersion: v1
kind: Namespace
metadata:
  name: production
  annotations:
    linkerd.io/inject: enabled

---
# 3. ServiceProfile для advanced routing
apiVersion: linkerd.io/v1alpha2
kind: ServiceProfile
metadata:
  name: order-service.production.svc.cluster.local
  namespace: production
spec:
  routes:
  - name: POST /api/orders
    condition:
      method: POST
      pathRegex: /api/orders
    isRetryable: false  # Не retry для создания

  - name: GET /api/orders/{id}
    condition:
      method: GET
      pathRegex: /api/orders/[^/]+
    isRetryable: true
    timeout: 5s

  retryBudget:
    retryRatio: 0.2
    minRetriesPerSecond: 10
    ttl: 10s
```

### Envoy: Низкоуровневая конфигурация

```yaml
# Envoy sidecar конфигурация
static_resources:
  listeners:
  - name: inbound_listener
    address:
      socket_address:
        address: 0.0.0.0
        port_value: 15001
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: ingress_http
          route_config:
            name: local_route
            virtual_hosts:
            - name: local_service
              domains: ["*"]
              routes:
              - match:
                  prefix: "/"
                route:
                  cluster: local_service
                  timeout: 10s
                  retry_policy:
                    retry_on: "5xx,reset,connect-failure"
                    num_retries: 3
                    per_try_timeout: 3s
          http_filters:
          - name: envoy.filters.http.router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router

  clusters:
  - name: local_service
    connect_timeout: 0.25s
    type: STATIC
    lb_policy: ROUND_ROBIN
    load_assignment:
      cluster_name: local_service
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: 127.0.0.1
                port_value: 8080

    # Circuit Breaker
    circuit_breakers:
      thresholds:
      - priority: DEFAULT
        max_connections: 100
        max_pending_requests: 100
        max_requests: 1000
        max_retries: 3

    # Outlier Detection (Circuit Breaker)
    outlier_detection:
      consecutive_5xx: 5
      interval: 10s
      base_ejection_time: 30s
      max_ejection_percent: 50
```

### Практический пример: Canary Deployment

```yaml
# Полный пример canary deployment с Istio

# 1. Deployment для v1
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service-v1
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment-service
      version: v1
  template:
    metadata:
      labels:
        app: payment-service
        version: v1
    spec:
      containers:
      - name: payment-service
        image: payment-service:v1.0.0
        ports:
        - containerPort: 8080

---
# 2. Deployment для v2 (canary)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service-v2
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: payment-service
      version: v2
  template:
    metadata:
      labels:
        app: payment-service
        version: v2
    spec:
      containers:
      - name: payment-service
        image: payment-service:v2.0.0
        ports:
        - containerPort: 8080

---
# 3. Service (общий для обеих версий)
apiVersion: v1
kind: Service
metadata:
  name: payment-service
  namespace: production
spec:
  selector:
    app: payment-service
  ports:
  - port: 8080

---
# 4. DestinationRule для subsets
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-service
  namespace: production
spec:
  host: payment-service
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  trafficPolicy:
    connectionPool:
      http:
        http2MaxRequests: 1000
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s

---
# 5. VirtualService для traffic splitting
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-service
  namespace: production
spec:
  hosts:
  - payment-service
  http:
  # Header-based routing для тестирования
  - match:
    - headers:
        x-canary:
          exact: "true"
    route:
    - destination:
        host: payment-service
        subset: v2

  # Weighted routing: 90% v1, 10% v2
  - route:
    - destination:
        host: payment-service
        subset: v1
      weight: 90
    - destination:
        host: payment-service
        subset: v2
      weight: 10

    # Retries только для v2 (осторожнее с новой версией)
    retries:
      attempts: 2
      perTryTimeout: 2s

    # Mirroring для shadow traffic
    mirror:
      host: payment-service
      subset: v2
    mirrorPercentage:
      value: 5.0  # 5% трафика зеркалируется
```

## Best Practices и антипаттерны

### Best Practices

#### 1. Постепенное внедрение

```yaml
# Начинаем с permissive mode
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
spec:
  mtls:
    mode: PERMISSIVE  # Сначала опционально

# Потом переходим на strict
# mtls:
#   mode: STRICT
```

#### 2. Правильная настройка ресурсов sidecar

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    sidecar.istio.io/proxyCPU: "100m"
    sidecar.istio.io/proxyMemory: "128Mi"
    sidecar.istio.io/proxyCPULimit: "200m"
    sidecar.istio.io/proxyMemoryLimit: "256Mi"
```

#### 3. Ограничение области видимости sidecar

```yaml
# Sidecar видит только нужные сервисы
apiVersion: networking.istio.io/v1beta1
kind: Sidecar
metadata:
  name: order-service
  namespace: production
spec:
  workloadSelector:
    labels:
      app: order-service
  egress:
  - hosts:
    - "./*"                    # Все сервисы в namespace
    - "istio-system/*"         # Istio components
    - "external/payment-gateway.external.svc.cluster.local"
```

#### 4. Мониторинг здоровья mesh

```python
# Prometheus queries для мониторинга
queries = {
    # Latency P99
    "latency_p99": 'histogram_quantile(0.99, sum(rate(istio_request_duration_milliseconds_bucket{reporter="destination"}[5m])) by (le, destination_service_name))',

    # Success rate
    "success_rate": 'sum(rate(istio_requests_total{reporter="destination", response_code!~"5.*"}[5m])) by (destination_service_name) / sum(rate(istio_requests_total{reporter="destination"}[5m])) by (destination_service_name)',

    # Circuit breaker ejections
    "cb_ejections": 'sum(rate(envoy_cluster_outlier_detection_ejections_total[5m])) by (cluster_name)',
}
```

### Антипаттерны

#### 1. Игнорирование mTLS проблем

```yaml
# Плохо: отключение mTLS при первых проблемах
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: disable-mtls  # НЕ ДЕЛАЙТЕ ТАК
spec:
  mtls:
    mode: DISABLE

# Лучше: debugging и исправление root cause
# kubectl exec -it <pod> -c istio-proxy -- curl localhost:15000/certs
```

#### 2. Слишком сложные правила маршрутизации

```yaml
# Плохо: сложная логика, трудно отлаживать
spec:
  http:
  - match:
    - headers:
        x-region:
          regex: "eu-.*"
        x-tier:
          exact: "premium"
      queryParams:
        version:
          exact: "2"
    # ... много условий
```

#### 3. Отсутствие стратегии rollback

```yaml
# Всегда имейте возможность откатиться
# Используйте GitOps для управления конфигурацией

# ArgoCD Application для mesh config
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: mesh-config
spec:
  source:
    repoURL: https://github.com/company/mesh-config
    path: production
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
    namespace: istio-system
  syncPolicy:
    automated:
      prune: false  # Не удалять ресурсы автоматически
      selfHeal: true
```

## Связанные паттерны

### Service Mesh и другие паттерны

```
┌────────────────────────────────────────────────────────────┐
│                Service Mesh encompasses:                    │
├────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌─────────────────┐   ┌─────────────────┐               │
│   │ Circuit Breaker │   │   Rate Limiting  │               │
│   │ (outlier detect)│   │   (Throttling)   │               │
│   └─────────────────┘   └─────────────────┘               │
│                                                             │
│   ┌─────────────────┐   ┌─────────────────┐               │
│   │     Retry       │   │     Timeout      │               │
│   │  with Backoff   │   │   Management     │               │
│   └─────────────────┘   └─────────────────┘               │
│                                                             │
│   ┌─────────────────┐   ┌─────────────────┐               │
│   │   Load Balancing│   │ Service Discovery│               │
│   │                 │   │                  │               │
│   └─────────────────┘   └─────────────────┘               │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

| Паттерн | Реализация в Service Mesh |
|---------|---------------------------|
| **Circuit Breaker** | Outlier detection в sidecar |
| **Rate Limiting** | Local/global rate limit filters |
| **Retry** | Configurable в VirtualService |
| **Timeout** | Request timeout в route config |
| **Bulkhead** | Connection pools per service |
| **Health Check** | Active/passive health checking |

### Service Mesh vs API Gateway

```
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│   External    ┌─────────────┐    Internal                   │
│   Traffic ──► │ API Gateway │ ──► Traffic                   │
│               └─────────────┘                                │
│                     │                                        │
│   • Authentication  │                                        │
│   • Rate limiting   │                                        │
│   • Protocol trans  │    ┌─────────────────────────────┐    │
│   • API versioning  │    │      Service Mesh           │    │
│                     └───►│                             │    │
│                          │  • mTLS                     │    │
│                          │  • Service-to-service auth  │    │
│                          │  • Internal load balancing  │    │
│                          │  • Observability            │    │
│                          └─────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Ресурсы для изучения

### Официальная документация
- [Istio Documentation](https://istio.io/latest/docs/)
- [Linkerd Documentation](https://linkerd.io/2/overview/)
- [Envoy Proxy Documentation](https://www.envoyproxy.io/docs/envoy/latest/)
- [Consul Connect](https://www.consul.io/docs/connect)

### Книги
- "Istio in Action" — Christian Posta, Rinor Maloku
- "The Enterprise Path to Service Mesh Architectures" — Lee Calcote
- "Cloud Native Patterns" — Cornelia Davis

### Курсы
- [Istio Fundamentals](https://academy.tetrate.io/courses/istio-fundamentals)
- [Linkerd Training](https://linkerd.io/2/tasks/installing-multicluster/)

### Инструменты
- **Kiali**: Dashboard для Istio
- **Jaeger/Zipkin**: Distributed tracing
- **Grafana**: Dashboards для mesh metrics

### Comparison
| Feature | Istio | Linkerd | Consul Connect |
|---------|-------|---------|----------------|
| Proxy | Envoy | linkerd2-proxy | Envoy |
| Complexity | High | Low | Medium |
| Performance | Good | Excellent | Good |
| mTLS | Yes | Yes | Yes |
| Multi-cluster | Yes | Yes | Yes |
| Learning curve | Steep | Gentle | Medium |
