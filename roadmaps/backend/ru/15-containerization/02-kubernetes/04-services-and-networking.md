# Services и Networking в Kubernetes

## Введение

Сетевая модель Kubernetes обеспечивает коммуникацию между подами, сервисами и внешним миром. Основные принципы:

- Каждый под получает уникальный IP-адрес
- Поды могут общаться друг с другом без NAT
- Агенты на узле могут общаться со всеми подами на этом узле

---

## Типы сервисов (Services)

Service — абстракция, определяющая логический набор подов и политику доступа к ним.

### 1. ClusterIP (по умолчанию)

Создаёт внутренний IP-адрес, доступный только внутри кластера.

```
┌─────────────────────────────────────────────────────────┐
│                     Kubernetes Cluster                   │
│                                                          │
│   ┌─────────────┐         ┌─────────────┐               │
│   │   Pod A     │         │   Pod B     │               │
│   │ 10.244.1.5  │         │ 10.244.2.8  │               │
│   └──────┬──────┘         └──────┬──────┘               │
│          │                       │                       │
│          └───────────┬───────────┘                       │
│                      │                                   │
│              ┌───────▼───────┐                          │
│              │   ClusterIP   │                          │
│              │  10.96.0.100  │                          │
│              │   :80/TCP     │                          │
│              └───────────────┘                          │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

**Пример манифеста:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: default
spec:
  type: ClusterIP  # можно опустить, это значение по умолчанию
  selector:
    app: my-app
  ports:
    - name: http
      protocol: TCP
      port: 80        # порт сервиса
      targetPort: 8080 # порт контейнера
```

**Использование:**
- Внутренние микросервисы
- Базы данных
- Кэш-серверы (Redis, Memcached)

### 2. NodePort

Открывает порт на каждом узле кластера (диапазон 30000-32767).

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│   Внешний клиент                                                 │
│        │                                                         │
│        ▼                                                         │
│   ┌─────────┐    ┌─────────┐    ┌─────────┐                     │
│   │ Node 1  │    │ Node 2  │    │ Node 3  │                     │
│   │ :30080  │    │ :30080  │    │ :30080  │                     │
│   └────┬────┘    └────┬────┘    └────┬────┘                     │
│        │              │              │                           │
│        └──────────────┼──────────────┘                           │
│                       │                                          │
│               ┌───────▼───────┐                                  │
│               │   ClusterIP   │                                  │
│               │  10.96.0.100  │                                  │
│               └───────┬───────┘                                  │
│                       │                                          │
│          ┌────────────┼────────────┐                             │
│          ▼            ▼            ▼                             │
│     ┌────────┐   ┌────────┐   ┌────────┐                        │
│     │  Pod   │   │  Pod   │   │  Pod   │                        │
│     └────────┘   └────────┘   └────────┘                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Пример манифеста:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nodeport-service
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080  # опционально, Kubernetes выберет автоматически
```

**Использование:**
- Разработка и тестирование
- Демонстрации
- Когда нет облачного Load Balancer

### 3. LoadBalancer

Создаёт внешний балансировщик нагрузки (в облачных провайдерах).

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│   Интернет                                                       │
│      │                                                           │
│      ▼                                                           │
│   ┌──────────────────────┐                                       │
│   │   Cloud Load         │                                       │
│   │   Balancer           │                                       │
│   │   203.0.113.50:80    │                                       │
│   └──────────┬───────────┘                                       │
│              │                                                   │
│   ┌──────────▼───────────────────────────────────────┐          │
│   │              Kubernetes Cluster                   │          │
│   │                                                   │          │
│   │   ┌─────────┐    ┌─────────┐    ┌─────────┐     │          │
│   │   │ Node 1  │    │ Node 2  │    │ Node 3  │     │          │
│   │   │ :30080  │    │ :30080  │    │ :30080  │     │          │
│   │   └────┬────┘    └────┬────┘    └────┬────┘     │          │
│   │        │              │              │           │          │
│   │        └──────────────┼──────────────┘           │          │
│   │                       ▼                          │          │
│   │                  ┌────────┐                      │          │
│   │                  │  Pods  │                      │          │
│   │                  └────────┘                      │          │
│   │                                                   │          │
│   └───────────────────────────────────────────────────┘          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Пример манифеста:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-loadbalancer-service
  annotations:
    # Аннотации зависят от облачного провайдера
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
  # Опционально: ограничить внешний трафик
  externalTrafficPolicy: Local  # или Cluster (по умолчанию)
```

**Использование:**
- Продакшн-окружения
- Публичные API
- Web-приложения

### 4. ExternalName

Маппинг сервиса на внешний DNS-адрес.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-external-db
spec:
  type: ExternalName
  externalName: database.example.com
```

**Использование:**
- Интеграция с внешними сервисами
- Миграция в Kubernetes (сначала ExternalName, потом замена на внутренний сервис)

### Сравнительная таблица типов сервисов

| Тип | Внутренний доступ | Внешний доступ | Use Case |
|-----|-------------------|----------------|----------|
| ClusterIP | ✅ | ❌ | Внутренние сервисы |
| NodePort | ✅ | ✅ (через IP узла) | Dev/Test |
| LoadBalancer | ✅ | ✅ (через LB) | Production |
| ExternalName | ❌ | Redirect | Внешние сервисы |

---

## Headless Services

Сервис без ClusterIP для прямого доступа к подам.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-headless-service
spec:
  clusterIP: None  # делает сервис headless
  selector:
    app: my-stateful-app
  ports:
    - port: 5432
```

**DNS-записи для headless service:**
```
my-headless-service.default.svc.cluster.local → IP всех подов
pod-0.my-headless-service.default.svc.cluster.local → IP pod-0
```

**Использование:**
- StatefulSets (базы данных, очереди сообщений)
- Когда клиент сам выбирает под

---

## Ingress

Ingress управляет внешним HTTP/HTTPS доступом к сервисам.

### Архитектура Ingress

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│   Интернет                                                       │
│      │                                                           │
│      ▼                                                           │
│   ┌──────────────────────┐                                       │
│   │   Load Balancer      │                                       │
│   │   (Cloud/MetalLB)    │                                       │
│   └──────────┬───────────┘                                       │
│              │                                                   │
│   ┌──────────▼───────────────────────────────────────┐          │
│   │              Ingress Controller                   │          │
│   │         (nginx, traefik, istio)                  │          │
│   └──────────┬───────────────────────────────────────┘          │
│              │                                                   │
│              │    ┌─── Ingress Rules ────┐                      │
│              │    │                       │                      │
│              │    │ api.example.com       │                      │
│              │    │   └─► api-service     │                      │
│              │    │                       │                      │
│              │    │ web.example.com       │                      │
│              │    │   └─► web-service     │                      │
│              │    │                       │                      │
│              │    │ example.com/api       │                      │
│              │    │   └─► api-service     │                      │
│              │    │                       │                      │
│              │    │ example.com/          │                      │
│              │    │   └─► web-service     │                      │
│              │    └───────────────────────┘                      │
│              │                                                   │
│         ┌────┴────┬────────────┐                                │
│         ▼         ▼            ▼                                 │
│   ┌──────────┐ ┌──────────┐ ┌──────────┐                        │
│   │ api-svc  │ │ web-svc  │ │ auth-svc │                        │
│   └────┬─────┘ └────┬─────┘ └────┬─────┘                        │
│        ▼            ▼            ▼                               │
│   ┌────────┐   ┌────────┐   ┌────────┐                          │
│   │  Pods  │   │  Pods  │   │  Pods  │                          │
│   └────────┘   └────────┘   └────────┘                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Ingress Controllers

Ingress сам по себе — только ресурс с правилами. Нужен Ingress Controller для их выполнения.

| Controller | Особенности |
|------------|-------------|
| **nginx-ingress** | Самый популярный, много функций |
| **Traefik** | Автоматическое обнаружение, Let's Encrypt |
| **HAProxy** | Высокая производительность |
| **Istio Gateway** | Часть Service Mesh |
| **Kong** | API Gateway функции |
| **AWS ALB** | Нативная интеграция с AWS |

**Установка nginx-ingress через Helm:**

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace
```

### Примеры Ingress

#### Простой Ingress с одним хостом

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: simple-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-service
                port:
                  number: 80
```

#### Ingress с несколькими хостами

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-host-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
    - host: web.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
```

#### Ingress с маршрутизацией по путям

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-based-ingress
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  ingressClassName: nginx
  rules:
    - host: example.com
      http:
        paths:
          - path: /api(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: api-service
                port:
                  number: 80
          - path: /admin
            pathType: Prefix
            backend:
              service:
                name: admin-service
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

#### Ingress с TLS (HTTPS)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - secure.example.com
      secretName: tls-secret  # Secret с сертификатом
  rules:
    - host: secure.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: secure-service
                port:
                  number: 443
```

**Создание TLS Secret:**

```bash
kubectl create secret tls tls-secret \
  --cert=path/to/tls.crt \
  --key=path/to/tls.key
```

#### Ingress с аннотациями nginx

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: annotated-ingress
  annotations:
    # Редирект HTTP → HTTPS
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # Таймауты
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"

    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "10"

    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://allowed-origin.com"

    # WebSocket support
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"

    # Basic Auth
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"

    # Custom headers
    nginx.ingress.kubernetes.io/configuration-snippet: |
      add_header X-Frame-Options "SAMEORIGIN";
      add_header X-Content-Type-Options "nosniff";
spec:
  ingressClassName: nginx
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-service
                port:
                  number: 80
```

### Path Types

| Type | Описание | Пример |
|------|----------|--------|
| Exact | Точное совпадение | `/foo` соответствует только `/foo` |
| Prefix | Префиксное совпадение | `/foo` соответствует `/foo`, `/foo/bar` |
| ImplementationSpecific | Зависит от IngressClass | Поддержка regex и т.д. |

---

## DNS в Kubernetes

Kubernetes использует CoreDNS для разрешения имён внутри кластера.

### Формат DNS-имён

```
<service-name>.<namespace>.svc.<cluster-domain>
```

**Примеры:**
```
# Полное имя
my-service.default.svc.cluster.local

# Короткое имя (в том же namespace)
my-service

# С namespace
my-service.production
```

### DNS для различных ресурсов

```
┌──────────────────────────────────────────────────────────────────┐
│                        DNS в Kubernetes                           │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Service (ClusterIP):                                             │
│    my-svc.my-ns.svc.cluster.local → ClusterIP                    │
│                                                                   │
│  Headless Service:                                                │
│    my-svc.my-ns.svc.cluster.local → IP всех подов                │
│    pod-name.my-svc.my-ns.svc.cluster.local → IP конкретного пода │
│                                                                   │
│  Pod (обычный):                                                   │
│    10-244-1-5.my-ns.pod.cluster.local → Pod IP                   │
│                                                                   │
│  StatefulSet Pod:                                                 │
│    web-0.my-svc.my-ns.svc.cluster.local → Pod IP                 │
│    web-1.my-svc.my-ns.svc.cluster.local → Pod IP                 │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### Конфигурация DNS для подов

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
    - name: app
      image: my-app
  dnsPolicy: ClusterFirst  # по умолчанию
  dnsConfig:
    nameservers:
      - 1.1.1.1  # дополнительный DNS-сервер
    searches:
      - my-ns.svc.cluster.local
    options:
      - name: ndots
        value: "2"
```

**DNS Policies:**
- `ClusterFirst` — сначала кластерный DNS, затем внешний
- `ClusterFirstWithHostNet` — для подов с hostNetwork: true
- `Default` — DNS-конфигурация узла
- `None` — полностью кастомная конфигурация

### Настройка CoreDNS

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf {
           max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
    # Кастомный домен
    example.local:53 {
        forward . 10.0.0.10
    }
```

---

## Network Policies

Network Policy — файрвол на уровне подов, контролирующий входящий и исходящий трафик.

### Принцип работы

```
┌─────────────────────────────────────────────────────────────────┐
│                     Network Policy                               │
│                                                                  │
│   ┌─────────────┐                         ┌─────────────┐       │
│   │   Pod A     │───────── ✅ ──────────►│   Pod B     │       │
│   │ app=frontend│  (разрешено политикой) │ app=backend │       │
│   └─────────────┘                         └─────────────┘       │
│                                                  │               │
│   ┌─────────────┐                                │               │
│   │   Pod C     │─────────── ❌ ─────────────────┘               │
│   │ app=other   │  (запрещено политикой)                        │
│   └─────────────┘                                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Важно:**
- По умолчанию весь трафик разрешён
- Network Policy применяется только к подам с matching labels
- Политики аддитивны (накапливаются)
- Требуется сетевой плагин с поддержкой Network Policies (Calico, Cilium, Weave)

### Примеры Network Policies

#### Deny All Ingress (запретить весь входящий трафик)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: production
spec:
  podSelector: {}  # применяется ко всем подам
  policyTypes:
    - Ingress
  # ingress не указан = всё запрещено
```

#### Deny All Egress (запретить весь исходящий трафик)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  # egress не указан = всё запрещено
```

#### Allow traffic from specific pods

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend  # к каким подам применяется
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend  # от каких подов разрешено
      ports:
        - protocol: TCP
          port: 8080
```

#### Allow traffic from namespace

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-monitoring
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: my-app
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: monitoring
      ports:
        - protocol: TCP
          port: 9090
```

#### Allow traffic from external IPs

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-ip
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
    - Ingress
  ingress:
    - from:
        - ipBlock:
            cidr: 172.17.0.0/16
            except:
              - 172.17.1.0/24
      ports:
        - protocol: TCP
          port: 80
```

#### Complex policy (комплексная политика)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-network-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
      tier: backend
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # Разрешить трафик от frontend
    - from:
        - podSelector:
            matchLabels:
              app: frontend
        - namespaceSelector:
            matchLabels:
              project: myproject
      ports:
        - protocol: TCP
          port: 8080
    # Разрешить трафик от ingress controller
    - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx
      ports:
        - protocol: TCP
          port: 8080
  egress:
    # Разрешить доступ к базе данных
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432
    # Разрешить DNS
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
    # Разрешить HTTPS наружу
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
      ports:
        - protocol: TCP
          port: 443
```

#### Database isolation policy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-isolation
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: postgres
      role: database
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # Только от backend подов
    - from:
        - podSelector:
            matchLabels:
              tier: backend
      ports:
        - protocol: TCP
          port: 5432
  egress:
    # Только DNS для разрешения имён
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
```

### Логические операторы в Network Policies

```yaml
# AND — все условия в одном from/to блоке
ingress:
  - from:
      - podSelector:
            matchLabels:
              role: frontend
        namespaceSelector:  # И podSelector, И namespaceSelector
            matchLabels:
              project: myproject

# OR — разные элементы в списке from/to
ingress:
  - from:
      - podSelector:
            matchLabels:
              role: frontend
      - namespaceSelector:  # ИЛИ podSelector, ИЛИ namespaceSelector
            matchLabels:
              project: myproject
```

---

## Service Mesh

Service Mesh — инфраструктурный слой для управления service-to-service коммуникацией.

### Архитектура Service Mesh

```
┌─────────────────────────────────────────────────────────────────┐
│                       Service Mesh                               │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    Control Plane                         │   │
│   │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │   │
│   │  │   Config    │  │  Service    │  │  Telemetry  │     │   │
│   │  │   (Galley)  │  │  Discovery  │  │   (Mixer)   │     │   │
│   │  └─────────────┘  └─────────────┘  └─────────────┘     │   │
│   │                          │                               │   │
│   │                  ┌───────▼───────┐                       │   │
│   │                  │    Pilot      │                       │   │
│   │                  │ (Istiod/etc)  │                       │   │
│   │                  └───────┬───────┘                       │   │
│   └──────────────────────────┼──────────────────────────────┘   │
│                              │                                   │
│   ┌──────────────────────────▼──────────────────────────────┐   │
│   │                     Data Plane                           │   │
│   │                                                          │   │
│   │   ┌─────────────────┐         ┌─────────────────┐       │   │
│   │   │      Pod A      │         │      Pod B      │       │   │
│   │   │  ┌───────────┐  │         │  ┌───────────┐  │       │   │
│   │   │  │    App    │  │◄───────►│  │    App    │  │       │   │
│   │   │  └─────┬─────┘  │  mTLS   │  └─────┬─────┘  │       │   │
│   │   │        │        │         │        │        │       │   │
│   │   │  ┌─────▼─────┐  │         │  ┌─────▼─────┐  │       │   │
│   │   │  │  Sidecar  │  │         │  │  Sidecar  │  │       │   │
│   │   │  │  Proxy    │  │         │  │  Proxy    │  │       │   │
│   │   │  │ (Envoy)   │  │         │  │ (Envoy)   │  │       │   │
│   │   │  └───────────┘  │         │  └───────────┘  │       │   │
│   │   └─────────────────┘         └─────────────────┘       │   │
│   │                                                          │   │
│   └──────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Возможности Service Mesh

| Функция | Описание |
|---------|----------|
| **mTLS** | Шифрование трафика между сервисами |
| **Traffic Management** | Canary deployments, A/B testing, traffic splitting |
| **Observability** | Распределённый трейсинг, метрики, логирование |
| **Resiliency** | Retries, timeouts, circuit breakers |
| **Security** | Политики авторизации, RBAC |
| **Rate Limiting** | Ограничение количества запросов |

### Istio

Наиболее популярный Service Mesh с богатым функционалом.

**Установка Istio:**

```bash
# Установка istioctl
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH=$PWD/bin:$PATH

# Установка Istio в кластер
istioctl install --set profile=demo -y

# Включение sidecar injection для namespace
kubectl label namespace default istio-injection=enabled
```

**Пример VirtualService (traffic routing):**

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: my-service-vs
spec:
  hosts:
    - my-service
  http:
    # Canary deployment: 90% на v1, 10% на v2
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

**Пример DestinationRule:**

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: my-service-dr
spec:
  host: my-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: UPGRADE
        http1MaxPendingRequests: 100
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```

**Пример Gateway:**

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: my-gateway
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
        credentialName: my-tls-secret
      hosts:
        - "*.example.com"
```

**Пример AuthorizationPolicy:**

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: frontend-to-backend
  namespace: production
spec:
  selector:
    matchLabels:
      app: backend
  action: ALLOW
  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/production/sa/frontend"]
      to:
        - operation:
            methods: ["GET", "POST"]
            paths: ["/api/*"]
```

### Linkerd

Легковесный Service Mesh, ориентированный на простоту.

**Установка Linkerd:**

```bash
# Установка CLI
curl -sL run.linkerd.io/install | sh
export PATH=$PATH:$HOME/.linkerd2/bin

# Проверка кластера
linkerd check --pre

# Установка в кластер
linkerd install | kubectl apply -f -

# Проверка установки
linkerd check

# Добавление мешинга в namespace
kubectl get deploy -n my-namespace -o yaml | linkerd inject - | kubectl apply -f -
```

**Особенности Linkerd:**
- Меньше потребление ресурсов
- Проще в настройке
- Rust-based proxy (не Envoy)
- Встроенная поддержка mTLS

**Пример ServiceProfile (traffic policy):**

```yaml
apiVersion: linkerd.io/v1alpha2
kind: ServiceProfile
metadata:
  name: my-service.production.svc.cluster.local
  namespace: production
spec:
  routes:
    - name: GET /api/users
      condition:
        method: GET
        pathRegex: /api/users
      responseClasses:
        - condition:
            status:
              min: 500
              max: 599
          isFailure: true
      timeout: 1s
      retryBudget:
        retryRatio: 0.2
        minRetriesPerSecond: 10
        ttl: 10s
```

### Сравнение Service Mesh решений

| Характеристика | Istio | Linkerd | Cilium |
|----------------|-------|---------|--------|
| **Прокси** | Envoy | linkerd2-proxy (Rust) | eBPF |
| **Сложность** | Высокая | Низкая | Средняя |
| **Ресурсы** | Высокие | Низкие | Низкие |
| **mTLS** | ✅ | ✅ | ✅ |
| **Traffic splitting** | ✅ | ✅ | ✅ |
| **Multi-cluster** | ✅ | ✅ | ✅ |
| **GUI Dashboard** | Kiali | Linkerd Viz | Hubble |

---

## Практические сценарии

### Сценарий 1: Микросервисная архитектура

```yaml
# Frontend Service
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  type: ClusterIP
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 3000
---
# Backend API Service
apiVersion: v1
kind: Service
metadata:
  name: backend-api
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 8080
---
# Database Service (Headless для StatefulSet)
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - port: 5432
---
# Ingress для внешнего доступа
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.example.com
      secretName: myapp-tls
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend-api
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
---
# Network Policy для изоляции
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
```

### Сценарий 2: Blue-Green Deployment с Ingress

```yaml
# Blue deployment service
apiVersion: v1
kind: Service
metadata:
  name: app-blue
spec:
  selector:
    app: myapp
    version: blue
  ports:
    - port: 80
---
# Green deployment service
apiVersion: v1
kind: Service
metadata:
  name: app-green
spec:
  selector:
    app: myapp
    version: green
  ports:
    - port: 80
---
# Ingress переключается между blue и green
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-blue  # или app-green для переключения
                port:
                  number: 80
```

---

## Best Practices

### Services

1. **Используйте ClusterIP** для внутренних сервисов
2. **Именуйте порты** для удобства и интеграции с Service Mesh
3. **Используйте readinessProbe** для правильной работы endpoints
4. **Избегайте NodePort в production** — используйте LoadBalancer или Ingress

### Ingress

1. **Один Ingress Controller** на кластер (или по одному на namespace для изоляции)
2. **TLS обязателен** для production
3. **Rate limiting** для защиты от DDoS
4. **Health checks** через annotations
5. **Используйте IngressClass** для выбора контроллера

### Network Policies

1. **Default Deny** — начинайте с запрета всего трафика
2. **Принцип наименьших привилегий** — разрешайте только необходимое
3. **Не забывайте про DNS** — разрешите доступ к kube-dns
4. **Тестируйте политики** перед применением в production
5. **Документируйте** все политики

### Service Mesh

1. **Начинайте с простого** — не включайте всё сразу
2. **Мониторинг обязателен** — без observability mesh бесполезен
3. **Учитывайте overhead** — sidecar потребляет ресурсы
4. **Постепенный rollout** — включайте mesh для отдельных namespace

---

## Полезные команды

```bash
# Просмотр сервисов
kubectl get svc -A
kubectl describe svc my-service

# Endpoints (куда направляется трафик)
kubectl get endpoints my-service

# Ingress
kubectl get ingress
kubectl describe ingress my-ingress

# Network Policies
kubectl get networkpolicies
kubectl describe networkpolicy my-policy

# DNS отладка
kubectl run dns-test --image=busybox:1.28 --rm -it --restart=Never -- nslookup my-service

# Проверка связности
kubectl run curl-test --image=curlimages/curl --rm -it --restart=Never -- curl my-service:80

# Просмотр Ingress Controller логов
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx

# Istio команды
istioctl analyze
istioctl proxy-status
istioctl dashboard kiali

# Linkerd команды
linkerd check
linkerd stat deploy
linkerd dashboard
```

---

## Заключение

Сетевая модель Kubernetes предоставляет гибкие инструменты для организации коммуникации между сервисами:

- **Services** — базовая абстракция для доступа к подам
- **Ingress** — управление HTTP/HTTPS трафиком извне
- **Network Policies** — файрвол на уровне подов
- **Service Mesh** — продвинутое управление трафиком и безопасностью

Правильное использование этих инструментов позволяет создавать безопасные, масштабируемые и отказоустойчивые приложения в Kubernetes.
