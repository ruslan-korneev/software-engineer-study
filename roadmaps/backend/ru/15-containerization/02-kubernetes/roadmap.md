# Kubernetes

- [ ] [Introduction](./01-introduction.md)
  - Что такое Kubernetes
  - Архитектура кластера (Control Plane, Nodes)
  - Ключевые концепции
  - Альтернативы Kubernetes

- [ ] [Setting up Kubernetes](./02-setting-up-kubernetes.md)
  - Локальные кластеры (Minikube, Kind, k3s)
  - Managed провайдеры (GKE, EKS, AKS)
  - Установка kubectl
  - Первое приложение

- [ ] [Running Applications](./03-running-applications.md)
  - Pods (жизненный цикл, multi-container)
  - ReplicaSets
  - Deployments (стратегии обновления)
  - StatefulSets
  - DaemonSets
  - Jobs и CronJobs

- [ ] [Services and Networking](./04-services-and-networking.md)
  - Типы сервисов (ClusterIP, NodePort, LoadBalancer)
  - Ingress и Ingress Controllers
  - DNS в Kubernetes
  - Network Policies
  - Service Mesh (Istio, Linkerd)

- [ ] [Configuration Management](./05-configuration-management.md)
  - ConfigMaps
  - Secrets
  - Environment Variables
  - Volumes для конфигураций

- [ ] [Storage and Volumes](./06-storage-and-volumes.md)
  - Volumes и Volume Types
  - PersistentVolumes (PV)
  - PersistentVolumeClaims (PVC)
  - StorageClasses
  - Dynamic Provisioning
  - CSI Drivers

- [ ] [Resource Management](./07-resource-management.md)
  - Requests и Limits
  - ResourceQuotas
  - LimitRanges
  - Quality of Service (QoS)

- [ ] [Security](./08-security.md)
  - RBAC (Roles, ClusterRoles, Bindings)
  - ServiceAccounts
  - Pod Security Standards
  - Network Policies
  - Secrets Management
  - Image Security

- [ ] [Monitoring and Logging](./09-monitoring-and-logging.md)
  - Metrics (Prometheus, Metrics Server)
  - Логирование (Fluentd, Loki)
  - Трейсинг (Jaeger, Zipkin)
  - Grafana dashboards
  - kubectl logs/top

- [ ] [Autoscaling](./10-autoscaling.md)
  - Horizontal Pod Autoscaler (HPA)
  - Vertical Pod Autoscaler (VPA)
  - Cluster Autoscaler
  - KEDA

- [ ] [Scheduling](./11-scheduling.md)
  - Node Selectors
  - Node Affinity/Anti-Affinity
  - Pod Affinity/Anti-Affinity
  - Taints и Tolerations
  - Pod Priority и Preemption

- [ ] [Deployment Patterns](./12-deployment-patterns.md)
  - Helm Charts
  - Kustomize
  - GitOps (ArgoCD, Flux)
  - Blue-Green Deployment
  - Canary Deployment
  - Rolling Updates

- [ ] [Advanced Topics](./13-advanced-topics.md)
  - Custom Resources (CRDs)
  - Operators
  - Admission Controllers
  - API Aggregation

- [ ] [Cluster Operations](./14-cluster-operations.md)
  - Cluster Upgrades
  - Backup и Restore (Velero)
  - Disaster Recovery
  - Multi-cluster Management
