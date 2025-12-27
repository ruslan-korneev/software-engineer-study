# Операции с кластером Kubernetes

## Введение

Операции с кластером Kubernetes охватывают широкий спектр задач по управлению, обслуживанию и поддержке production-ready кластеров. Правильное выполнение этих операций критически важно для обеспечения надёжности, безопасности и производительности инфраструктуры.

---

## 1. Backup и Restore

### Резервное копирование etcd

etcd — это распределённое хранилище ключ-значение, которое содержит всё состояние кластера Kubernetes. Регулярное резервное копирование etcd — обязательная практика.

#### Создание snapshot etcd

```bash
# Получение информации о etcd endpoints
kubectl get pods -n kube-system -l component=etcd

# Создание snapshot с использованием etcdctl
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot-$(date +%Y%m%d-%H%M%S).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Проверка состояния snapshot
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot.db --write-out=table
```

#### Автоматизация backup с помощью CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup
  namespace: kube-system
spec:
  schedule: "0 */6 * * *"  # Каждые 6 часов
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: etcd-backup
            image: registry.k8s.io/etcd:3.5.9-0
            command:
            - /bin/sh
            - -c
            - |
              etcdctl snapshot save /backup/etcd-$(date +%Y%m%d-%H%M%S).db \
                --endpoints=https://etcd-server:2379 \
                --cacert=/etc/kubernetes/pki/etcd/ca.crt \
                --cert=/etc/kubernetes/pki/etcd/server.crt \
                --key=/etc/kubernetes/pki/etcd/server.key
            volumeMounts:
            - name: etcd-certs
              mountPath: /etc/kubernetes/pki/etcd
              readOnly: true
            - name: backup-volume
              mountPath: /backup
          restartPolicy: OnFailure
          hostNetwork: true
          nodeSelector:
            node-role.kubernetes.io/control-plane: ""
          tolerations:
          - effect: NoSchedule
            operator: Exists
          volumes:
          - name: etcd-certs
            hostPath:
              path: /etc/kubernetes/pki/etcd
          - name: backup-volume
            persistentVolumeClaim:
              claimName: etcd-backup-pvc
```

### Резервное копирование ресурсов Kubernetes

#### Использование Velero

Velero — популярный инструмент для backup и restore ресурсов Kubernetes и persistent volumes.

```bash
# Установка Velero CLI
brew install velero  # macOS
# или
wget https://github.com/vmware-tanzu/velero/releases/download/v1.12.0/velero-v1.12.0-linux-amd64.tar.gz

# Установка Velero в кластер (пример для AWS)
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.8.0 \
  --bucket my-velero-bucket \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1 \
  --secret-file ./credentials-velero

# Создание backup
velero backup create my-backup --include-namespaces production

# Создание backup с labels
velero backup create app-backup --selector app=my-app

# Backup всего кластера
velero backup create full-cluster-backup

# Проверка статуса backup
velero backup describe my-backup --details

# Просмотр логов backup
velero backup logs my-backup
```

#### Scheduled Backup с Velero

```yaml
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: daily-backup
  namespace: velero
spec:
  schedule: "0 2 * * *"  # Ежедневно в 2:00
  template:
    includedNamespaces:
    - production
    - staging
    excludedResources:
    - events
    - nodes
    ttl: 720h  # 30 дней
    storageLocation: default
    volumeSnapshotLocations:
    - default
```

### Restore процедуры

#### Восстановление etcd из snapshot

```bash
# Остановка kube-apiserver и etcd
sudo systemctl stop kubelet

# Восстановление из snapshot
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --name etcd-member \
  --initial-cluster etcd-member=https://192.168.1.10:2380 \
  --initial-advertise-peer-urls https://192.168.1.10:2380 \
  --data-dir /var/lib/etcd-restored

# Обновление пути к данным в etcd manifest
sudo sed -i 's|/var/lib/etcd|/var/lib/etcd-restored|g' /etc/kubernetes/manifests/etcd.yaml

# Перезапуск kubelet
sudo systemctl start kubelet
```

#### Восстановление с Velero

```bash
# Просмотр доступных backup
velero backup get

# Восстановление из backup
velero restore create --from-backup my-backup

# Восстановление в другой namespace
velero restore create --from-backup my-backup --namespace-mappings production:production-restored

# Восстановление только определённых ресурсов
velero restore create --from-backup my-backup --include-resources deployments,services

# Проверка статуса restore
velero restore describe my-backup-restore --details
```

---

## 2. Upgrade процедуры

### Стратегии обновления кластера

#### In-place upgrade

Обновление существующих узлов без создания новых.

```bash
# 1. Проверка текущей версии
kubectl version
kubeadm version

# 2. Обновление kubeadm
sudo apt-get update
sudo apt-get install -y --allow-change-held-packages kubeadm=1.29.0-00

# 3. Проверка плана обновления
sudo kubeadm upgrade plan

# 4. Применение обновления на control plane
sudo kubeadm upgrade apply v1.29.0

# 5. Обновление kubelet и kubectl
sudo apt-get install -y --allow-change-held-packages \
  kubelet=1.29.0-00 \
  kubectl=1.29.0-00

# 6. Перезапуск kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

#### Обновление worker nodes

```bash
# 1. Подготовка узла (drain)
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data

# 2. Обновление kubeadm на узле
ssh node-1
sudo apt-get update
sudo apt-get install -y --allow-change-held-packages kubeadm=1.29.0-00

# 3. Обновление конфигурации узла
sudo kubeadm upgrade node

# 4. Обновление kubelet
sudo apt-get install -y --allow-change-held-packages kubelet=1.29.0-00

# 5. Перезапуск kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# 6. Возврат узла в работу
kubectl uncordon node-1
```

### Blue-Green Cluster Upgrade

Создание нового кластера и постепенная миграция workloads.

```bash
# 1. Создание нового кластера с новой версией
# (Используя выбранный инструмент: kubeadm, kops, EKS, GKE, etc.)

# 2. Копирование конфигураций
kubectl get all --all-namespaces -o yaml > cluster-resources.yaml

# 3. Применение ресурсов в новом кластере
kubectl --context new-cluster apply -f cluster-resources.yaml

# 4. Переключение DNS/Load Balancer на новый кластер

# 5. Проверка работоспособности

# 6. Удаление старого кластера
```

### Канареечное обновление (Canary Upgrade)

```bash
# Обновление части worker nodes для тестирования
kubectl label node node-1 kubernetes.io/version-test=canary

# Drain и upgrade одного узла
kubectl drain node-1 --ignore-daemonsets
# ... upgrade процедура ...
kubectl uncordon node-1

# Мониторинг метрик и ошибок

# Продолжение с остальными узлами при успехе
```

---

## 3. Disaster Recovery

### Disaster Recovery Plan (DRP)

#### Компоненты DRP

1. **RTO (Recovery Time Objective)** — максимально допустимое время восстановления
2. **RPO (Recovery Point Objective)** — максимально допустимая потеря данных
3. **Процедуры восстановления**
4. **Тестирование DR**

#### Multi-region setup

```yaml
# Пример конфигурации для мульти-региональной федерации
apiVersion: types.kubefed.io/v1beta1
kind: FederatedDeployment
metadata:
  name: my-app
  namespace: production
spec:
  template:
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: my-app
      template:
        spec:
          containers:
          - name: my-app
            image: my-app:v1.0
  placement:
    clusters:
    - name: cluster-us-east
    - name: cluster-us-west
    - name: cluster-eu-west
  overrides:
  - clusterName: cluster-eu-west
    clusterOverrides:
    - path: "/spec/replicas"
      value: 5
```

### Восстановление Control Plane

```bash
# 1. Восстановление etcd из backup
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-new

# 2. Пересоздание control plane компонентов
# При использовании kubeadm
sudo kubeadm init phase control-plane all --config kubeadm-config.yaml

# 3. Восстановление PKI сертификатов (если утеряны)
# Копирование из backup
sudo cp -r /backup/pki /etc/kubernetes/pki

# 4. Перезапуск всех компонентов
sudo systemctl restart kubelet
```

### Автоматизация DR с помощью GitOps

```yaml
# ArgoCD Application для DR
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: disaster-recovery
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/k8s-configs
    targetRevision: HEAD
    path: clusters/dr
  destination:
    server: https://dr-cluster.example.com
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

---

## 4. Troubleshooting

### Диагностика проблем с Pod

```bash
# Проверка статуса pod
kubectl get pods -o wide

# Детальная информация о pod
kubectl describe pod <pod-name>

# Просмотр логов
kubectl logs <pod-name>
kubectl logs <pod-name> -c <container-name>  # Для multi-container pods
kubectl logs <pod-name> --previous  # Логи предыдущего контейнера

# Логи в реальном времени
kubectl logs -f <pod-name>

# Вход в контейнер
kubectl exec -it <pod-name> -- /bin/bash

# Debug контейнер (ephemeral container)
kubectl debug <pod-name> -it --image=busybox
```

### Диагностика проблем с узлами

```bash
# Статус узлов
kubectl get nodes -o wide

# Детальная информация об узле
kubectl describe node <node-name>

# Проверка условий узла
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.conditions[*].type}{"\n"}{end}'

# Просмотр системных pods на узле
kubectl get pods -n kube-system -o wide --field-selector spec.nodeName=<node-name>

# SSH на узел для диагностики
ssh <node-name>
journalctl -u kubelet -f  # Логи kubelet
systemctl status kubelet  # Статус kubelet
systemctl status containerd  # Статус container runtime
```

### Диагностика сетевых проблем

```bash
# Проверка endpoints сервиса
kubectl get endpoints <service-name>

# Проверка DNS
kubectl run dnsutils --image=tutum/dnsutils --rm -it --restart=Never -- nslookup kubernetes

# Проверка сетевой связности
kubectl run netshoot --image=nicolaka/netshoot --rm -it --restart=Never -- bash
# Внутри контейнера:
# curl http://service-name.namespace.svc.cluster.local
# traceroute <ip>
# tcpdump -i eth0

# Проверка NetworkPolicy
kubectl get networkpolicy -A
kubectl describe networkpolicy <policy-name>

# Проверка CNI
kubectl get pods -n kube-system -l k8s-app=calico-node  # для Calico
kubectl logs -n kube-system -l k8s-app=calico-node
```

### Диагностика проблем с хранилищем

```bash
# Проверка PV и PVC
kubectl get pv,pvc -A

# Детали PVC
kubectl describe pvc <pvc-name>

# Проверка StorageClass
kubectl get storageclass
kubectl describe storageclass <sc-name>

# Проверка CSI drivers
kubectl get csidrivers
kubectl get pods -n kube-system | grep csi
```

### Инструменты диагностики

```bash
# kubectl-debug plugin
kubectl debug node/<node-name> -it --image=ubuntu

# k9s — интерактивный терминальный UI
brew install k9s
k9s

# stern — агрегация логов
stern <pod-pattern> -n <namespace>

# kubectx/kubens — быстрое переключение контекстов
kubectx production
kubens kube-system
```

---

## 5. Capacity Planning

### Мониторинг использования ресурсов

```bash
# Текущее использование ресурсов узлами
kubectl top nodes

# Использование ресурсов pods
kubectl top pods -A

# Детальная информация о ресурсах узла
kubectl describe node <node-name> | grep -A 10 "Allocated resources"
```

### Metrics Server и анализ

```yaml
# Установка Metrics Server
apiVersion: apps/v1
kind: Deployment
metadata:
  name: metrics-server
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: metrics-server
  template:
    spec:
      containers:
      - name: metrics-server
        image: registry.k8s.io/metrics-server/metrics-server:v0.6.4
        args:
        - --cert-dir=/tmp
        - --secure-port=4443
        - --kubelet-preferred-address-types=InternalIP
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
```

### Рекомендации по ресурсам

```bash
# Vertical Pod Autoscaler для рекомендаций
kubectl apply -f https://github.com/kubernetes/autoscaler/releases/download/vpa-release-1.0/vpa-v1.0.0-full.yaml

# Получение рекомендаций VPA
kubectl get vpa
kubectl describe vpa <vpa-name>
```

```yaml
# VPA для анализа
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Off"  # Только рекомендации, без автоматического изменения
  resourcePolicy:
    containerPolicies:
    - containerName: '*'
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 4
        memory: 8Gi
```

### Планирование ёмкости

```yaml
# ResourceQuota для namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: production
spec:
  hard:
    requests.cpu: "100"
    requests.memory: 200Gi
    limits.cpu: "200"
    limits.memory: 400Gi
    pods: "100"
    persistentvolumeclaims: "20"
---
# LimitRange для дефолтных лимитов
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
  - default:
      cpu: 500m
      memory: 512Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    type: Container
```

### Cluster Autoscaler

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  template:
    spec:
      containers:
      - name: cluster-autoscaler
        image: registry.k8s.io/autoscaling/cluster-autoscaler:v1.28.0
        command:
        - ./cluster-autoscaler
        - --v=4
        - --stderrthreshold=info
        - --cloud-provider=aws
        - --skip-nodes-with-local-storage=false
        - --expander=least-waste
        - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/my-cluster
        - --balance-similar-node-groups
        - --scale-down-delay-after-add=10m
        - --scale-down-unneeded-time=10m
```

---

## 6. Node Maintenance

### Drain и Cordon

```bash
# Cordon — запрет размещения новых pods на узле
kubectl cordon <node-name>

# Uncordon — возврат узла в работу
kubectl uncordon <node-name>

# Drain — безопасная эвакуация pods с узла
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Drain с таймаутом
kubectl drain <node-name> --ignore-daemonsets --timeout=300s

# Drain с принудительным удалением (осторожно!)
kubectl drain <node-name> --ignore-daemonsets --force --delete-emptydir-data

# Проверка статуса узла
kubectl get node <node-name> -o jsonpath='{.spec.unschedulable}'
```

### Pod Disruption Budget (PDB)

```yaml
# PDB для обеспечения доступности при maintenance
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-pdb
spec:
  minAvailable: 2  # Минимум 2 пода должны быть доступны
  # или
  # maxUnavailable: 1  # Максимум 1 под может быть недоступен
  selector:
    matchLabels:
      app: my-app
---
# PDB с процентами
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: "50%"
  selector:
    matchLabels:
      app: web
```

### Maintenance Window

```bash
#!/bin/bash
# Скрипт для планового обслуживания узла

NODE_NAME=$1

echo "Starting maintenance for node: $NODE_NAME"

# 1. Cordon узла
kubectl cordon $NODE_NAME

# 2. Ожидание завершения текущих задач
sleep 60

# 3. Drain узла
kubectl drain $NODE_NAME --ignore-daemonsets --delete-emptydir-data --timeout=600s

# 4. Выполнение обслуживания
echo "Perform maintenance tasks..."
# ssh $NODE_NAME "sudo apt-get update && sudo apt-get upgrade -y"
# ssh $NODE_NAME "sudo reboot"

# 5. Ожидание перезагрузки
sleep 120

# 6. Проверка готовности узла
until kubectl get node $NODE_NAME | grep -q "Ready"; do
  echo "Waiting for node to be ready..."
  sleep 10
done

# 7. Возврат узла в работу
kubectl uncordon $NODE_NAME

echo "Maintenance completed for node: $NODE_NAME"
```

### Node Problem Detector

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-problem-detector
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: node-problem-detector
  template:
    metadata:
      labels:
        app: node-problem-detector
    spec:
      containers:
      - name: node-problem-detector
        image: registry.k8s.io/node-problem-detector/node-problem-detector:v0.8.14
        securityContext:
          privileged: true
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        volumeMounts:
        - name: log
          mountPath: /var/log
          readOnly: true
        - name: kmsg
          mountPath: /dev/kmsg
          readOnly: true
      volumes:
      - name: log
        hostPath:
          path: /var/log
      - name: kmsg
        hostPath:
          path: /dev/kmsg
      tolerations:
      - operator: Exists
        effect: NoSchedule
```

---

## 7. Certificate Management

### Управление сертификатами Kubernetes

```bash
# Проверка срока действия сертификатов
kubeadm certs check-expiration

# Обновление всех сертификатов
kubeadm certs renew all

# Обновление конкретного сертификата
kubeadm certs renew apiserver
kubeadm certs renew apiserver-kubelet-client
kubeadm certs renew front-proxy-client
kubeadm certs renew etcd-server

# Проверка сертификатов вручную
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout | grep "Not After"
```

### Автоматическое обновление сертификатов

```yaml
# CronJob для мониторинга сертификатов
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cert-expiry-check
  namespace: kube-system
spec:
  schedule: "0 8 * * *"  # Ежедневно в 8:00
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cert-check
            image: bitnami/kubectl:latest
            command:
            - /bin/bash
            - -c
            - |
              EXPIRY_DAYS=30
              for cert in /etc/kubernetes/pki/*.crt; do
                EXPIRY=$(openssl x509 -enddate -noout -in $cert | cut -d= -f2)
                EXPIRY_EPOCH=$(date -d "$EXPIRY" +%s)
                NOW_EPOCH=$(date +%s)
                DAYS_LEFT=$(( ($EXPIRY_EPOCH - $NOW_EPOCH) / 86400 ))
                if [ $DAYS_LEFT -lt $EXPIRY_DAYS ]; then
                  echo "WARNING: $cert expires in $DAYS_LEFT days"
                  # Отправка уведомления (webhook, email, etc.)
                fi
              done
            volumeMounts:
            - name: certs
              mountPath: /etc/kubernetes/pki
              readOnly: true
          restartPolicy: Never
          volumes:
          - name: certs
            hostPath:
              path: /etc/kubernetes/pki
          nodeSelector:
            node-role.kubernetes.io/control-plane: ""
          tolerations:
          - effect: NoSchedule
            operator: Exists
```

### cert-manager для управления TLS сертификатами

```yaml
# Установка cert-manager
# kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# ClusterIssuer для Let's Encrypt
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
    - http01:
        ingress:
          class: nginx
---
# Certificate
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: my-app-tls
  namespace: production
spec:
  secretName: my-app-tls-secret
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  commonName: app.example.com
  dnsNames:
  - app.example.com
  - www.app.example.com
  duration: 2160h  # 90 дней
  renewBefore: 360h  # Обновлять за 15 дней до истечения
```

---

## 8. etcd операции и обслуживание

### Мониторинг etcd

```bash
# Проверка состояния кластера etcd
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Статус endpoint
ETCDCTL_API=3 etcdctl endpoint status --write-out=table \
  --endpoints=https://10.0.0.1:2379,https://10.0.0.2:2379,https://10.0.0.3:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Список членов кластера
ETCDCTL_API=3 etcdctl member list --write-out=table \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

### Дефрагментация etcd

```bash
# Проверка размера базы данных
ETCDCTL_API=3 etcdctl endpoint status --write-out=table

# Дефрагментация (выполнять на каждом члене кластера поочерёдно)
ETCDCTL_API=3 etcdctl defrag \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Compaction — очистка старых ревизий
ETCDCTL_API=3 etcdctl compact $(ETCDCTL_API=3 etcdctl endpoint status --write-out=json | jq -r '.[0].Status.header.revision')
```

### Управление членами кластера etcd

```bash
# Добавление нового члена
ETCDCTL_API=3 etcdctl member add etcd-new \
  --peer-urls=https://10.0.0.4:2380 \
  --endpoints=https://10.0.0.1:2379

# Удаление члена
ETCDCTL_API=3 etcdctl member remove <member-id>

# Обновление peer URLs
ETCDCTL_API=3 etcdctl member update <member-id> \
  --peer-urls=https://new-ip:2380
```

### Мониторинг метрик etcd

```yaml
# ServiceMonitor для Prometheus
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: etcd
  namespace: monitoring
spec:
  endpoints:
  - interval: 30s
    port: metrics
    scheme: https
    tlsConfig:
      caFile: /etc/prometheus/secrets/etcd-certs/ca.crt
      certFile: /etc/prometheus/secrets/etcd-certs/client.crt
      keyFile: /etc/prometheus/secrets/etcd-certs/client.key
      insecureSkipVerify: false
  namespaceSelector:
    matchNames:
    - kube-system
  selector:
    matchLabels:
      component: etcd
```

### Настройки производительности etcd

```yaml
# Оптимизация etcd для production
apiVersion: v1
kind: Pod
metadata:
  name: etcd
  namespace: kube-system
spec:
  containers:
  - name: etcd
    image: registry.k8s.io/etcd:3.5.9-0
    command:
    - etcd
    - --advertise-client-urls=https://$(POD_IP):2379
    - --initial-advertise-peer-urls=https://$(POD_IP):2380
    - --initial-cluster=master-0=https://10.0.0.1:2380,master-1=https://10.0.0.2:2380,master-2=https://10.0.0.3:2380
    - --initial-cluster-state=new
    - --listen-client-urls=https://0.0.0.0:2379
    - --listen-peer-urls=https://0.0.0.0:2380
    - --name=$(POD_NAME)
    - --data-dir=/var/lib/etcd
    # Оптимизации
    - --auto-compaction-mode=periodic
    - --auto-compaction-retention=8h
    - --quota-backend-bytes=8589934592  # 8GB
    - --snapshot-count=10000
    - --heartbeat-interval=100
    - --election-timeout=1000
```

---

## 9. Cluster API

### Введение в Cluster API

Cluster API (CAPI) — это проект Kubernetes для декларативного управления lifecycle кластеров.

```bash
# Установка clusterctl
curl -L https://github.com/kubernetes-sigs/cluster-api/releases/download/v1.6.0/clusterctl-linux-amd64 -o clusterctl
chmod +x clusterctl
sudo mv clusterctl /usr/local/bin/

# Инициализация management cluster
clusterctl init --infrastructure aws

# Проверка установленных providers
clusterctl config repositories
```

### Создание кластера через Cluster API

```yaml
# Cluster definition
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: my-cluster
  namespace: default
spec:
  clusterNetwork:
    pods:
      cidrBlocks: ["192.168.0.0/16"]
    services:
      cidrBlocks: ["10.96.0.0/12"]
  controlPlaneRef:
    apiVersion: controlplane.cluster.x-k8s.io/v1beta1
    kind: KubeadmControlPlane
    name: my-cluster-control-plane
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
    kind: AWSCluster
    name: my-cluster
---
# AWS Cluster Infrastructure
apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
kind: AWSCluster
metadata:
  name: my-cluster
  namespace: default
spec:
  region: us-east-1
  sshKeyName: my-ssh-key
  network:
    vpc:
      cidrBlock: 10.0.0.0/16
    subnets:
    - availabilityZone: us-east-1a
      cidrBlock: 10.0.1.0/24
      isPublic: true
---
# Control Plane
apiVersion: controlplane.cluster.x-k8s.io/v1beta1
kind: KubeadmControlPlane
metadata:
  name: my-cluster-control-plane
  namespace: default
spec:
  replicas: 3
  version: v1.28.0
  machineTemplate:
    infrastructureRef:
      apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
      kind: AWSMachineTemplate
      name: my-cluster-control-plane
  kubeadmConfigSpec:
    initConfiguration:
      nodeRegistration:
        name: '{{ ds.meta_data.local_hostname }}'
    clusterConfiguration:
      apiServer:
        extraArgs:
          cloud-provider: aws
      controllerManager:
        extraArgs:
          cloud-provider: aws
---
# Worker MachineDeployment
apiVersion: cluster.x-k8s.io/v1beta1
kind: MachineDeployment
metadata:
  name: my-cluster-md-0
  namespace: default
spec:
  clusterName: my-cluster
  replicas: 3
  selector:
    matchLabels: null
  template:
    spec:
      clusterName: my-cluster
      version: v1.28.0
      bootstrap:
        configRef:
          apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
          kind: KubeadmConfigTemplate
          name: my-cluster-md-0
      infrastructureRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
        kind: AWSMachineTemplate
        name: my-cluster-md-0
```

### Управление кластерами

```bash
# Получение kubeconfig для workload cluster
clusterctl get kubeconfig my-cluster > my-cluster.kubeconfig

# Проверка статуса кластера
kubectl get clusters
kubectl describe cluster my-cluster

# Масштабирование control plane
kubectl scale kubeadmcontrolplane my-cluster-control-plane --replicas=5

# Обновление версии Kubernetes
kubectl patch kubeadmcontrolplane my-cluster-control-plane \
  --type merge \
  -p '{"spec":{"version":"v1.29.0"}}'

# Удаление кластера
kubectl delete cluster my-cluster
```

---

## 10. Инструменты для управления кластерами

### kubeadm

```bash
# Инициализация control plane
kubeadm init \
  --control-plane-endpoint "load-balancer:6443" \
  --upload-certs \
  --pod-network-cidr=10.244.0.0/16

# Присоединение worker узла
kubeadm join load-balancer:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>

# Присоединение control plane узла
kubeadm join load-balancer:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane \
  --certificate-key <cert-key>

# Генерация нового токена
kubeadm token create --print-join-command

# Сброс кластера
kubeadm reset
```

### kubeadm config

```yaml
# kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: v1.28.0
controlPlaneEndpoint: "load-balancer:6443"
networking:
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/12"
apiServer:
  certSANs:
  - "load-balancer"
  - "10.0.0.100"
  extraArgs:
    enable-admission-plugins: "NodeRestriction,PodSecurityAdmission"
    audit-log-path: "/var/log/kubernetes/audit.log"
    audit-policy-file: "/etc/kubernetes/audit-policy.yaml"
controllerManager:
  extraArgs:
    bind-address: "0.0.0.0"
scheduler:
  extraArgs:
    bind-address: "0.0.0.0"
etcd:
  local:
    dataDir: /var/lib/etcd
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/control-plane
```

### kops (Kubernetes Operations)

```bash
# Установка kops
brew install kops  # macOS

# Настройка AWS credentials
export AWS_ACCESS_KEY_ID=<access-key>
export AWS_SECRET_ACCESS_KEY=<secret-key>
export KOPS_STATE_STORE=s3://my-kops-state-store

# Создание кластера
kops create cluster \
  --name=my-cluster.k8s.local \
  --state=s3://my-kops-state-store \
  --zones=us-east-1a,us-east-1b,us-east-1c \
  --node-count=3 \
  --node-size=t3.medium \
  --control-plane-size=t3.medium \
  --control-plane-count=3

# Применение конфигурации
kops update cluster --name my-cluster.k8s.local --yes

# Валидация кластера
kops validate cluster --wait 10m

# Редактирование кластера
kops edit cluster my-cluster.k8s.local

# Обновление кластера
kops update cluster --yes
kops rolling-update cluster --yes

# Удаление кластера
kops delete cluster --name my-cluster.k8s.local --yes
```

### Kubespray

```bash
# Клонирование репозитория
git clone https://github.com/kubernetes-sigs/kubespray.git
cd kubespray

# Установка зависимостей
pip install -r requirements.txt

# Копирование inventory
cp -rfp inventory/sample inventory/mycluster

# Редактирование inventory
cat > inventory/mycluster/hosts.yaml <<EOF
all:
  hosts:
    node1:
      ansible_host: 10.0.0.1
      ip: 10.0.0.1
    node2:
      ansible_host: 10.0.0.2
      ip: 10.0.0.2
    node3:
      ansible_host: 10.0.0.3
      ip: 10.0.0.3
  children:
    kube_control_plane:
      hosts:
        node1:
    kube_node:
      hosts:
        node1:
        node2:
        node3:
    etcd:
      hosts:
        node1:
    k8s_cluster:
      children:
        kube_control_plane:
        kube_node:
EOF

# Запуск установки
ansible-playbook -i inventory/mycluster/hosts.yaml \
  --become --become-user=root \
  cluster.yml

# Обновление кластера
ansible-playbook -i inventory/mycluster/hosts.yaml \
  --become --become-user=root \
  upgrade-cluster.yml

# Добавление узлов
ansible-playbook -i inventory/mycluster/hosts.yaml \
  --become --become-user=root \
  scale.yml

# Удаление узлов
ansible-playbook -i inventory/mycluster/hosts.yaml \
  --become --become-user=root \
  remove-node.yml \
  -e "node=node3"
```

### Сравнение инструментов

| Характеристика | kubeadm | kops | Kubespray |
|---------------|---------|------|-----------|
| Сложность | Средняя | Низкая | Высокая |
| Облачные провайдеры | Любые (вручную) | AWS, GCE, Azure | Любые |
| Bare-metal | Да | Нет | Да |
| Автоматизация | Базовая | Полная | Полная |
| HA setup | Вручную | Встроено | Встроено |
| Обновления | kubeadm upgrade | kops rolling-update | upgrade-cluster.yml |
| Гибкость | Высокая | Средняя | Очень высокая |

---

## 11. Cost Optimization

### Мониторинг затрат

```bash
# Установка Kubecost
helm install kubecost kubecost/cost-analyzer \
  --namespace kubecost \
  --create-namespace \
  --set kubecostToken="your-token"

# Доступ к UI
kubectl port-forward -n kubecost svc/kubecost-cost-analyzer 9090:9090
```

### Оптимизация ресурсов

```yaml
# LimitRange для предотвращения избыточных запросов
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-constraints
  namespace: production
spec:
  limits:
  - type: Container
    default:
      cpu: 500m
      memory: 512Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    max:
      cpu: "2"
      memory: 4Gi
    min:
      cpu: 50m
      memory: 64Mi
```

### Spot/Preemptible Instances

```yaml
# Node pool с spot instances (AWS)
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: my-cluster
  region: us-east-1
managedNodeGroups:
- name: spot-workers
  instanceTypes: ["m5.large", "m5a.large", "m4.large"]
  spot: true
  minSize: 2
  maxSize: 10
  desiredCapacity: 3
  labels:
    node-type: spot
  taints:
  - key: spot
    value: "true"
    effect: NoSchedule
---
# Deployment с tolerations для spot nodes
apiVersion: apps/v1
kind: Deployment
metadata:
  name: batch-processor
spec:
  template:
    spec:
      tolerations:
      - key: spot
        operator: Equal
        value: "true"
        effect: NoSchedule
      nodeSelector:
        node-type: spot
      containers:
      - name: processor
        image: batch-processor:v1
```

### Автоматическое масштабирование для экономии

```yaml
# HPA с масштабированием до нуля в нерабочее время
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: dev-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: dev-app
  minReplicas: 0  # Масштабирование до нуля (требует KEDA)
  maxReplicas: 10
  metrics:
  - type: External
    external:
      metric:
        name: workday
        selector:
          matchLabels:
            timezone: "UTC"
      target:
        type: AverageValue
        averageValue: "1"
```

### KEDA для scale-to-zero

```yaml
# Установка KEDA
# helm install keda kedacore/keda --namespace keda --create-namespace

# ScaledObject для масштабирования по расписанию
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: cron-scaledobject
spec:
  scaleTargetRef:
    name: my-deployment
  minReplicaCount: 0
  maxReplicaCount: 10
  triggers:
  - type: cron
    metadata:
      timezone: Europe/Moscow
      start: 0 9 * * 1-5  # Пн-Пт 9:00
      end: 0 18 * * 1-5   # Пн-Пт 18:00
      desiredReplicas: "5"
```

### Рекомендации по экономии

```bash
# 1. Анализ неиспользуемых ресурсов
kubectl get pods --all-namespaces -o json | \
  jq -r '.items[] | select(.status.phase=="Running") |
    "\(.metadata.namespace)/\(.metadata.name): CPU=\(.spec.containers[].resources.requests.cpu // "not set"), MEM=\(.spec.containers[].resources.requests.memory // "not set")"'

# 2. Поиск pods без resource limits
kubectl get pods --all-namespaces -o json | \
  jq -r '.items[] | select(.spec.containers[].resources.limits == null) |
    .metadata.namespace + "/" + .metadata.name'

# 3. Анализ PVC использования
kubectl get pvc --all-namespaces -o json | \
  jq -r '.items[] | "\(.metadata.namespace)/\(.metadata.name): \(.spec.resources.requests.storage)"'

# 4. Удаление неиспользуемых ресурсов
kubectl delete pods --field-selector=status.phase=Succeeded -A
kubectl delete pods --field-selector=status.phase=Failed -A
```

---

## 12. Best Practices для Production кластеров

### Высокая доступность

```yaml
# 1. Минимум 3 control plane узла
# 2. Pod anti-affinity для критических сервисов
apiVersion: apps/v1
kind: Deployment
metadata:
  name: critical-app
spec:
  replicas: 3
  template:
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: critical-app
            topologyKey: kubernetes.io/hostname
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: critical-app
              topologyKey: topology.kubernetes.io/zone
```

### Безопасность

```yaml
# Pod Security Standards
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
---
# Network Policies
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
# RBAC - Principle of Least Privilege
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-reader
  namespace: production
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list", "watch"]
```

### Мониторинг и алертинг

```yaml
# Критические алерты
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: kubernetes-critical
spec:
  groups:
  - name: kubernetes.rules
    rules:
    - alert: KubernetesNodeNotReady
      expr: kube_node_status_condition{condition="Ready",status="true"} == 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Node {{ $labels.node }} is not ready"

    - alert: KubernetesPodCrashLooping
      expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} is crash looping"

    - alert: KubernetesETCDNoLeader
      expr: etcd_server_has_leader == 0
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "etcd cluster has no leader"
```

### Logging и Audit

```yaml
# Audit Policy
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# Не логировать read-only endpoints
- level: None
  users: ["system:kube-proxy"]
  verbs: ["watch"]
  resources:
  - group: ""
    resources: ["endpoints", "services", "services/status"]

# Логировать все изменения secrets на уровне Metadata
- level: Metadata
  resources:
  - group: ""
    resources: ["secrets"]

# Логировать все остальные изменения на уровне RequestResponse
- level: RequestResponse
  verbs: ["create", "update", "patch", "delete"]
  resources:
  - group: ""
  - group: "apps"
  - group: "batch"
```

### Resource Management

```yaml
# Priority Classes
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "Critical production workloads"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 1000
globalDefault: false
preemptionPolicy: Never
description: "Batch jobs and non-critical workloads"
```

### GitOps и Infrastructure as Code

```yaml
# ArgoCD Application
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: production-apps
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/k8s-manifests
    targetRevision: HEAD
    path: production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
    - CreateNamespace=true
    - PrunePropagationPolicy=foreground
    - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### Чек-лист для Production

```markdown
## Control Plane
- [ ] Минимум 3 control plane узла
- [ ] etcd размещён на SSD дисках
- [ ] Backup etcd настроен и тестируется
- [ ] Сертификаты мониторятся на истечение
- [ ] API Server за load balancer

## Worker Nodes
- [ ] Минимум 3 worker узла в разных availability zones
- [ ] Node auto-scaling настроен
- [ ] Node Problem Detector установлен
- [ ] Достаточные resource limits на системные pods

## Networking
- [ ] CNI plugin для production (Calico, Cilium)
- [ ] NetworkPolicy по умолчанию deny
- [ ] Ingress Controller с HA
- [ ] Service Mesh при необходимости

## Security
- [ ] RBAC настроен по принципу least privilege
- [ ] Pod Security Standards применены
- [ ] Secrets зашифрованы at rest
- [ ] Image scanning в CI/CD
- [ ] Audit logging включён

## Monitoring
- [ ] Prometheus + Grafana
- [ ] Alerting настроен
- [ ] Centralized logging (EFK/Loki)
- [ ] Distributed tracing при необходимости

## Disaster Recovery
- [ ] Backup стратегия документирована
- [ ] DR план протестирован
- [ ] RPO и RTO определены

## Operations
- [ ] GitOps для deployments
- [ ] CI/CD pipeline
- [ ] Runbooks для типичных операций
- [ ] On-call rotation
```

---

## Заключение

Операции с кластером Kubernetes требуют системного подхода и глубокого понимания всех компонентов системы. Ключевые принципы:

1. **Автоматизация** — максимально автоматизируйте рутинные операции
2. **Документация** — все процедуры должны быть задокументированы
3. **Тестирование** — регулярно тестируйте DR процедуры и backup
4. **Мониторинг** — проактивный мониторинг позволяет предотвратить проблемы
5. **Security by default** — безопасность должна быть встроена с самого начала

Правильная организация операционных процессов критически важна для надёжной работы production кластеров Kubernetes.
