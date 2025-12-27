# Управление конфигурацией в Kubernetes

Управление конфигурацией — это критически важная часть работы с Kubernetes. Правильное разделение кода приложения и его конфигурации позволяет использовать один и тот же образ контейнера в разных средах (dev, staging, production) без пересборки.

Kubernetes предоставляет несколько механизмов для управления конфигурацией:
- **ConfigMaps** — для хранения неконфиденциальных данных
- **Secrets** — для хранения конфиденциальных данных
- **Environment Variables** — для передачи параметров в контейнеры
- **Volumes** — для монтирования конфигурационных файлов

---

## ConfigMaps

**ConfigMap** — это объект API, который хранит неконфиденциальные данные в формате ключ-значение. Pods могут использовать ConfigMaps как переменные окружения, аргументы командной строки или как конфигурационные файлы в volume.

### Способы создания ConfigMap

#### 1. Из литеральных значений (командная строка)

```bash
kubectl create configmap app-config \
  --from-literal=DATABASE_HOST=postgres.default.svc.cluster.local \
  --from-literal=DATABASE_PORT=5432 \
  --from-literal=LOG_LEVEL=info
```

#### 2. Из файла

```bash
# Создание из одного файла
kubectl create configmap nginx-config --from-file=nginx.conf

# Создание из нескольких файлов
kubectl create configmap app-config \
  --from-file=config.json \
  --from-file=settings.yaml

# С указанием ключа
kubectl create configmap app-config --from-file=my-key=config.json
```

#### 3. Из директории

```bash
kubectl create configmap app-config --from-file=./config-dir/
```

Все файлы в директории станут ключами ConfigMap.

#### 4. Из YAML-манифеста

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: default
  labels:
    app: myapp
data:
  # Простые ключ-значение
  DATABASE_HOST: "postgres.default.svc.cluster.local"
  DATABASE_PORT: "5432"
  LOG_LEVEL: "info"

  # Многострочные значения (конфигурационные файлы)
  application.properties: |
    server.port=8080
    spring.datasource.url=jdbc:postgresql://postgres:5432/mydb
    spring.datasource.username=app_user
    logging.level.root=INFO

  nginx.conf: |
    server {
        listen 80;
        server_name localhost;

        location / {
            proxy_pass http://backend:8080;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
```

#### 5. Бинарные данные (binaryData)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: binary-config
binaryData:
  # Данные в base64
  logo.png: iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mNk...
data:
  # Обычные текстовые данные
  config.txt: "some text"
```

### Использование ConfigMap в Pod

#### Способ 1: Все ключи как переменные окружения

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myapp:1.0
    envFrom:
    - configMapRef:
        name: app-config
        # optional: true  # Pod запустится даже если ConfigMap не существует
```

#### Способ 2: Отдельные ключи как переменные окружения

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myapp:1.0
    env:
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DATABASE_HOST
          optional: false  # Pod не запустится без этого ключа
    - name: DB_PORT
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DATABASE_PORT
```

#### Способ 3: Монтирование как Volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
      readOnly: true
  volumes:
  - name: config-volume
    configMap:
      name: app-config
      # Опционально: указать права доступа
      defaultMode: 0644
```

При таком монтировании каждый ключ становится файлом в директории `/etc/config/`.

#### Способ 4: Монтирование отдельных ключей

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    volumeMounts:
    - name: nginx-config
      mountPath: /etc/nginx/nginx.conf
      subPath: nginx.conf  # Монтируем только этот файл
      readOnly: true
  volumes:
  - name: nginx-config
    configMap:
      name: app-config
      items:
      - key: nginx.conf
        path: nginx.conf
        mode: 0644  # Права для конкретного файла
```

### Автоматическое обновление ConfigMap

При монтировании ConfigMap как volume, Kubernetes автоматически обновляет файлы при изменении ConfigMap (с задержкой до kubelet sync period, обычно ~1 минута).

**Важно**: При использовании `subPath` автоматическое обновление НЕ работает!

```yaml
# Это НЕ будет обновляться автоматически
volumeMounts:
- name: config
  mountPath: /app/config.yaml
  subPath: config.yaml

# Это БУДЕТ обновляться автоматически
volumeMounts:
- name: config
  mountPath: /app/config
```

### Immutable ConfigMaps

Начиная с Kubernetes 1.21, можно создавать неизменяемые ConfigMaps:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: immutable-config
data:
  key: value
immutable: true
```

Преимущества:
- Защита от случайных изменений
- Улучшение производительности (kubelet не следит за изменениями)
- Для изменения нужно удалить и создать заново

---

## Secrets

**Secret** — объект для хранения конфиденциальных данных: паролей, токенов, ключей SSH, сертификатов TLS. Secrets похожи на ConfigMaps, но предназначены для чувствительной информации.

### Типы Secrets

| Тип | Описание |
|-----|----------|
| `Opaque` | Произвольные данные (по умолчанию) |
| `kubernetes.io/service-account-token` | Токен сервисного аккаунта |
| `kubernetes.io/dockerconfigjson` | Учётные данные Docker registry |
| `kubernetes.io/basic-auth` | Базовая аутентификация |
| `kubernetes.io/ssh-auth` | SSH ключи |
| `kubernetes.io/tls` | TLS сертификаты |
| `bootstrap.kubernetes.io/token` | Bootstrap токены |

### Способы создания Secrets

#### 1. Из литеральных значений

```bash
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password='S3cur3P@ssw0rd!'
```

#### 2. Из файлов

```bash
# Из файлов с ключами
kubectl create secret generic ssh-keys \
  --from-file=id_rsa=/path/to/id_rsa \
  --from-file=id_rsa.pub=/path/to/id_rsa.pub

# Из файла с паролем
echo -n 'mypassword' > ./password.txt
kubectl create secret generic db-password --from-file=password=./password.txt
```

#### 3. TLS Secret

```bash
kubectl create secret tls my-tls-secret \
  --cert=path/to/cert.pem \
  --key=path/to/key.pem
```

#### 4. Docker Registry Secret

```bash
kubectl create secret docker-registry my-registry-secret \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=password \
  --docker-email=user@example.com
```

#### 5. Из YAML-манифеста

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: default
type: Opaque
# stringData - значения в plain text (удобно для написания)
stringData:
  username: admin
  password: S3cur3P@ssw0rd!
  connection-string: "postgresql://admin:S3cur3P@ssw0rd!@postgres:5432/mydb"
---
# Альтернативно: data с base64
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials-base64
type: Opaque
data:
  # echo -n 'admin' | base64
  username: YWRtaW4=
  # echo -n 'S3cur3P@ssw0rd!' | base64
  password: UzNjdXIzUEBzc3cwcmQh
```

**Важно**: `stringData` автоматически конвертируется в base64 при сохранении. При чтении секрета через `kubectl get secret -o yaml` всегда отображается `data` в base64.

#### 6. TLS Secret из манифеста

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
type: kubernetes.io/tls
stringData:
  tls.crt: |
    -----BEGIN CERTIFICATE-----
    MIICpDCCAYwCCQDU+...
    -----END CERTIFICATE-----
  tls.key: |
    -----BEGIN RSA PRIVATE KEY-----
    MIIEowIBAAKCAQEA...
    -----END RSA PRIVATE KEY-----
```

### Использование Secrets в Pod

#### Способ 1: Все ключи как переменные окружения

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myapp:1.0
    envFrom:
    - secretRef:
        name: db-credentials
```

#### Способ 2: Отдельные ключи как переменные окружения

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myapp:1.0
    env:
    - name: DATABASE_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: username
    - name: DATABASE_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
```

#### Способ 3: Монтирование как Volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: secrets-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secrets-volume
    secret:
      secretName: db-credentials
      defaultMode: 0400  # Строгие права доступа
```

#### Способ 4: Монтирование отдельных ключей

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: certs
      mountPath: /etc/ssl/certs/ca.crt
      subPath: ca.crt
      readOnly: true
  volumes:
  - name: certs
    secret:
      secretName: tls-secret
      items:
      - key: tls.crt
        path: ca.crt
        mode: 0444
```

#### Использование ImagePullSecrets

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: registry.example.com/myapp:1.0
  imagePullSecrets:
  - name: my-registry-secret
```

---

## Environment Variables

Переменные окружения — самый простой способ передачи конфигурации в контейнеры.

### Способы определения переменных окружения

#### 1. Статические значения

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myapp:1.0
    env:
    - name: APP_ENV
      value: "production"
    - name: LOG_LEVEL
      value: "info"
    - name: MAX_CONNECTIONS
      value: "100"  # Все значения — строки
```

#### 2. Из ConfigMap

```yaml
env:
- name: DATABASE_HOST
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: DATABASE_HOST
```

#### 3. Из Secret

```yaml
env:
- name: DATABASE_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-credentials
      key: password
```

#### 4. Из полей Pod (Downward API)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  labels:
    app: myapp
    version: v1
spec:
  containers:
  - name: app
    image: myapp:1.0
    env:
    # Информация о Pod
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: POD_NAMESPACE
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace
    - name: POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
    - name: NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName
    - name: SERVICE_ACCOUNT
      valueFrom:
        fieldRef:
          fieldPath: spec.serviceAccountName

    # Лейблы и аннотации
    - name: APP_LABEL
      valueFrom:
        fieldRef:
          fieldPath: metadata.labels['app']

    # Ресурсы контейнера
    - name: MEMORY_LIMIT
      valueFrom:
        resourceFieldRef:
          containerName: app
          resource: limits.memory
    - name: CPU_REQUEST
      valueFrom:
        resourceFieldRef:
          containerName: app
          resource: requests.cpu
```

#### 5. Комбинированный пример

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myapp:1.0
    env:
    # Статическое значение
    - name: APP_VERSION
      value: "1.0.0"

    # Из ConfigMap (все ключи)
    envFrom:
    - configMapRef:
        name: app-config
        prefix: CONFIG_  # Добавляет префикс ко всем ключам

    # Из Secret (все ключи)
    - secretRef:
        name: db-credentials
        prefix: DB_

    # Дополнительные переменные
    env:
    - name: SPECIAL_KEY
      valueFrom:
        configMapKeyRef:
          name: special-config
          key: special.key
```

### Порядок приоритета

Если одна и та же переменная определена несколько раз:
1. Переменные из `env` переопределяют переменные из `envFrom`
2. Последние записи в списке `envFrom` переопределяют предыдущие
3. Переменные из манифеста переопределяют переменные из образа

---

## Volumes для конфигураций

Volumes позволяют монтировать конфигурационные данные как файлы в контейнеры.

### Типы volumes для конфигурации

#### 1. ConfigMap Volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    volumeMounts:
    - name: nginx-config
      mountPath: /etc/nginx/conf.d
      readOnly: true
  volumes:
  - name: nginx-config
    configMap:
      name: nginx-config
      # Выборочное монтирование ключей
      items:
      - key: default.conf
        path: default.conf
      - key: ssl.conf
        path: ssl.conf
      defaultMode: 0644
```

#### 2. Secret Volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: certs
      mountPath: /etc/ssl/private
      readOnly: true
  volumes:
  - name: certs
    secret:
      secretName: tls-certificates
      defaultMode: 0400  # Только чтение владельцем
      items:
      - key: tls.crt
        path: server.crt
      - key: tls.key
        path: server.key
```

#### 3. Projected Volume (комбинированный)

Projected volume позволяет комбинировать несколько источников в одной директории:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: all-configs
      mountPath: /etc/app
      readOnly: true
  volumes:
  - name: all-configs
    projected:
      defaultMode: 0440
      sources:
      # ConfigMap
      - configMap:
          name: app-config
          items:
          - key: config.yaml
            path: config.yaml

      # Secret
      - secret:
          name: app-secrets
          items:
          - key: api-key
            path: secrets/api-key

      # Downward API
      - downwardAPI:
          items:
          - path: labels
            fieldRef:
              fieldPath: metadata.labels
          - path: annotations
            fieldRef:
              fieldPath: metadata.annotations

      # ServiceAccount Token
      - serviceAccountToken:
          path: token
          expirationSeconds: 3600
          audience: api.example.com
```

#### 4. Downward API Volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  labels:
    app: myapp
    version: v1.0.0
  annotations:
    build: "2024-01-15"
spec:
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: podinfo
      mountPath: /etc/podinfo
      readOnly: true
  volumes:
  - name: podinfo
    downwardAPI:
      items:
      - path: "labels"
        fieldRef:
          fieldPath: metadata.labels
      - path: "annotations"
        fieldRef:
          fieldPath: metadata.annotations
      - path: "name"
        fieldRef:
          fieldPath: metadata.name
      - path: "namespace"
        fieldRef:
          fieldPath: metadata.namespace
      - path: "cpu_limit"
        resourceFieldRef:
          containerName: app
          resource: limits.cpu
          divisor: 1m  # В милликорах
      - path: "memory_limit"
        resourceFieldRef:
          containerName: app
          resource: limits.memory
          divisor: 1Mi  # В мегабайтах
```

---

## Best Practices по безопасности

### 1. Никогда не храните секреты в Git

```bash
# .gitignore
*.secret
*-secret.yaml
secrets/

# Используйте шаблоны
# secret-template.yaml (коммитится)
# secret.yaml (НЕ коммитится)
```

### 2. Используйте RBAC для ограничения доступа

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
  namespace: production
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["app-secrets"]  # Только конкретные секреты
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-app-secrets
  namespace: production
subjects:
- kind: ServiceAccount
  name: myapp-sa
  namespace: production
roleRef:
  kind: Role
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

### 3. Включите шифрование etcd

```yaml
# /etc/kubernetes/encryption-config.yaml
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
  - identity: {}  # Fallback для чтения незашифрованных данных
```

### 4. Используйте внешние системы управления секретами

#### HashiCorp Vault с Vault Agent Injector

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "myapp"
    vault.hashicorp.com/agent-inject-secret-db-creds: "database/creds/myapp"
    vault.hashicorp.com/agent-inject-template-db-creds: |
      {{- with secret "database/creds/myapp" -}}
      export DATABASE_USERNAME="{{ .Data.username }}"
      export DATABASE_PASSWORD="{{ .Data.password }}"
      {{- end }}
spec:
  serviceAccountName: myapp
  containers:
  - name: app
    image: myapp:1.0
    command: ["/bin/sh", "-c", "source /vault/secrets/db-creds && ./start.sh"]
```

#### External Secrets Operator

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: db-credentials
    creationPolicy: Owner
  data:
  - secretKey: username
    remoteRef:
      key: database/credentials
      property: username
  - secretKey: password
    remoteRef:
      key: database/credentials
      property: password
```

### 5. Ротация секретов

```yaml
# Используйте разные секреты для разных версий
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials-v2
  labels:
    version: "2"
type: Opaque
stringData:
  username: admin
  password: NewS3cur3P@ssw0rd!
---
# Обновите Deployment для использования нового секрета
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: app
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials-v2  # Новый секрет
              key: password
```

### 6. Аудит доступа к секретам

```yaml
# audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Metadata
  resources:
  - group: ""
    resources: ["secrets"]
  verbs: ["get", "list", "watch"]
```

### 7. Минимальные права для volumes

```yaml
volumes:
- name: secrets
  secret:
    secretName: app-secrets
    defaultMode: 0400  # Только чтение владельцем
    items:
    - key: api-key
      path: api-key
      mode: 0400
```

### 8. Используйте Pod Security Standards

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
---
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  namespace: production
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: myapp:1.0
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
    volumeMounts:
    - name: secrets
      mountPath: /etc/secrets
      readOnly: true
    - name: tmp
      mountPath: /tmp
  volumes:
  - name: secrets
    secret:
      secretName: app-secrets
  - name: tmp
    emptyDir: {}
```

---

## Практический пример: Полное приложение

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
  MAX_CONNECTIONS: "100"
  config.yaml: |
    server:
      port: 8080
      host: 0.0.0.0
    features:
      cache_enabled: true
      rate_limit: 1000
---
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secrets
type: Opaque
stringData:
  DATABASE_URL: "postgresql://user:password@postgres:5432/mydb"
  API_KEY: "sk-xxxxxxxxxxxxxxxxxxxx"
  JWT_SECRET: "super-secret-jwt-key"
---
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      serviceAccountName: myapp-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      containers:
      - name: app
        image: myapp:1.0
        ports:
        - containerPort: 8080

        # Переменные окружения из ConfigMap
        envFrom:
        - configMapRef:
            name: myapp-config

        # Отдельные переменные из Secret
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: DATABASE_URL
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: API_KEY
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name

        volumeMounts:
        # Конфигурационный файл
        - name: config
          mountPath: /app/config
          readOnly: true
        # Секретные файлы
        - name: secrets
          mountPath: /app/secrets
          readOnly: true

        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL

        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"

      volumes:
      - name: config
        configMap:
          name: myapp-config
          items:
          - key: config.yaml
            path: config.yaml
      - name: secrets
        secret:
          secretName: myapp-secrets
          defaultMode: 0400
          items:
          - key: JWT_SECRET
            path: jwt-secret
```

---

## Полезные команды

```bash
# Просмотр ConfigMaps
kubectl get configmaps
kubectl describe configmap app-config
kubectl get configmap app-config -o yaml

# Просмотр Secrets
kubectl get secrets
kubectl describe secret db-credentials
kubectl get secret db-credentials -o yaml

# Декодирование секрета
kubectl get secret db-credentials -o jsonpath='{.data.password}' | base64 -d

# Редактирование
kubectl edit configmap app-config
kubectl edit secret db-credentials

# Создание из файла
kubectl create configmap nginx-conf --from-file=nginx.conf --dry-run=client -o yaml > configmap.yaml

# Обновление ConfigMap/Secret
kubectl apply -f configmap.yaml

# Перезапуск Deployment после обновления конфигурации
kubectl rollout restart deployment myapp

# Просмотр использования ConfigMap/Secret в Pod
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{range .spec.volumes[*]}{.configMap.name}{.secret.secretName}{"\n"}{end}{"\n"}{end}'
```

---

## Резюме

| Механизм | Когда использовать | Безопасность |
|----------|-------------------|--------------|
| ConfigMap | Неконфиденциальные данные, настройки | Низкая (видны в plain text) |
| Secret | Пароли, токены, ключи | Средняя (base64, можно шифровать) |
| Environment Variables | Простые настройки | Могут попасть в логи |
| Volume Mount | Конфигурационные файлы | Зависит от источника |
| External Secrets | Критические секреты | Высокая (внешнее хранилище) |

**Рекомендации:**
1. Используйте ConfigMaps для неконфиденциальных настроек
2. Используйте Secrets для всех паролей и ключей
3. Включите шифрование etcd в production
4. Рассмотрите внешние системы (Vault, AWS Secrets Manager) для критических секретов
5. Применяйте RBAC для ограничения доступа
6. Регулярно ротируйте секреты
7. Используйте Immutable ConfigMaps/Secrets где возможно
