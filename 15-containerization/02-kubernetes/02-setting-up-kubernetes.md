# Настройка Kubernetes

## Обзор

Для работы с Kubernetes необходимо развернуть кластер. Существует несколько способов: локальные решения для разработки и обучения, а также управляемые облачные сервисы для production-среды. В этом разделе рассмотрим все основные варианты.

---

## Локальные кластеры

Локальные кластеры идеально подходят для обучения, разработки и тестирования. Они запускаются на вашей машине и не требуют облачной инфраструктуры.

### Minikube

**Minikube** — официальный инструмент от Kubernetes для запуска локального кластера. Создаёт виртуальную машину (или использует контейнеры) с одним узлом.

#### Установка Minikube

**macOS (Homebrew):**
```bash
brew install minikube
```

**Linux:**
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

**Windows (Chocolatey):**
```powershell
choco install minikube
```

#### Запуск кластера

```bash
# Запуск с драйвером Docker (рекомендуется)
minikube start --driver=docker

# Запуск с указанием ресурсов
minikube start --cpus=4 --memory=8192 --driver=docker

# Проверка статуса
minikube status

# Остановка кластера
minikube stop

# Удаление кластера
minikube delete
```

#### Полезные команды Minikube

```bash
# Открыть Dashboard
minikube dashboard

# Получить IP кластера
minikube ip

# SSH в узел
minikube ssh

# Включить addon (например, Ingress)
minikube addons enable ingress

# Список доступных addons
minikube addons list
```

---

### Kind (Kubernetes in Docker)

**Kind** запускает Kubernetes-кластер в Docker-контейнерах. Быстрее Minikube и поддерживает multi-node кластеры.

#### Установка Kind

**macOS (Homebrew):**
```bash
brew install kind
```

**Linux/macOS (Go):**
```bash
go install sigs.k8s.io/kind@latest
```

**Бинарный файл:**
```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

#### Создание кластера

```bash
# Простой кластер с одним узлом
kind create cluster

# Кластер с именем
kind create cluster --name my-cluster

# Удаление кластера
kind delete cluster --name my-cluster
```

#### Multi-node кластер

Создайте файл `kind-config.yaml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

```bash
kind create cluster --config kind-config.yaml
```

---

### k3s

**k3s** — легковесный дистрибутив Kubernetes от Rancher. Идеален для edge-устройств, IoT и маломощных серверов.

#### Установка k3s

```bash
# Установка сервера (master)
curl -sfL https://get.k3s.io | sh -

# Проверка статуса
sudo systemctl status k3s

# Kubeconfig находится в /etc/rancher/k3s/k3s.yaml
sudo cat /etc/rancher/k3s/k3s.yaml
```

#### Добавление worker-узла

```bash
# На master-узле получите токен
sudo cat /var/lib/rancher/k3s/server/node-token

# На worker-узле
curl -sfL https://get.k3s.io | K3S_URL=https://<master-ip>:6443 K3S_TOKEN=<token> sh -
```

#### k3d (k3s в Docker)

**k3d** — обёртка для запуска k3s в Docker-контейнерах:

```bash
# Установка
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash

# Создание кластера
k3d cluster create mycluster

# Кластер с несколькими узлами
k3d cluster create mycluster --servers 1 --agents 3
```

---

### Сравнение локальных решений

| Характеристика | Minikube | Kind | k3s/k3d |
|----------------|----------|------|---------|
| **Скорость запуска** | Медленная (VM) | Быстрая | Очень быстрая |
| **Multi-node** | Да (v1.10+) | Да | Да |
| **Ресурсы** | Высокие | Средние | Низкие |
| **Addons** | Встроенные | Нет | Частичные |
| **CI/CD** | Сложнее | Отлично | Хорошо |
| **Production-ready** | Нет | Нет | Да (k3s) |
| **Лучше для** | Обучение | Тестирование | Edge/IoT |

---

## Managed провайдеры (Cloud)

Для production-среды рекомендуется использовать управляемые Kubernetes-сервисы. Провайдер берёт на себя управление control plane, обновления и масштабирование.

### Google Kubernetes Engine (GKE)

**GKE** — Kubernetes от Google Cloud. Считается наиболее зрелым решением, так как Google создал Kubernetes.

```bash
# Установка gcloud CLI
# https://cloud.google.com/sdk/docs/install

# Аутентификация
gcloud auth login
gcloud config set project <PROJECT_ID>

# Создание кластера
gcloud container clusters create my-cluster \
  --zone us-central1-a \
  --num-nodes 3 \
  --machine-type e2-medium

# Получение credentials для kubectl
gcloud container clusters get-credentials my-cluster --zone us-central1-a

# Удаление кластера
gcloud container clusters delete my-cluster --zone us-central1-a
```

**Особенности GKE:**
- Автоматические обновления
- GKE Autopilot (полностью управляемые узлы)
- Интеграция с Google Cloud сервисами
- Отличный мониторинг и логирование

---

### Amazon Elastic Kubernetes Service (EKS)

**EKS** — Kubernetes от AWS. Глубокая интеграция с AWS-сервисами.

```bash
# Установка eksctl
brew install eksctl  # macOS
# или
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# Создание кластера
eksctl create cluster \
  --name my-cluster \
  --region us-west-2 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 3

# Kubeconfig обновляется автоматически

# Удаление кластера
eksctl delete cluster --name my-cluster --region us-west-2
```

**Особенности EKS:**
- Интеграция с IAM для авторизации
- EKS Fargate (serverless pods)
- AWS App Mesh для service mesh
- Managed node groups

---

### Azure Kubernetes Service (AKS)

**AKS** — Kubernetes от Microsoft Azure. Бесплатный control plane.

```bash
# Установка Azure CLI
brew install azure-cli  # macOS

# Аутентификация
az login

# Создание resource group
az group create --name myResourceGroup --location eastus

# Создание кластера
az aks create \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --node-count 3 \
  --node-vm-size Standard_DS2_v2 \
  --generate-ssh-keys

# Получение credentials
az aks get-credentials --resource-group myResourceGroup --name myAKSCluster

# Удаление кластера
az aks delete --resource-group myResourceGroup --name myAKSCluster
```

**Особенности AKS:**
- Бесплатный control plane
- Azure Active Directory интеграция
- Azure Monitor для мониторинга
- Virtual nodes (ACI интеграция)

---

### Сравнение облачных провайдеров

| Характеристика | GKE | EKS | AKS |
|----------------|-----|-----|-----|
| **Control plane** | Платный | Платный ($0.10/час) | Бесплатный |
| **Зрелость** | Самая высокая | Высокая | Высокая |
| **Автообновления** | Да | Да | Да |
| **Serverless pods** | GKE Autopilot | Fargate | Virtual Nodes |
| **Интеграция** | Google Cloud | AWS | Azure |
| **Лучше для** | Любые workloads | AWS-экосистема | Azure-экосистема |

---

## Установка kubectl

**kubectl** — CLI-инструмент для управления Kubernetes-кластерами. Он необходим для работы с любым кластером.

### Установка

**macOS (Homebrew):**
```bash
brew install kubectl
```

**Linux:**
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

**Windows (Chocolatey):**
```powershell
choco install kubernetes-cli
```

### Проверка установки

```bash
kubectl version --client

# Вывод:
# Client Version: v1.28.0
# Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3
```

### Настройка автодополнения

```bash
# Bash
echo 'source <(kubectl completion bash)' >> ~/.bashrc
source ~/.bashrc

# Zsh
echo 'source <(kubectl completion zsh)' >> ~/.zshrc
source ~/.zshrc

# Алиас k для kubectl
echo 'alias k=kubectl' >> ~/.bashrc
echo 'complete -o default -F __start_kubectl k' >> ~/.bashrc
```

### Kubeconfig

kubectl использует файл конфигурации для подключения к кластерам. По умолчанию: `~/.kube/config`

```bash
# Просмотр текущего контекста
kubectl config current-context

# Список всех контекстов
kubectl config get-contexts

# Переключение контекста
kubectl config use-context <context-name>

# Просмотр конфигурации
kubectl config view
```

#### Пример kubeconfig

```yaml
apiVersion: v1
kind: Config
clusters:
  - name: minikube
    cluster:
      server: https://192.168.49.2:8443
      certificate-authority: /home/user/.minikube/ca.crt
contexts:
  - name: minikube
    context:
      cluster: minikube
      user: minikube
current-context: minikube
users:
  - name: minikube
    user:
      client-certificate: /home/user/.minikube/profiles/minikube/client.crt
      client-key: /home/user/.minikube/profiles/minikube/client.key
```

---

## Первое приложение

Теперь развернём простое приложение в Kubernetes-кластере.

### Шаг 1: Проверка подключения к кластеру

```bash
# Информация о кластере
kubectl cluster-info

# Список узлов
kubectl get nodes

# Ожидаемый вывод:
# NAME       STATUS   ROLES           AGE   VERSION
# minikube   Ready    control-plane   10m   v1.28.0
```

### Шаг 2: Создание Deployment

**Deployment** управляет репликами подов и обеспечивает их обновление.

#### Императивный способ (быстрый)

```bash
# Создание deployment
kubectl create deployment nginx-demo --image=nginx:latest

# Проверка
kubectl get deployments
kubectl get pods
```

#### Декларативный способ (рекомендуется)

Создайте файл `nginx-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-demo
  labels:
    app: nginx
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
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
          resources:
            requests:
              memory: "64Mi"
              cpu: "100m"
            limits:
              memory: "128Mi"
              cpu: "200m"
```

```bash
# Применение манифеста
kubectl apply -f nginx-deployment.yaml

# Проверка статуса
kubectl get deployments
kubectl get pods -o wide

# Подробная информация
kubectl describe deployment nginx-demo
```

### Шаг 3: Создание Service

**Service** предоставляет стабильную точку доступа к подам.

Создайте файл `nginx-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  type: NodePort
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30080
```

```bash
# Применение
kubectl apply -f nginx-service.yaml

# Проверка
kubectl get services

# Вывод:
# NAME            TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
# nginx-service   NodePort   10.96.45.123   <none>        80:30080/TCP   5s
```

### Шаг 4: Доступ к приложению

**Для Minikube:**
```bash
# Получить URL сервиса
minikube service nginx-service --url

# Или открыть в браузере
minikube service nginx-service
```

**Для Kind:**
```bash
# Port forwarding
kubectl port-forward service/nginx-service 8080:80

# Открыть http://localhost:8080
```

**Для облачных провайдеров:**
```bash
# Использовать LoadBalancer вместо NodePort
kubectl patch service nginx-service -p '{"spec": {"type": "LoadBalancer"}}'

# Получить внешний IP
kubectl get service nginx-service
```

### Шаг 5: Масштабирование

```bash
# Увеличить количество реплик
kubectl scale deployment nginx-demo --replicas=5

# Проверка
kubectl get pods

# Вывод: 5 подов nginx-demo-xxx
```

### Шаг 6: Обновление приложения

```bash
# Обновить образ
kubectl set image deployment/nginx-demo nginx=nginx:1.26

# Следить за rollout
kubectl rollout status deployment/nginx-demo

# История обновлений
kubectl rollout history deployment/nginx-demo

# Откат к предыдущей версии
kubectl rollout undo deployment/nginx-demo
```

### Шаг 7: Очистка

```bash
# Удаление ресурсов
kubectl delete -f nginx-deployment.yaml
kubectl delete -f nginx-service.yaml

# Или
kubectl delete deployment nginx-demo
kubectl delete service nginx-service
```

---

## Полезные команды kubectl

```bash
# Просмотр логов пода
kubectl logs <pod-name>
kubectl logs -f <pod-name>  # follow

# Выполнение команды в поде
kubectl exec -it <pod-name> -- /bin/bash

# Просмотр всех ресурсов в namespace
kubectl get all

# Описание ресурса
kubectl describe pod <pod-name>

# Редактирование ресурса
kubectl edit deployment nginx-demo

# Просмотр событий
kubectl get events --sort-by='.lastTimestamp'

# Удаление пода (будет пересоздан deployment'ом)
kubectl delete pod <pod-name>
```

---

## Рекомендации

1. **Для обучения** — используйте Minikube или Kind
2. **Для CI/CD тестирования** — Kind (быстрый запуск в контейнерах)
3. **Для edge/IoT** — k3s
4. **Для production** — используйте managed-решения (GKE, EKS, AKS)
5. **Всегда используйте декларативные манифесты** (YAML-файлы) вместо императивных команд
6. **Храните манифесты в Git** для version control
7. **Настройте автодополнение** для kubectl — значительно ускоряет работу

---

## Дополнительные ресурсы

- [Официальная документация Kubernetes](https://kubernetes.io/docs/)
- [Minikube](https://minikube.sigs.k8s.io/docs/)
- [Kind](https://kind.sigs.k8s.io/)
- [k3s](https://k3s.io/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
