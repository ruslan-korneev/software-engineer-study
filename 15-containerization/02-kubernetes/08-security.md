# Security в Kubernetes

Безопасность в Kubernetes — это комплексная тема, охватывающая множество уровней: от контроля доступа до сетевой изоляции и защиты секретов. Kubernetes предоставляет мощные инструменты для реализации принципа "defense in depth" (глубокая защита).

## Модель безопасности Kubernetes

Kubernetes использует многоуровневую модель безопасности:

```
┌─────────────────────────────────────────────────────────────┐
│                    Cluster Level                             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │               Namespace Level                           │ │
│  │  ┌───────────────────────────────────────────────────┐ │ │
│  │  │                 Pod Level                          │ │ │
│  │  │  ┌──────────────────────────────────────────────┐ │ │ │
│  │  │  │            Container Level                    │ │ │ │
│  │  │  └──────────────────────────────────────────────┘ │ │ │
│  │  └───────────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

**4C безопасности:**
1. **Cloud** — безопасность облачной инфраструктуры
2. **Cluster** — безопасность кластера Kubernetes
3. **Container** — безопасность контейнеров
4. **Code** — безопасность приложения

---

## RBAC (Role-Based Access Control)

RBAC — это механизм контроля доступа, основанный на ролях. Он определяет, кто (Subject) может выполнять какие действия (Verbs) над какими ресурсами (Resources).

### Основные компоненты RBAC

```
┌──────────────────────────────────────────────────────────────┐
│                        RBAC Model                             │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│   Subject ──────────► RoleBinding ──────────► Role            │
│   (User,               (связывает)            (разрешения     │
│    Group,                                      в namespace)   │
│    ServiceAccount)                                            │
│                                                               │
│   Subject ──────────► ClusterRoleBinding ───► ClusterRole     │
│                        (связывает)            (разрешения     │
│                                                на уровне      │
│                                                кластера)      │
└──────────────────────────────────────────────────────────────┘
```

### Role и ClusterRole

**Role** — определяет разрешения в рамках одного namespace:

```yaml
# role-pod-reader.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: pod-reader
rules:
  # Разрешение читать pods
  - apiGroups: [""]           # "" означает core API group
    resources: ["pods"]
    verbs: ["get", "watch", "list"]

  # Разрешение читать логи pods
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get"]
```

**ClusterRole** — определяет разрешения на уровне всего кластера:

```yaml
# clusterrole-node-reader.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
  # Разрешение читать nodes (cluster-scoped ресурс)
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "watch", "list"]

  # Разрешение читать persistent volumes
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list"]
```

**Расширенный пример Role с различными ресурсами:**

```yaml
# role-developer.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: developer
rules:
  # Полный доступ к deployments
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

  # Доступ к pods (чтение + exec)
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods/exec", "pods/log"]
    verbs: ["create", "get"]

  # Доступ к services
  - apiGroups: [""]
    resources: ["services"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]

  # Доступ к ConfigMaps и Secrets
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list"]  # Только чтение для secrets

  # Доступ к конкретным ресурсам по имени
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["app-config", "feature-flags"]
    verbs: ["update", "patch"]
```

### Доступные Verbs

| Verb | HTTP метод | Описание |
|------|------------|----------|
| `get` | GET | Получить конкретный ресурс |
| `list` | GET | Получить список ресурсов |
| `watch` | GET (WebSocket) | Следить за изменениями |
| `create` | POST | Создать ресурс |
| `update` | PUT | Обновить ресурс полностью |
| `patch` | PATCH | Частично обновить ресурс |
| `delete` | DELETE | Удалить ресурс |
| `deletecollection` | DELETE | Удалить коллекцию ресурсов |

### RoleBinding и ClusterRoleBinding

**RoleBinding** — связывает Role с пользователями в namespace:

```yaml
# rolebinding-developers.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developers-binding
  namespace: development
subjects:
  # Привязка к конкретному пользователю
  - kind: User
    name: alice@example.com
    apiGroup: rbac.authorization.k8s.io

  # Привязка к группе
  - kind: Group
    name: developers
    apiGroup: rbac.authorization.k8s.io

  # Привязка к ServiceAccount
  - kind: ServiceAccount
    name: ci-bot
    namespace: development
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

**ClusterRoleBinding** — связывает ClusterRole с пользователями на уровне кластера:

```yaml
# clusterrolebinding-cluster-admins.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admins
subjects:
  - kind: Group
    name: platform-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin  # Встроенная ClusterRole
  apiGroup: rbac.authorization.k8s.io
```

**RoleBinding с ClusterRole** — применяет ClusterRole в конкретном namespace:

```yaml
# rolebinding-view-in-namespace.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: view-binding
  namespace: production
subjects:
  - kind: Group
    name: qa-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole      # Используем ClusterRole
  name: view             # Встроенная ClusterRole (только чтение)
  apiGroup: rbac.authorization.k8s.io
```

### Встроенные ClusterRoles

Kubernetes предоставляет несколько встроенных ролей:

| ClusterRole | Описание |
|-------------|----------|
| `cluster-admin` | Полный доступ ко всему кластеру |
| `admin` | Полный доступ в namespace (кроме квот) |
| `edit` | Чтение/запись большинства ресурсов в namespace |
| `view` | Только чтение в namespace (без secrets) |

---

## ServiceAccount

ServiceAccount — это идентичность для процессов, работающих в Pod. Каждый Pod автоматически получает ServiceAccount (по умолчанию `default`).

### Создание ServiceAccount

```yaml
# serviceaccount-app.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  namespace: production
automountServiceAccountToken: false  # Отключить автоматическое монтирование токена
```

### ServiceAccount с ImagePullSecrets

```yaml
# serviceaccount-with-pull-secret.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  namespace: production
imagePullSecrets:
  - name: docker-registry-secret
```

### Использование ServiceAccount в Pod

```yaml
# pod-with-serviceaccount.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  namespace: production
spec:
  serviceAccountName: my-app
  automountServiceAccountToken: true  # Монтировать токен в Pod
  containers:
    - name: app
      image: my-app:latest
```

### ServiceAccount для CI/CD

```yaml
# Полный пример для CI/CD бота
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ci-deployer
  namespace: production
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployment-manager
  namespace: production
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["services", "configmaps"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ci-deployer-binding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: ci-deployer
    namespace: production
roleRef:
  kind: Role
  name: deployment-manager
  apiGroup: rbac.authorization.k8s.io
```

### Получение токена ServiceAccount

```bash
# Kubernetes 1.24+: создание долгоживущего токена
kubectl create token ci-deployer -n production --duration=8760h

# Или через Secret (для совместимости)
```

```yaml
# secret-sa-token.yaml
apiVersion: v1
kind: Secret
metadata:
  name: ci-deployer-token
  namespace: production
  annotations:
    kubernetes.io/service-account.name: ci-deployer
type: kubernetes.io/service-account-token
```

---

## SecurityContext

SecurityContext определяет настройки безопасности для Pod или отдельного контейнера.

### SecurityContext на уровне Pod

```yaml
# pod-security-context.yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    # Запуск всех контейнеров от пользователя с UID 1000
    runAsUser: 1000
    runAsGroup: 3000

    # Запретить запуск от root
    runAsNonRoot: true

    # Дополнительные группы для файловой системы
    fsGroup: 2000
    fsGroupChangePolicy: "OnRootMismatch"

    # SELinux контекст
    seLinuxOptions:
      level: "s0:c123,c456"

    # Seccomp профиль
    seccompProfile:
      type: RuntimeDefault

    # Sysctls
    sysctls:
      - name: net.core.somaxconn
        value: "1024"

  containers:
    - name: app
      image: nginx:alpine
      # Контейнер наследует securityContext от Pod
```

### SecurityContext на уровне контейнера

```yaml
# container-security-context.yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-container
spec:
  containers:
    - name: app
      image: my-app:latest
      securityContext:
        # Запуск от конкретного пользователя
        runAsUser: 1000
        runAsGroup: 1000
        runAsNonRoot: true

        # Запретить повышение привилегий
        allowPrivilegeEscalation: false

        # Только для чтения корневая ФС
        readOnlyRootFilesystem: true

        # Удалить все capabilities и добавить только нужные
        capabilities:
          drop:
            - ALL
          add:
            - NET_BIND_SERVICE

        # Privileged режим (ОПАСНО - использовать только если необходимо)
        privileged: false

        # Seccomp профиль для контейнера
        seccompProfile:
          type: RuntimeDefault
```

### Практический пример: безопасный веб-сервер

```yaml
# secure-nginx.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 101      # nginx user
        runAsGroup: 101     # nginx group
        fsGroup: 101
        seccompProfile:
          type: RuntimeDefault

      containers:
        - name: nginx
          image: nginxinc/nginx-unprivileged:alpine
          ports:
            - containerPort: 8080
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL

          volumeMounts:
            # Временные директории для nginx
            - name: tmp
              mountPath: /tmp
            - name: cache
              mountPath: /var/cache/nginx
            - name: run
              mountPath: /var/run

      volumes:
        - name: tmp
          emptyDir: {}
        - name: cache
          emptyDir: {}
        - name: run
          emptyDir: {}
```

### Linux Capabilities

Некоторые часто используемые capabilities:

| Capability | Описание |
|------------|----------|
| `NET_BIND_SERVICE` | Привязка к портам < 1024 |
| `NET_ADMIN` | Настройка сети |
| `SYS_ADMIN` | Множество административных операций |
| `SYS_PTRACE` | Отладка процессов |
| `CHOWN` | Изменение владельца файлов |
| `DAC_OVERRIDE` | Обход проверки прав доступа |
| `SETUID` / `SETGID` | Изменение UID/GID |

---

## Pod Security Standards

Pod Security Standards (PSS) — это набор политик, определяющих уровни безопасности Pod. Заменяет устаревшие PodSecurityPolicy (PSP).

### Уровни безопасности

| Уровень | Описание |
|---------|----------|
| `privileged` | Без ограничений (для системных workloads) |
| `baseline` | Минимальные ограничения (предотвращает известные эскалации) |
| `restricted` | Максимальные ограничения (best practices) |

### Применение через Namespace labels

```yaml
# namespace-with-pss.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    # Режим enforce — Pod не будет создан при нарушении
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest

    # Режим warn — предупреждение при нарушении
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest

    # Режим audit — запись в audit log
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
```

### Пример Pod, соответствующий restricted уровню

```yaml
# restricted-compliant-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: restricted-pod
  namespace: production
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault

  containers:
    - name: app
      image: my-app:latest
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop:
            - ALL
      resources:
        limits:
          memory: "128Mi"
          cpu: "500m"
        requests:
          memory: "64Mi"
          cpu: "250m"
```

### Что запрещено на уровне restricted

- `hostNetwork`, `hostPID`, `hostIPC`
- `hostPath` volumes
- Privileged containers
- Capabilities кроме `NET_BIND_SERVICE`
- Запуск от root
- `allowPrivilegeEscalation: true`
- Небезопасные seccomp профили

---

## Network Policies

Network Policies — это правила сетевой фильтрации на уровне Pod. По умолчанию все Pod могут общаться друг с другом.

### Типы политик

```
┌─────────────────────────────────────────────────────────────┐
│                    Network Policy                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   Ingress (входящий трафик)                                  │
│   ┌──────────┐      ┌──────────────┐                        │
│   │  Source  │ ───► │  Target Pod  │                        │
│   └──────────┘      └──────────────┘                        │
│                                                              │
│   Egress (исходящий трафик)                                  │
│   ┌──────────────┐      ┌─────────────┐                     │
│   │  Target Pod  │ ───► │ Destination │                     │
│   └──────────────┘      └─────────────┘                     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Deny All — запретить весь трафик

```yaml
# deny-all-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: production
spec:
  podSelector: {}      # Применяется ко всем Pod в namespace
  policyTypes:
    - Ingress
  # ingress: []        # Пустой список = запретить всё
```

```yaml
# deny-all-egress.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  # egress: []
```

### Разрешить трафик между конкретными Pod

```yaml
# allow-frontend-to-backend.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  # Политика применяется к Pod с label app=backend
  podSelector:
    matchLabels:
      app: backend

  policyTypes:
    - Ingress

  ingress:
    # Разрешить входящие соединения от frontend на порт 8080
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
```

### Разрешить трафик из другого namespace

```yaml
# allow-from-monitoring-namespace.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend

  policyTypes:
    - Ingress

  ingress:
    # Разрешить от Pod из namespace monitoring
    - from:
        - namespaceSelector:
            matchLabels:
              name: monitoring
          podSelector:
            matchLabels:
              app: prometheus
      ports:
        - protocol: TCP
          port: 9090
```

### Разрешить доступ к внешним ресурсам

```yaml
# allow-egress-to-external.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend

  policyTypes:
    - Egress

  egress:
    # Разрешить DNS
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53

    # Разрешить HTTPS к внешним адресам
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.0.0.0/8
              - 172.16.0.0/12
              - 192.168.0.0/16
      ports:
        - protocol: TCP
          port: 443
```

### Комплексный пример: микросервисная архитектура

```yaml
# network-policy-microservices.yaml
---
# Запретить всё по умолчанию
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
---
# Frontend может получать трафик от Ingress и отправлять в Backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx
      ports:
        - protocol: TCP
          port: 80
  egress:
    # DNS
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: UDP
          port: 53
    # Backend
    - to:
        - podSelector:
            matchLabels:
              tier: backend
      ports:
        - protocol: TCP
          port: 8080
---
# Backend может получать от Frontend и отправлять в Database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              tier: frontend
      ports:
        - protocol: TCP
          port: 8080
  egress:
    # DNS
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: UDP
          port: 53
    # Database
    - to:
        - podSelector:
            matchLabels:
              tier: database
      ports:
        - protocol: TCP
          port: 5432
---
# Database только принимает от Backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              tier: backend
      ports:
        - protocol: TCP
          port: 5432
  egress:
    # DNS только
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: UDP
          port: 53
```

---

## Secrets

Secrets — объекты для хранения конфиденциальных данных (пароли, токены, ключи).

### Типы Secrets

| Тип | Описание |
|-----|----------|
| `Opaque` | Произвольные данные (по умолчанию) |
| `kubernetes.io/dockerconfigjson` | Данные Docker registry |
| `kubernetes.io/tls` | TLS сертификат и ключ |
| `kubernetes.io/service-account-token` | Токен ServiceAccount |
| `kubernetes.io/basic-auth` | Basic auth credentials |
| `kubernetes.io/ssh-auth` | SSH ключи |

### Создание Secrets

```yaml
# secret-opaque.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: production
type: Opaque
data:
  # Данные в base64 (echo -n 'value' | base64)
  DB_PASSWORD: cGFzc3dvcmQxMjM=
  API_KEY: c2VjcmV0LWFwaS1rZXk=
stringData:
  # Или в открытом виде (будет закодировано при создании)
  DB_HOST: postgres.production.svc.cluster.local
```

```yaml
# secret-tls.yaml
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
  namespace: production
type: kubernetes.io/tls
data:
  tls.crt: LS0tLS1CRUdJTi...  # base64 encoded certificate
  tls.key: LS0tLS1CRUdJTi...  # base64 encoded private key
```

```yaml
# secret-docker-registry.yaml
apiVersion: v1
kind: Secret
metadata:
  name: docker-registry
  namespace: production
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: eyJhdXRocyI6...  # base64 encoded docker config
```

### Использование Secrets в Pod

**Как переменные окружения:**

```yaml
# pod-secret-env.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
    - name: app
      image: my-app:latest
      env:
        # Конкретное значение из Secret
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: DB_PASSWORD
              optional: false

      # Все значения из Secret как переменные окружения
      envFrom:
        - secretRef:
            name: app-secrets
            optional: false
```

**Как файлы через Volume:**

```yaml
# pod-secret-volume.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
    - name: app
      image: my-app:latest
      volumeMounts:
        - name: secrets
          mountPath: /etc/secrets
          readOnly: true

  volumes:
    - name: secrets
      secret:
        secretName: app-secrets
        # Опционально: задать права доступа
        defaultMode: 0400
        # Опционально: выбрать конкретные ключи
        items:
          - key: DB_PASSWORD
            path: db-password
          - key: API_KEY
            path: api-key
```

### Безопасное хранение Secrets

По умолчанию Secrets хранятся в etcd в base64 (НЕ зашифрованы!). Для настоящей безопасности используйте:

**1. Encryption at Rest**

```yaml
# encryption-config.yaml (для kube-apiserver)
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-encoded-32-byte-key>
      - identity: {}  # Fallback для чтения незашифрованных
```

**2. External Secrets Operator**

```yaml
# external-secret.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-secrets
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: app-secrets
  data:
    - secretKey: DB_PASSWORD
      remoteRef:
        key: production/database
        property: password
```

**3. Sealed Secrets (для GitOps)**

```yaml
# sealedsecret.yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: app-secrets
  namespace: production
spec:
  encryptedData:
    DB_PASSWORD: AgBy8hCQ...  # Зашифровано публичным ключом
```

---

## Команды kubectl для работы с безопасностью

### RBAC

```bash
# Проверить права пользователя
kubectl auth can-i create pods --namespace=production
kubectl auth can-i create pods --namespace=production --as=alice@example.com
kubectl auth can-i '*' '*' --as=system:serviceaccount:kube-system:default

# Список всех разрешений для пользователя
kubectl auth can-i --list --as=alice@example.com
kubectl auth can-i --list --as=system:serviceaccount:production:my-app

# Просмотр ролей
kubectl get roles -n production
kubectl get clusterroles
kubectl describe role developer -n production

# Просмотр привязок
kubectl get rolebindings -n production
kubectl get clusterrolebindings
kubectl describe rolebinding developers-binding -n production

# Кто имеет доступ к ресурсу (требует плагин)
kubectl who-can create pods -n production
```

### ServiceAccounts

```bash
# Список ServiceAccounts
kubectl get serviceaccounts -n production
kubectl describe serviceaccount my-app -n production

# Создать токен
kubectl create token my-app -n production
kubectl create token my-app -n production --duration=8760h

# Создать ServiceAccount
kubectl create serviceaccount ci-deployer -n production
```

### Secrets

```bash
# Список Secrets
kubectl get secrets -n production
kubectl describe secret app-secrets -n production

# Создать Secret
kubectl create secret generic app-secrets \
  --from-literal=DB_PASSWORD=mypassword \
  --from-file=api-key=./api-key.txt \
  -n production

# Создать TLS Secret
kubectl create secret tls tls-secret \
  --cert=tls.crt \
  --key=tls.key \
  -n production

# Создать Docker registry Secret
kubectl create secret docker-registry docker-creds \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=password \
  -n production

# Получить значение (декодировать base64)
kubectl get secret app-secrets -n production -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
```

### Network Policies

```bash
# Список Network Policies
kubectl get networkpolicies -n production
kubectl describe networkpolicy allow-frontend -n production

# Применить политику
kubectl apply -f network-policy.yaml
```

### Pod Security

```bash
# Проверить namespace labels
kubectl get namespace production --show-labels

# Добавить PSS labels
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/warn=restricted

# Проверить нарушения (dry-run)
kubectl label --dry-run=server --overwrite namespace production \
  pod-security.kubernetes.io/enforce=restricted
```

### Аудит и диагностика

```bash
# Проверить SecurityContext Pod
kubectl get pod my-pod -n production -o yaml | grep -A 20 securityContext

# Просмотреть события безопасности
kubectl get events -n production --field-selector reason=FailedCreate

# Проверить, под каким пользователем работает контейнер
kubectl exec -it my-pod -n production -- id
kubectl exec -it my-pod -n production -- whoami

# Проверить capabilities
kubectl exec -it my-pod -n production -- cat /proc/1/status | grep Cap
```

---

## Best Practices безопасности

### 1. Принцип наименьших привилегий

```yaml
# Плохо: слишком широкие права
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]

# Хорошо: только необходимые права
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch"]
```

### 2. Не используйте default ServiceAccount

```yaml
# Хорошо: отдельный ServiceAccount для каждого приложения
spec:
  serviceAccountName: my-app
  automountServiceAccountToken: false  # Отключить если не нужен
```

### 3. Всегда устанавливайте SecurityContext

```yaml
# Минимальный безопасный SecurityContext
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
      - ALL
  seccompProfile:
    type: RuntimeDefault
```

### 4. Используйте Network Policies

```yaml
# Начните с deny-all, затем разрешайте явно
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

### 5. Защищайте Secrets

- Используйте External Secrets или Sealed Secrets
- Включите encryption at rest
- Ограничьте доступ через RBAC
- Не храните secrets в Git
- Ротируйте секреты регулярно

### 6. Сканируйте образы

```bash
# Trivy
trivy image my-app:latest

# Grype
grype my-app:latest
```

### 7. Используйте Pod Security Standards

```yaml
# На уровне namespace
metadata:
  labels:
    pod-security.kubernetes.io/enforce: restricted
```

### 8. Минимизируйте образы

```dockerfile
# Используйте distroless или Alpine
FROM gcr.io/distroless/static:nonroot
# или
FROM alpine:latest
RUN adduser -D -u 1000 appuser
USER appuser
```

---

## Типичные уязвимости и как их избежать

### 1. Запуск от root

**Проблема:**
```yaml
# Контейнер запускается от root
spec:
  containers:
    - name: app
      image: nginx:latest  # Запускается от root
```

**Решение:**
```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
```

### 2. Privileged контейнеры

**Проблема:**
```yaml
securityContext:
  privileged: true  # Полный доступ к host
```

**Решение:**
```yaml
securityContext:
  privileged: false
  allowPrivilegeEscalation: false
```

### 3. Host namespaces

**Проблема:**
```yaml
spec:
  hostNetwork: true  # Доступ к сети хоста
  hostPID: true      # Видит все процессы хоста
  hostIPC: true      # Общая память с хостом
```

**Решение:** Не используйте host namespaces без крайней необходимости.

### 4. Записываемая файловая система

**Проблема:**
```yaml
# Атакующий может изменить бинарники
```

**Решение:**
```yaml
securityContext:
  readOnlyRootFilesystem: true
volumeMounts:
  - name: tmp
    mountPath: /tmp
volumes:
  - name: tmp
    emptyDir: {}
```

### 5. Слишком широкие capabilities

**Проблема:**
```yaml
securityContext:
  capabilities:
    add:
      - SYS_ADMIN  # Практически root
```

**Решение:**
```yaml
securityContext:
  capabilities:
    drop:
      - ALL
    add:
      - NET_BIND_SERVICE  # Только если нужно
```

### 6. Отсутствие сетевой изоляции

**Проблема:** Все Pod могут общаться друг с другом.

**Решение:** Применяйте Network Policies с deny-all по умолчанию.

### 7. Secrets в переменных окружения

**Проблема:** Переменные окружения видны в /proc и логах.

**Решение:**
```yaml
# Используйте volumes вместо env
volumeMounts:
  - name: secrets
    mountPath: /etc/secrets
    readOnly: true
```

### 8. Использование latest тега

**Проблема:**
```yaml
image: nginx:latest  # Непредсказуемая версия
```

**Решение:**
```yaml
image: nginx:1.25.3@sha256:abc123...  # Конкретная версия с digest
```

### 9. Отсутствие resource limits

**Проблема:** Pod может потребить все ресурсы ноды (DoS).

**Решение:**
```yaml
resources:
  limits:
    memory: "128Mi"
    cpu: "500m"
  requests:
    memory: "64Mi"
    cpu: "250m"
```

### 10. RBAC: избыточные права

**Проблема:**
```yaml
# cluster-admin для обычного приложения
subjects:
  - kind: ServiceAccount
    name: my-app
roleRef:
  kind: ClusterRole
  name: cluster-admin
```

**Решение:** Создавайте минимальные роли для каждого приложения.

---

## Чек-лист безопасности

### Pod Security

- [ ] `runAsNonRoot: true`
- [ ] `allowPrivilegeEscalation: false`
- [ ] `readOnlyRootFilesystem: true`
- [ ] `capabilities.drop: [ALL]`
- [ ] `seccompProfile.type: RuntimeDefault`
- [ ] Конкретный `runAsUser` и `runAsGroup`
- [ ] Нет `privileged: true`
- [ ] Нет `hostNetwork`, `hostPID`, `hostIPC`

### RBAC

- [ ] Отдельный ServiceAccount для каждого приложения
- [ ] Минимальные необходимые права
- [ ] Нет wildcard (`*`) в rules
- [ ] `automountServiceAccountToken: false` где не нужен

### Network

- [ ] Default deny Network Policy
- [ ] Явные правила для необходимого трафика
- [ ] Изоляция между namespaces

### Secrets

- [ ] Encryption at rest включен
- [ ] External secrets manager для чувствительных данных
- [ ] Secrets не в Git
- [ ] Ограниченный RBAC доступ к secrets

### Images

- [ ] Конкретные версии с digest
- [ ] Регулярное сканирование уязвимостей
- [ ] Minimal base images (distroless/alpine)
- [ ] Образы из доверенных источников

---

## Полезные инструменты

| Инструмент | Назначение |
|------------|------------|
| **Trivy** | Сканирование образов и манифестов |
| **Falco** | Runtime security мониторинг |
| **OPA/Gatekeeper** | Policy enforcement |
| **Kyverno** | Kubernetes-native policies |
| **cert-manager** | Управление TLS сертификатами |
| **External Secrets** | Интеграция с secret managers |
| **Sealed Secrets** | Шифрование secrets для GitOps |
| **kubescape** | Compliance и security scanning |
| **kube-bench** | CIS benchmark проверки |
| **Polaris** | Best practices validation |

---

## Заключение

Безопасность в Kubernetes требует комплексного подхода на всех уровнях:

1. **RBAC** — контролируйте кто и что может делать
2. **ServiceAccounts** — давайте минимальные права приложениям
3. **SecurityContext** — ограничивайте возможности контейнеров
4. **Pod Security Standards** — применяйте политики на уровне namespace
5. **Network Policies** — изолируйте сетевой трафик
6. **Secrets** — защищайте конфиденциальные данные

Применяйте принцип defence in depth и регулярно проводите аудит безопасности кластера.
