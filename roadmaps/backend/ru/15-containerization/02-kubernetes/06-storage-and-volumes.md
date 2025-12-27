# Storage and Volumes в Kubernetes

## Введение в проблему хранения данных

### Эфемерность контейнеров

Контейнеры по своей природе эфемерны (ephemeral) — они могут быть уничтожены, перезапущены или перемещены на другой узел в любой момент. При этом все данные внутри контейнера теряются. Это создает серьезные проблемы для:

- **Stateful-приложений** — базы данных, очереди сообщений, файловые хранилища
- **Обмена данными** — между контейнерами в одном Pod
- **Персистентности** — сохранения данных между перезапусками

### Зачем нужны Volumes

Volumes в Kubernetes решают несколько задач:

1. **Сохранение данных** — данные переживают перезапуск контейнера
2. **Sharing данных** — несколько контейнеров могут работать с одними данными
3. **Абстракция хранилища** — приложение не знает, где физически хранятся данные
4. **Декларативное управление** — хранилище описывается в манифестах

```
┌─────────────────────────────────────────────────────────────┐
│                           Pod                                │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐      │
│  │ Container 1 │    │ Container 2 │    │ Container 3 │      │
│  │  /app/data  │    │  /shared    │    │   /logs     │      │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘      │
│         │                  │                  │              │
│         └──────────────────┼──────────────────┘              │
│                            │                                 │
│                     ┌──────┴──────┐                         │
│                     │   Volume    │                         │
│                     └──────┬──────┘                         │
└────────────────────────────┼────────────────────────────────┘
                             │
                    ┌────────┴────────┐
                    │  Storage Backend │
                    │  (NFS, AWS EBS,  │
                    │   GCE PD, etc.)  │
                    └─────────────────┘
```

---

## Типы томов в Kubernetes

### emptyDir

**emptyDir** — временный том, который создается при запуске Pod и удаляется при его завершении. Идеален для:

- Кеширования данных
- Обмена файлами между контейнерами
- Временных файлов

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-example
spec:
  containers:
  - name: writer
    image: busybox
    command: ['sh', '-c', 'echo "Hello from writer" > /data/message.txt && sleep 3600']
    volumeMounts:
    - name: shared-data
      mountPath: /data

  - name: reader
    image: busybox
    command: ['sh', '-c', 'sleep 5 && cat /data/message.txt && sleep 3600']
    volumeMounts:
    - name: shared-data
      mountPath: /data

  volumes:
  - name: shared-data
    emptyDir: {}
```

**Варианты emptyDir:**

```yaml
volumes:
# Стандартный emptyDir на диске
- name: cache-volume
  emptyDir: {}

# emptyDir в памяти (RAM) — быстрее, но ограничен памятью
- name: memory-cache
  emptyDir:
    medium: Memory
    sizeLimit: 256Mi
```

**Когда использовать:**
- Кеш для web-сервера
- Checkpoint для сортировки данных
- Shared storage между init-контейнером и основным контейнером

---

### hostPath

**hostPath** монтирует файл или директорию с файловой системы узла в Pod. Это создает прямую связь с конкретным узлом.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-example
spec:
  containers:
  - name: logger
    image: busybox
    command: ['sh', '-c', 'tail -f /var/log/host-logs/syslog']
    volumeMounts:
    - name: host-logs
      mountPath: /var/log/host-logs
      readOnly: true

  volumes:
  - name: host-logs
    hostPath:
      path: /var/log
      type: Directory
```

**Типы hostPath:**

| type | Описание |
|------|----------|
| `""` | Пустая строка — без проверок |
| `DirectoryOrCreate` | Создаст директорию если нет |
| `Directory` | Директория должна существовать |
| `FileOrCreate` | Создаст файл если нет |
| `File` | Файл должен существовать |
| `Socket` | UNIX socket должен существовать |
| `CharDevice` | Character device должен существовать |
| `BlockDevice` | Block device должен существовать |

```yaml
# Пример: доступ к Docker socket
volumes:
- name: docker-socket
  hostPath:
    path: /var/run/docker.sock
    type: Socket

# Пример: создание директории для логов
volumes:
- name: app-logs
  hostPath:
    path: /var/log/myapp
    type: DirectoryOrCreate
```

**Предостережения:**
- Pod привязывается к конкретному узлу
- Риски безопасности — доступ к хост-системе
- Не подходит для production stateful-приложений
- Используется в основном для системных компонентов (мониторинг, логирование)

---

### PersistentVolume (PV)

**PersistentVolume** — это ресурс кластера, представляющий физическое хранилище. PV существует независимо от Pod и имеет собственный жизненный цикл.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nfs-data
  labels:
    type: nfs
spec:
  capacity:
    storage: 10Gi

  accessModes:
    - ReadWriteMany

  persistentVolumeReclaimPolicy: Retain

  storageClassName: manual

  nfs:
    server: 192.168.1.100
    path: /exports/data
```

**Жизненный цикл PV:**

```
Available → Bound → Released → Available/Deleted
    ↑                              │
    └──────────────────────────────┘
         (после reclaim)
```

**Reclaim Policies:**

| Policy | Описание |
|--------|----------|
| `Retain` | PV сохраняется после удаления PVC, требует ручной очистки |
| `Recycle` | Deprecated. Выполняет `rm -rf /thevolume/*` |
| `Delete` | Удаляет PV и соответствующий ресурс в облаке |

**Пример PV для разных бэкендов:**

```yaml
# AWS EBS
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-aws-ebs
spec:
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: aws-gp3
  awsElasticBlockStore:
    volumeID: vol-0123456789abcdef0
    fsType: ext4

---
# Local storage
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-local
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /mnt/disks/ssd1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - node-1
```

---

### PersistentVolumeClaim (PVC)

**PersistentVolumeClaim** — это запрос на хранилище от пользователя. PVC абстрагирует детали хранилища от приложения.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: database-pvc
spec:
  accessModes:
    - ReadWriteOnce

  resources:
    requests:
      storage: 10Gi

  storageClassName: fast-ssd

  selector:
    matchLabels:
      type: nfs
```

**Использование PVC в Pod:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: postgres-pod
spec:
  containers:
  - name: postgres
    image: postgres:15
    env:
    - name: POSTGRES_PASSWORD
      value: secretpassword
    - name: PGDATA
      value: /var/lib/postgresql/data/pgdata
    volumeMounts:
    - name: postgres-storage
      mountPath: /var/lib/postgresql/data
    ports:
    - containerPort: 5432

  volumes:
  - name: postgres-storage
    persistentVolumeClaim:
      claimName: database-pvc
```

**Полный пример с Deployment:**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: standard

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stateful-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: stateful-app
  template:
    metadata:
      labels:
        app: stateful-app
    spec:
      containers:
      - name: app
        image: myapp:latest
        volumeMounts:
        - name: data
          mountPath: /app/data
        - name: uploads
          mountPath: /app/uploads
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: app-data-pvc
      - name: uploads
        emptyDir: {}
```

---

## StorageClass и динамическое Provisioning

### Что такое StorageClass

**StorageClass** определяет "класс" хранилища с определенными характеристиками. Он позволяет динамически создавать PV при запросе PVC.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  fsType: ext4
  encrypted: "true"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

### Provisioners

Каждый StorageClass использует provisioner для создания томов:

```yaml
# AWS EBS
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: aws-ebs-gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
  kmsKeyId: arn:aws:kms:...

---
# GCP Persistent Disk
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gcp-pd-ssd
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
  replication-type: regional-pd

---
# NFS (с external provisioner)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage
provisioner: nfs.csi.k8s.io
parameters:
  server: nfs-server.default.svc.cluster.local
  share: /exports

---
# Local Path Provisioner (для разработки)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-path
provisioner: rancher.io/local-path
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

### Volume Binding Mode

```yaml
# Immediate — PV создается сразу при создании PVC
volumeBindingMode: Immediate

# WaitForFirstConsumer — PV создается только при запуске Pod
# Позволяет учитывать topology (зону, узел)
volumeBindingMode: WaitForFirstConsumer
```

### Динамическое provisioning в действии

```yaml
# 1. StorageClass уже существует
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2

---
# 2. Создаем PVC — PV создастся автоматически
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: standard  # Ссылка на StorageClass
```

После создания PVC Kubernetes автоматически:
1. Находит подходящий StorageClass
2. Вызывает provisioner
3. Создает PV с нужными параметрами
4. Связывает PVC с PV

---

## Access Modes

Access Modes определяют, как том может быть смонтирован:

| Mode | Сокращение | Описание |
|------|------------|----------|
| `ReadWriteOnce` | RWO | Чтение/запись одним узлом |
| `ReadOnlyMany` | ROX | Только чтение многими узлами |
| `ReadWriteMany` | RWX | Чтение/запись многими узлами |
| `ReadWriteOncePod` | RWOP | Чтение/запись одним Pod (K8s 1.22+) |

### Поддержка разными провайдерами

| Тип хранилища | RWO | ROX | RWX |
|---------------|-----|-----|-----|
| AWS EBS | ✅ | ❌ | ❌ |
| GCP PD | ✅ | ✅ | ❌ |
| Azure Disk | ✅ | ❌ | ❌ |
| NFS | ✅ | ✅ | ✅ |
| CephFS | ✅ | ✅ | ✅ |
| Local | ✅ | ❌ | ❌ |

### Примеры использования Access Modes

```yaml
# RWO — база данных (один Pod пишет)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
  storageClassName: fast-ssd

---
# ROX — статические файлы (много Pod читают)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: static-assets
spec:
  accessModes:
    - ReadOnlyMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: nfs-storage

---
# RWX — shared storage для масштабируемого приложения
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-uploads
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 50Gi
  storageClassName: efs-storage
```

---

## ConfigMap и Secret как тома

### ConfigMap как Volume

ConfigMap может быть смонтирован как том для конфигурационных файлов:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    events {
        worker_connections 1024;
    }
    http {
        server {
            listen 80;
            location / {
                root /usr/share/nginx/html;
            }
        }
    }
  mime.types: |
    types {
        text/html html;
        text/css css;
        application/javascript js;
    }

---
apiVersion: v1
kind: Pod
metadata:
  name: nginx-with-config
spec:
  containers:
  - name: nginx
    image: nginx:latest
    volumeMounts:
    - name: config-volume
      mountPath: /etc/nginx/nginx.conf
      subPath: nginx.conf
    - name: config-volume
      mountPath: /etc/nginx/mime.types
      subPath: mime.types
  volumes:
  - name: config-volume
    configMap:
      name: nginx-config
```

**Расширенные опции ConfigMap:**

```yaml
volumes:
- name: config-volume
  configMap:
    name: app-config
    # Выбрать только определенные ключи
    items:
    - key: config.yaml
      path: application.yaml
    - key: logging.properties
      path: log4j.properties
    # Установить права доступа
    defaultMode: 0644
    # Сделать optional
    optional: false
```

### Secret как Volume

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
type: kubernetes.io/tls
data:
  tls.crt: LS0tLS1CRUdJTi... # base64 encoded
  tls.key: LS0tLS1CRUdJTi... # base64 encoded

---
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: tls-certs
      mountPath: /etc/ssl/certs
      readOnly: true
    - name: app-secrets
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: tls-certs
    secret:
      secretName: tls-secret
      defaultMode: 0400
  - name: app-secrets
    secret:
      secretName: app-secrets
      items:
      - key: database-password
        path: db-pass
      - key: api-key
        path: api-key
```

### Автоматическое обновление

ConfigMap и Secret, смонтированные как тома, автоматически обновляются при изменении (с задержкой до минуты). **Исключение**: если используется `subPath`, обновления не происходят.

```yaml
# Автоматически обновляется
volumeMounts:
- name: config
  mountPath: /etc/config

# НЕ обновляется автоматически
volumeMounts:
- name: config
  mountPath: /etc/config/app.conf
  subPath: app.conf
```

### Projected Volumes

Projected volumes позволяют комбинировать несколько источников в один том:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: projected-volume-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: combined
      mountPath: /etc/combined
  volumes:
  - name: combined
    projected:
      sources:
      - configMap:
          name: app-config
          items:
          - key: config.yaml
            path: config.yaml
      - secret:
          name: app-secrets
          items:
          - key: password
            path: secrets/password
      - downwardAPI:
          items:
          - path: labels
            fieldRef:
              fieldPath: metadata.labels
      - serviceAccountToken:
          path: token
          expirationSeconds: 3600
          audience: api
```

---

## Best Practices

### 1. Выбор правильного типа хранилища

```yaml
# Для Production баз данных — используйте managed storage
# с поддержкой snapshots и репликации
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: database-storage
provisioner: ebs.csi.aws.com
parameters:
  type: io2
  iops: "10000"
allowVolumeExpansion: true
reclaimPolicy: Retain  # Важно для данных!

---
# Для кеша — emptyDir в памяти
volumes:
- name: cache
  emptyDir:
    medium: Memory
    sizeLimit: 1Gi

# Для логов — emptyDir или hostPath
volumes:
- name: logs
  emptyDir:
    sizeLimit: 5Gi
```

### 2. Правильные Reclaim Policies

```yaml
# Для production данных — ВСЕГДА Retain
persistentVolumeReclaimPolicy: Retain

# Для dev/test — можно Delete
persistentVolumeReclaimPolicy: Delete
```

### 3. Используйте StatefulSet для stateful-приложений

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  # Каждая реплика получит свой PVC
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 100Gi
```

### 4. Ограничение размера томов

```yaml
# Ограничение emptyDir
volumes:
- name: temp
  emptyDir:
    sizeLimit: 500Mi

# Resource quotas для PVC
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storage-quota
  namespace: production
spec:
  hard:
    requests.storage: 500Gi
    persistentvolumeclaims: 20
    fast-ssd.storageclass.storage.k8s.io/requests.storage: 100Gi
```

### 5. Мониторинг использования

```yaml
# Включите VolumeStats в kubelet
# В kubelet config:
# serverTLSBootstrap: true

# Метрики доступны через:
# kubelet_volume_stats_available_bytes
# kubelet_volume_stats_capacity_bytes
# kubelet_volume_stats_used_bytes
```

### 6. Backup и Disaster Recovery

```yaml
# Используйте VolumeSnapshots для бэкапов
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: postgres-snapshot
spec:
  volumeSnapshotClassName: csi-snapshotter
  source:
    persistentVolumeClaimName: postgres-data

---
# Восстановление из snapshot
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data-restored
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
  dataSource:
    name: postgres-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```

---

## Команды kubectl для управления томами

### Просмотр ресурсов

```bash
# Список PersistentVolumes
kubectl get pv
kubectl get pv -o wide

# Список PersistentVolumeClaims
kubectl get pvc
kubectl get pvc -n my-namespace
kubectl get pvc -A  # все namespaces

# Список StorageClasses
kubectl get sc
kubectl get storageclass

# Детальная информация
kubectl describe pv <pv-name>
kubectl describe pvc <pvc-name>
kubectl describe sc <sc-name>
```

### Вывод в удобных форматах

```bash
# Показать capacity и access modes
kubectl get pv -o custom-columns=\
NAME:.metadata.name,\
CAPACITY:.spec.capacity.storage,\
ACCESS:.spec.accessModes,\
STATUS:.status.phase,\
CLAIM:.spec.claimRef.name

# JSON вывод для скриптов
kubectl get pvc my-pvc -o json | jq '.status.capacity.storage'

# Показать все PVC с их storage class
kubectl get pvc -o custom-columns=\
NAME:.metadata.name,\
STATUS:.status.phase,\
VOLUME:.spec.volumeName,\
CAPACITY:.status.capacity.storage,\
STORAGECLASS:.spec.storageClassName
```

### Управление PVC

```bash
# Создание PVC из файла
kubectl apply -f pvc.yaml

# Удаление PVC
kubectl delete pvc my-pvc

# Изменение размера PVC (если allowVolumeExpansion: true)
kubectl patch pvc my-pvc -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'

# Проверить, к какому PV привязан PVC
kubectl get pvc my-pvc -o jsonpath='{.spec.volumeName}'
```

### Работа с StorageClass

```bash
# Установить default StorageClass
kubectl patch storageclass standard -p \
  '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# Убрать default флаг
kubectl patch storageclass old-default -p \
  '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

### Отладка проблем

```bash
# Проверить events связанные с PVC
kubectl describe pvc my-pvc | grep -A 20 Events

# Посмотреть почему PVC в Pending
kubectl get pvc my-pvc -o yaml | grep -A 10 status

# Проверить, какие PVC использует Pod
kubectl get pod my-pod -o jsonpath='{.spec.volumes[*].persistentVolumeClaim.claimName}'

# Проверить mount points внутри Pod
kubectl exec my-pod -- df -h
kubectl exec my-pod -- mount | grep /data
```

### Работа со Snapshots

```bash
# Получить VolumeSnapshots
kubectl get volumesnapshots
kubectl get vs  # короткое имя

# Получить VolumeSnapshotClasses
kubectl get volumesnapshotclass
kubectl get vsclass

# Описание snapshot
kubectl describe volumesnapshot my-snapshot
```

### Полезные однострочники

```bash
# Найти unbound PV (доступные для привязки)
kubectl get pv | grep Available

# Найти PVC в состоянии Pending
kubectl get pvc -A | grep Pending

# Посмотреть использование storage по namespaces
kubectl get pvc -A -o custom-columns=\
NS:.metadata.namespace,\
NAME:.metadata.name,\
SIZE:.spec.resources.requests.storage \
| sort | awk '{sum[$1]+=$3} END {for (ns in sum) print ns, sum[ns]}'

# Удалить все Released PV
kubectl get pv | grep Released | awk '{print $1}' | xargs kubectl delete pv

# Экспорт PVC для миграции
kubectl get pvc my-pvc -o yaml | grep -v "uid\|resourceVersion\|creationTimestamp" > pvc-backup.yaml
```

---

## Резюме

| Тип | Персистентность | Use Case |
|-----|-----------------|----------|
| `emptyDir` | Нет (живет с Pod) | Кеш, временные файлы, sharing между контейнерами |
| `hostPath` | Да (на узле) | Системные компоненты, доступ к хосту |
| `PV/PVC` | Да (независимо) | Production данные, базы данных |
| `ConfigMap` | Нет | Конфигурационные файлы |
| `Secret` | Нет | Секреты, сертификаты |

**Ключевые моменты:**
1. Используйте динамический provisioning через StorageClass
2. Выбирайте правильный Access Mode для вашего сценария
3. Для production данных используйте `reclaimPolicy: Retain`
4. StatefulSet — лучший выбор для stateful-приложений
5. Регулярно делайте VolumeSnapshots для backup
