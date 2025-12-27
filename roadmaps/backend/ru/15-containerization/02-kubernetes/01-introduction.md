# Введение в Kubernetes

## Что такое Kubernetes?

**Kubernetes** (K8s) — это открытая платформа для автоматизации развёртывания, масштабирования и управления контейнеризированными приложениями. Проект был создан в Google на основе их внутренней системы Borg и передан в Cloud Native Computing Foundation (CNCF) в 2014 году.

### Основные задачи Kubernetes

- **Оркестрация контейнеров** — управление жизненным циклом контейнеров
- **Автоматическое масштабирование** — увеличение/уменьшение количества реплик
- **Самовосстановление** — перезапуск упавших контейнеров, замена узлов
- **Балансировка нагрузки** — распределение трафика между подами
- **Service Discovery** — автоматическое обнаружение сервисов
- **Управление конфигурацией** — хранение секретов и настроек
- **Декларативное управление** — описание желаемого состояния в YAML

---

## Архитектура кластера Kubernetes

Кластер Kubernetes состоит из двух основных компонентов: **Control Plane** (плоскость управления) и **Worker Nodes** (рабочие узлы).

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           KUBERNETES CLUSTER                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                        CONTROL PLANE                              │  │
│  │                                                                   │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐ │  │
│  │  │  API Server  │  │  Scheduler   │  │  Controller Manager      │ │  │
│  │  │   (kube-     │  │  (kube-      │  │  (kube-controller-       │ │  │
│  │  │   apiserver) │  │  scheduler)  │  │   manager)               │ │  │
│  │  └──────┬───────┘  └──────────────┘  └──────────────────────────┘ │  │
│  │         │                                                         │  │
│  │  ┌──────┴───────┐  ┌──────────────────────────────────────────┐   │  │
│  │  │     etcd     │  │  Cloud Controller Manager (optional)     │   │  │
│  │  │  (key-value  │  │  (cloud-controller-manager)              │   │  │
│  │  │   store)     │  └──────────────────────────────────────────┘   │  │
│  │  └──────────────┘                                                 │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                    │                                    │
│                                    │ API calls                          │
│                                    ▼                                    │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                         WORKER NODES                              │  │
│  │                                                                   │  │
│  │  ┌─────────────────────────────┐  ┌─────────────────────────────┐ │  │
│  │  │         NODE 1              │  │         NODE 2              │ │  │
│  │  │  ┌────────┐ ┌────────┐     │  │  ┌────────┐ ┌────────┐     │ │  │
│  │  │  │ kubelet│ │kube-   │     │  │  │ kubelet│ │kube-   │     │ │  │
│  │  │  │        │ │proxy   │     │  │  │        │ │proxy   │     │ │  │
│  │  │  └────────┘ └────────┘     │  │  └────────┘ └────────┘     │ │  │
│  │  │  ┌────────────────────┐    │  │  ┌────────────────────┐    │ │  │
│  │  │  │ Container Runtime  │    │  │  │ Container Runtime  │    │ │  │
│  │  │  │ (containerd/CRI-O) │    │  │  │ (containerd/CRI-O) │    │ │  │
│  │  │  └────────────────────┘    │  │  └────────────────────┘    │ │  │
│  │  │  ┌─────┐ ┌─────┐ ┌─────┐  │  │  ┌─────┐ ┌─────┐ ┌─────┐  │ │  │
│  │  │  │ Pod │ │ Pod │ │ Pod │  │  │  │ Pod │ │ Pod │ │ Pod │  │ │  │
│  │  │  └─────┘ └─────┘ └─────┘  │  │  └─────┘ └─────┘ └─────┘  │ │  │
│  │  └─────────────────────────────┘  └─────────────────────────────┘ │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

### Control Plane (Плоскость управления)

Control Plane отвечает за глобальные решения о кластере и реагирует на события.

#### kube-apiserver
- **Центральный компонент** кластера
- Предоставляет REST API для взаимодействия с кластером
- Точка входа для всех административных задач
- Валидирует и обрабатывает запросы

```bash
# Все команды kubectl работают через API Server
kubectl get pods
kubectl apply -f deployment.yaml
kubectl describe node worker-1
```

#### etcd
- **Распределённое хранилище** ключ-значение
- Хранит всё состояние кластера
- Критически важный компонент — требует резервного копирования
- Используется только API Server

```bash
# Бэкап etcd
ETCDCTL_API=3 etcdctl snapshot save backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

#### kube-scheduler
- Назначает поды на узлы
- Учитывает ресурсные требования, ограничения, affinity/anti-affinity
- Выбирает оптимальный узел для размещения пода

#### kube-controller-manager
- Запускает контроллеры (циклы управления)
- **Node Controller** — отслеживает состояние узлов
- **Replication Controller** — поддерживает нужное количество реплик
- **Endpoints Controller** — связывает Services и Pods
- **Service Account Controller** — создаёт аккаунты для неймспейсов

#### cloud-controller-manager (опционально)
- Интеграция с облачными провайдерами
- Управление Load Balancers, Nodes, Routes
- Используется в AWS, GCP, Azure и других облаках

### Worker Nodes (Рабочие узлы)

На каждом узле запущены компоненты для выполнения приложений.

#### kubelet
- **Агент** на каждом узле
- Получает спецификации подов от API Server
- Следит за запуском и здоровьем контейнеров
- Отчитывается о состоянии узла

#### kube-proxy
- Сетевой прокси на каждом узле
- Реализует абстракцию Service
- Управляет правилами iptables/IPVS
- Обеспечивает связь между подами и внешним миром

#### Container Runtime
- Программа для запуска контейнеров
- **containerd** — стандарт de facto
- **CRI-O** — легковесная альтернатива
- Docker (устарел как runtime с K8s 1.24)

---

## Ключевые концепции Kubernetes

### Pod (Под)

**Pod** — минимальная единица развёртывания в Kubernetes. Один или несколько контейнеров с общими ресурсами.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  labels:
    app: web
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

### ReplicaSet и Deployment

**ReplicaSet** обеспечивает заданное количество реплик пода.
**Deployment** — высокоуровневая абстракция для декларативного обновления.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```

### Service (Сервис)

**Service** — абстракция для доступа к группе подов. Типы сервисов:

| Тип | Описание |
|-----|----------|
| **ClusterIP** | Внутренний IP в кластере (по умолчанию) |
| **NodePort** | Открывает порт на всех узлах |
| **LoadBalancer** | Внешний балансировщик (облако) |
| **ExternalName** | CNAME запись DNS |

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

### Namespace (Пространство имён)

Логическое разделение ресурсов кластера.

```bash
# Стандартные namespace
kubectl get namespaces
# default          - ресурсы по умолчанию
# kube-system      - системные компоненты
# kube-public      - публичные ресурсы
# kube-node-lease  - heartbeat узлов
```

### ConfigMap и Secret

Хранение конфигурации и секретов отдельно от образов.

```yaml
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_HOST: "postgres.default.svc.cluster.local"
  LOG_LEVEL: "info"

---
# Secret (base64 encoded)
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  DB_PASSWORD: cGFzc3dvcmQxMjM=  # password123
```

### Ingress

Управление внешним HTTP/HTTPS доступом к сервисам.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
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
            name: web-service
            port:
              number: 80
```

---

## Взаимодействие компонентов

```
┌─────────────────────────────────────────────────────────────────┐
│                    Жизненный цикл запроса                       │
└─────────────────────────────────────────────────────────────────┘

   User
     │
     │ kubectl apply -f deployment.yaml
     ▼
┌─────────────┐
│ API Server  │◄─────────────────────────────────────┐
└──────┬──────┘                                      │
       │                                              │
       │ 1. Сохранение в etcd                        │
       ▼                                              │
┌─────────────┐                                      │
│    etcd     │                                      │
└──────┬──────┘                                      │
       │                                              │
       │ 2. Событие: новый Deployment                │
       ▼                                              │
┌─────────────────────┐                              │
│ Controller Manager  │                              │
│ (Deployment         │                              │
│  Controller)        │                              │
└──────┬──────────────┘                              │
       │                                              │
       │ 3. Создаёт ReplicaSet → Pods                │
       ▼                                              │
┌─────────────┐                                      │
│  Scheduler  │                                      │
└──────┬──────┘                                      │
       │                                              │
       │ 4. Назначает Pod на Node                    │
       ▼                                              │
┌─────────────┐        ┌─────────────┐               │
│   kubelet   │◄───────│  API Server │               │
│  (на Node)  │        │  (watch)    │               │
└──────┬──────┘        └─────────────┘               │
       │                                              │
       │ 5. Запускает контейнеры                     │
       ▼                                              │
┌─────────────┐                                      │
│ containerd  │                                      │
└──────┬──────┘                                      │
       │                                              │
       │ 6. Отчёт о статусе                          │
       └──────────────────────────────────────────────┘
```

---

## Альтернативы Kubernetes

### Docker Swarm

| Аспект | Docker Swarm | Kubernetes |
|--------|--------------|------------|
| Сложность | Простой в освоении | Крутая кривая обучения |
| Масштаб | Малые/средние проекты | Любой масштаб |
| Функциональность | Базовая | Богатая экосистема |
| Состояние | Режим поддержки | Активная разработка |

```bash
# Docker Swarm
docker swarm init
docker service create --replicas 3 nginx

# Kubernetes
kubectl create deployment nginx --image=nginx --replicas=3
```

### Nomad (HashiCorp)

- **Преимущества**: Простота, поддержка не только контейнеров
- **Использование**: VM, контейнеры, Java-приложения
- **Интеграция**: Consul, Vault
- **Когда выбирать**: Гетерогенные workloads, простые требования

### Amazon ECS / AWS Fargate

- **Преимущества**: Интеграция с AWS, Serverless (Fargate)
- **Недостатки**: Vendor lock-in
- **Когда выбирать**: Полностью в экосистеме AWS

### Podman + systemd

- **Преимущества**: Без daemon, rootless контейнеры
- **Использование**: Локальная разработка, простые deployments
- **Когда выбирать**: Единичные сервера, безопасность

### Сравнительная таблица

| Решение | Сложность | Масштаб | Облачная интеграция | Экосистема |
|---------|-----------|---------|---------------------|------------|
| Kubernetes | Высокая | Огромный | Все провайдеры | Богатейшая |
| Docker Swarm | Низкая | Средний | Ограниченная | Базовая |
| Nomad | Средняя | Большой | HashiCloud | Умеренная |
| ECS/Fargate | Средняя | Большой | AWS только | AWS-specific |
| Podman | Низкая | Малый | Нет | Минимальная |

---

## Best Practices

### 1. Организация ресурсов

```yaml
# Используйте namespaces для изоляции
kubectl create namespace production
kubectl create namespace staging
kubectl create namespace development

# Применяйте labels для организации
metadata:
  labels:
    app: web-api
    environment: production
    team: backend
    version: v1.2.3
```

### 2. Resource Limits

**Всегда** указывайте requests и limits:

```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "100m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

### 3. Health Checks

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

### 4. Декларативный подход

```bash
# Плохо (императивно)
kubectl create deployment nginx --image=nginx
kubectl scale deployment nginx --replicas=3

# Хорошо (декларативно)
kubectl apply -f deployment.yaml
# Все изменения через Git (GitOps)
```

### 5. Безопасность

```yaml
# Не запускайте от root
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
```

### 6. Используйте конкретные теги образов

```yaml
# Плохо
image: nginx:latest

# Хорошо
image: nginx:1.25.3-alpine
```

---

## Инструменты для работы с Kubernetes

### kubectl — основной CLI

```bash
# Установка (macOS)
brew install kubectl

# Проверка версии
kubectl version --client

# Основные команды
kubectl get pods                    # Список подов
kubectl describe pod <name>         # Детали пода
kubectl logs <pod-name>             # Логи
kubectl exec -it <pod> -- /bin/sh   # Shell в контейнер
kubectl apply -f manifest.yaml      # Применить конфигурацию
kubectl delete -f manifest.yaml     # Удалить ресурсы
```

### Helm — менеджер пакетов

```bash
# Установка
brew install helm

# Добавление репозитория
helm repo add bitnami https://charts.bitnami.com/bitnami

# Установка приложения
helm install my-redis bitnami/redis

# Обновление
helm upgrade my-redis bitnami/redis --set auth.enabled=false
```

### k9s — терминальный UI

```bash
# Установка
brew install k9s

# Запуск
k9s
```

### Lens — графический UI

Десктопное приложение для управления кластерами с визуализацией ресурсов.

---

## Полезные команды для начала

```bash
# Информация о кластере
kubectl cluster-info
kubectl get nodes -o wide

# Работа с ресурсами
kubectl get all -A                      # Все ресурсы во всех namespace
kubectl api-resources                   # Список типов ресурсов
kubectl explain pod.spec.containers     # Документация по полям

# Отладка
kubectl describe pod <name>             # Детальная информация
kubectl logs -f <pod> -c <container>    # Следить за логами
kubectl port-forward pod/<name> 8080:80 # Проброс порта
kubectl top pods                        # Потребление ресурсов

# Быстрый запуск для тестов
kubectl run test --image=busybox --rm -it -- /bin/sh
```

---

## Резюме

- **Kubernetes** — стандарт оркестрации контейнеров
- **Control Plane** управляет кластером (API Server, etcd, Scheduler, Controllers)
- **Worker Nodes** выполняют workloads (kubelet, kube-proxy, container runtime)
- **Ключевые объекты**: Pod, Deployment, Service, ConfigMap, Secret, Ingress
- **Декларативный подход** — описываем желаемое состояние в YAML
- **GitOps** — хранение конфигурации в Git, автоматическое применение
- **Альтернативы** существуют, но K8s — индустриальный стандарт для масштабных систем
