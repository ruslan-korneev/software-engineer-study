# Продвинутые темы Kubernetes

Kubernetes предоставляет мощные механизмы расширения, позволяющие адаптировать платформу под специфические нужды организации. В этом разделе рассмотрим продвинутые концепции, которые используются для создания собственных расширений, операторов и интеграций.

---

## 1. Custom Resource Definitions (CRDs)

### Что такое CRD?

Custom Resource Definition (CRD) — это механизм расширения Kubernetes API, позволяющий создавать собственные типы ресурсов. После создания CRD можно использовать kubectl и другие инструменты для работы с кастомными ресурсами так же, как с встроенными (Pod, Service и т.д.).

### Зачем нужны CRDs?

- **Расширение функциональности** — добавление новых абстракций
- **Декларативное управление** — использование привычного подхода Kubernetes
- **Интеграция с экосистемой** — RBAC, kubectl, API versioning
- **Основа для операторов** — большинство операторов используют CRDs

### Базовый пример CRD

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.mycompany.io
spec:
  group: mycompany.io
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required:
                - engine
                - version
              properties:
                engine:
                  type: string
                  enum: ["postgresql", "mysql", "mongodb"]
                version:
                  type: string
                replicas:
                  type: integer
                  minimum: 1
                  maximum: 10
                  default: 1
                storage:
                  type: object
                  properties:
                    size:
                      type: string
                      pattern: '^[0-9]+Gi$'
                    storageClass:
                      type: string
            status:
              type: object
              properties:
                phase:
                  type: string
                ready:
                  type: boolean
                replicas:
                  type: integer
      subresources:
        status: {}
        scale:
          specReplicasPath: .spec.replicas
          statusReplicasPath: .status.replicas
      additionalPrinterColumns:
        - name: Engine
          type: string
          jsonPath: .spec.engine
        - name: Version
          type: string
          jsonPath: .spec.version
        - name: Replicas
          type: integer
          jsonPath: .spec.replicas
        - name: Ready
          type: boolean
          jsonPath: .status.ready
        - name: Age
          type: date
          jsonPath: .metadata.creationTimestamp
  scope: Namespaced
  names:
    plural: databases
    singular: database
    kind: Database
    shortNames:
      - db
    categories:
      - all
```

### Использование Custom Resource

```yaml
apiVersion: mycompany.io/v1
kind: Database
metadata:
  name: production-db
  namespace: default
spec:
  engine: postgresql
  version: "15.2"
  replicas: 3
  storage:
    size: 100Gi
    storageClass: fast-ssd
```

### Работа с CRD через kubectl

```bash
# Создание CRD
kubectl apply -f database-crd.yaml

# Проверка регистрации
kubectl get crd databases.mycompany.io

# Создание ресурса
kubectl apply -f production-db.yaml

# Получение списка
kubectl get databases
kubectl get db  # используя shortName

# Детальная информация
kubectl describe database production-db

# Редактирование
kubectl edit database production-db

# Удаление
kubectl delete database production-db
```

### Версионирование CRD

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.mycompany.io
spec:
  group: mycompany.io
  versions:
    - name: v1
      served: true
      storage: false  # старая версия, не используется для хранения
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                engine:
                  type: string
    - name: v2
      served: true
      storage: true  # новая версия, используется для хранения
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                engine:
                  type: string
                configuration:  # новое поле
                  type: object
                  additionalProperties:
                    type: string
  conversion:
    strategy: Webhook
    webhook:
      clientConfig:
        service:
          name: database-conversion
          namespace: default
          path: /convert
        caBundle: <base64-encoded-ca-cert>
      conversionReviewVersions: ["v1", "v1beta1"]
  # ...остальная конфигурация
```

---

## 2. Operators

### Что такое Operator?

Operator — это паттерн расширения Kubernetes, который использует Custom Resources для управления приложениями и их компонентами. Оператор кодифицирует операционные знания (operational knowledge) о приложении в программном обеспечении.

### Паттерн Operator

```
┌─────────────────────────────────────────────────────────────┐
│                      Control Loop                            │
│                                                              │
│   ┌──────────┐     ┌──────────────┐     ┌──────────────┐   │
│   │  Watch   │────▶│   Compare    │────▶│  Reconcile   │   │
│   │ Resources│     │ Desired vs   │     │   (Act)      │   │
│   │          │◀────│   Actual     │◀────│              │   │
│   └──────────┘     └──────────────┘     └──────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Основные компоненты оператора

1. **Custom Resource Definition (CRD)** — определяет желаемое состояние
2. **Controller** — реализует логику управления
3. **Reconciliation Loop** — бесконечный цикл синхронизации состояния

### Пример оператора на Go (используя controller-runtime)

```go
package controllers

import (
    "context"
    "fmt"

    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    "k8s.io/apimachinery/pkg/api/errors"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/apimachinery/pkg/types"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/log"

    mycompanyv1 "github.com/mycompany/database-operator/api/v1"
)

// DatabaseReconciler reconciles a Database object
type DatabaseReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

// +kubebuilder:rbac:groups=mycompany.io,resources=databases,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=mycompany.io,resources=databases/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=apps,resources=statefulsets,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=services,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=configmaps,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=secrets,verbs=get;list;watch;create;update;patch;delete

func (r *DatabaseReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    logger := log.FromContext(ctx)

    // Получаем Database ресурс
    database := &mycompanyv1.Database{}
    err := r.Get(ctx, req.NamespacedName, database)
    if err != nil {
        if errors.IsNotFound(err) {
            // Ресурс удален, ничего не делаем
            logger.Info("Database resource not found. Ignoring since object must be deleted")
            return ctrl.Result{}, nil
        }
        logger.Error(err, "Failed to get Database")
        return ctrl.Result{}, err
    }

    // Проверяем, нужно ли удаление (finalizers)
    if database.ObjectMeta.DeletionTimestamp.IsZero() {
        // Добавляем finalizer если его нет
        if !containsString(database.ObjectMeta.Finalizers, databaseFinalizer) {
            database.ObjectMeta.Finalizers = append(database.ObjectMeta.Finalizers, databaseFinalizer)
            if err := r.Update(ctx, database); err != nil {
                return ctrl.Result{}, err
            }
        }
    } else {
        // Объект помечен для удаления
        if containsString(database.ObjectMeta.Finalizers, databaseFinalizer) {
            // Выполняем cleanup
            if err := r.cleanupDatabase(ctx, database); err != nil {
                return ctrl.Result{}, err
            }

            // Удаляем finalizer
            database.ObjectMeta.Finalizers = removeString(database.ObjectMeta.Finalizers, databaseFinalizer)
            if err := r.Update(ctx, database); err != nil {
                return ctrl.Result{}, err
            }
        }
        return ctrl.Result{}, nil
    }

    // Создаем или обновляем ConfigMap
    configMap, err := r.reconcileConfigMap(ctx, database)
    if err != nil {
        logger.Error(err, "Failed to reconcile ConfigMap")
        return ctrl.Result{}, err
    }

    // Создаем или обновляем Secret
    secret, err := r.reconcileSecret(ctx, database)
    if err != nil {
        logger.Error(err, "Failed to reconcile Secret")
        return ctrl.Result{}, err
    }

    // Создаем или обновляем Service
    service, err := r.reconcileService(ctx, database)
    if err != nil {
        logger.Error(err, "Failed to reconcile Service")
        return ctrl.Result{}, err
    }

    // Создаем или обновляем StatefulSet
    statefulSet, err := r.reconcileStatefulSet(ctx, database, configMap, secret)
    if err != nil {
        logger.Error(err, "Failed to reconcile StatefulSet")
        return ctrl.Result{}, err
    }

    // Обновляем статус
    database.Status.Phase = "Running"
    database.Status.Ready = statefulSet.Status.ReadyReplicas == *statefulSet.Spec.Replicas
    database.Status.Replicas = statefulSet.Status.ReadyReplicas

    if err := r.Status().Update(ctx, database); err != nil {
        logger.Error(err, "Failed to update Database status")
        return ctrl.Result{}, err
    }

    logger.Info("Successfully reconciled Database",
        "name", database.Name,
        "ready", database.Status.Ready)

    return ctrl.Result{RequeueAfter: time.Minute * 5}, nil
}

func (r *DatabaseReconciler) reconcileStatefulSet(
    ctx context.Context,
    database *mycompanyv1.Database,
    configMap *corev1.ConfigMap,
    secret *corev1.Secret,
) (*appsv1.StatefulSet, error) {

    // Определяем желаемый StatefulSet
    desired := &appsv1.StatefulSet{
        ObjectMeta: metav1.ObjectMeta{
            Name:      database.Name,
            Namespace: database.Namespace,
        },
        Spec: appsv1.StatefulSetSpec{
            ServiceName: database.Name,
            Replicas:    &database.Spec.Replicas,
            Selector: &metav1.LabelSelector{
                MatchLabels: map[string]string{
                    "app":      "database",
                    "database": database.Name,
                },
            },
            Template: corev1.PodTemplateSpec{
                ObjectMeta: metav1.ObjectMeta{
                    Labels: map[string]string{
                        "app":      "database",
                        "database": database.Name,
                    },
                },
                Spec: corev1.PodSpec{
                    Containers: []corev1.Container{
                        {
                            Name:  "database",
                            Image: fmt.Sprintf("%s:%s", database.Spec.Engine, database.Spec.Version),
                            Ports: []corev1.ContainerPort{
                                {
                                    Name:          "db",
                                    ContainerPort: 5432,
                                },
                            },
                            EnvFrom: []corev1.EnvFromSource{
                                {
                                    ConfigMapRef: &corev1.ConfigMapEnvSource{
                                        LocalObjectReference: corev1.LocalObjectReference{
                                            Name: configMap.Name,
                                        },
                                    },
                                },
                                {
                                    SecretRef: &corev1.SecretEnvSource{
                                        LocalObjectReference: corev1.LocalObjectReference{
                                            Name: secret.Name,
                                        },
                                    },
                                },
                            },
                            VolumeMounts: []corev1.VolumeMount{
                                {
                                    Name:      "data",
                                    MountPath: "/var/lib/postgresql/data",
                                },
                            },
                        },
                    },
                },
            },
            VolumeClaimTemplates: []corev1.PersistentVolumeClaim{
                {
                    ObjectMeta: metav1.ObjectMeta{
                        Name: "data",
                    },
                    Spec: corev1.PersistentVolumeClaimSpec{
                        AccessModes: []corev1.PersistentVolumeAccessMode{
                            corev1.ReadWriteOnce,
                        },
                        StorageClassName: &database.Spec.Storage.StorageClass,
                        Resources: corev1.ResourceRequirements{
                            Requests: corev1.ResourceList{
                                corev1.ResourceStorage: resource.MustParse(database.Spec.Storage.Size),
                            },
                        },
                    },
                },
            },
        },
    }

    // Устанавливаем owner reference
    if err := ctrl.SetControllerReference(database, desired, r.Scheme); err != nil {
        return nil, err
    }

    // Проверяем существующий StatefulSet
    existing := &appsv1.StatefulSet{}
    err := r.Get(ctx, types.NamespacedName{
        Name:      database.Name,
        Namespace: database.Namespace,
    }, existing)

    if err != nil {
        if errors.IsNotFound(err) {
            // Создаем новый
            if err := r.Create(ctx, desired); err != nil {
                return nil, err
            }
            return desired, nil
        }
        return nil, err
    }

    // Обновляем существующий
    existing.Spec.Replicas = desired.Spec.Replicas
    existing.Spec.Template = desired.Spec.Template
    if err := r.Update(ctx, existing); err != nil {
        return nil, err
    }

    return existing, nil
}

// SetupWithManager sets up the controller with the Manager.
func (r *DatabaseReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&mycompanyv1.Database{}).
        Owns(&appsv1.StatefulSet{}).
        Owns(&corev1.Service{}).
        Owns(&corev1.ConfigMap{}).
        Owns(&corev1.Secret{}).
        Complete(r)
}

const databaseFinalizer = "mycompany.io/finalizer"

func containsString(slice []string, s string) bool {
    for _, item := range slice {
        if item == s {
            return true
        }
    }
    return false
}

func removeString(slice []string, s string) []string {
    result := []string{}
    for _, item := range slice {
        if item != s {
            result = append(result, item)
        }
    }
    return result
}
```

### API Definition (types.go)

```go
package v1

import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// DatabaseSpec defines the desired state of Database
type DatabaseSpec struct {
    // Engine is the database engine type
    // +kubebuilder:validation:Enum=postgresql;mysql;mongodb
    Engine string `json:"engine"`

    // Version is the database version
    Version string `json:"version"`

    // Replicas is the number of database replicas
    // +kubebuilder:validation:Minimum=1
    // +kubebuilder:validation:Maximum=10
    // +kubebuilder:default=1
    Replicas int32 `json:"replicas,omitempty"`

    // Storage configuration
    Storage StorageSpec `json:"storage,omitempty"`
}

// StorageSpec defines storage configuration
type StorageSpec struct {
    // Size of storage (e.g., "10Gi")
    Size string `json:"size"`

    // StorageClass name
    StorageClass string `json:"storageClass,omitempty"`
}

// DatabaseStatus defines the observed state of Database
type DatabaseStatus struct {
    // Phase of the database
    Phase string `json:"phase,omitempty"`

    // Ready indicates if all replicas are ready
    Ready bool `json:"ready,omitempty"`

    // Replicas is the current number of ready replicas
    Replicas int32 `json:"replicas,omitempty"`

    // Conditions represent the latest available observations
    Conditions []metav1.Condition `json:"conditions,omitempty"`
}

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:subresource:scale:specpath=.spec.replicas,statuspath=.status.replicas
// +kubebuilder:printcolumn:name="Engine",type="string",JSONPath=".spec.engine"
// +kubebuilder:printcolumn:name="Version",type="string",JSONPath=".spec.version"
// +kubebuilder:printcolumn:name="Replicas",type="integer",JSONPath=".spec.replicas"
// +kubebuilder:printcolumn:name="Ready",type="boolean",JSONPath=".status.ready"
// +kubebuilder:printcolumn:name="Age",type="date",JSONPath=".metadata.creationTimestamp"

// Database is the Schema for the databases API
type Database struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   DatabaseSpec   `json:"spec,omitempty"`
    Status DatabaseStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true

// DatabaseList contains a list of Database
type DatabaseList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`
    Items           []Database `json:"items"`
}

func init() {
    SchemeBuilder.Register(&Database{}, &DatabaseList{})
}
```

### Популярные Operator SDK и фреймворки

| Инструмент | Язык | Особенности |
|-----------|------|-------------|
| **Kubebuilder** | Go | Официальный SDK от Kubernetes SIG |
| **Operator SDK** | Go, Ansible, Helm | Разработан Red Hat, часть OperatorHub |
| **Metacontroller** | Any (webhooks) | Декларативный подход |
| **KUDO** | YAML | Stateful operators without code |
| **Kopf** | Python | Pythonic operator framework |

### Operator Lifecycle Manager (OLM)

OLM управляет жизненным циклом операторов в кластере:

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: ClusterServiceVersion
metadata:
  name: database-operator.v1.0.0
  namespace: operators
spec:
  displayName: Database Operator
  description: Manages databases on Kubernetes
  version: 1.0.0
  replaces: database-operator.v0.9.0
  maturity: stable
  maintainers:
    - name: DevOps Team
      email: devops@mycompany.io
  provider:
    name: MyCompany
  installModes:
    - type: OwnNamespace
      supported: true
    - type: SingleNamespace
      supported: true
    - type: MultiNamespace
      supported: false
    - type: AllNamespaces
      supported: true
  install:
    strategy: deployment
    spec:
      deployments:
        - name: database-operator
          spec:
            replicas: 1
            selector:
              matchLabels:
                app: database-operator
            template:
              metadata:
                labels:
                  app: database-operator
              spec:
                serviceAccountName: database-operator
                containers:
                  - name: operator
                    image: mycompany/database-operator:v1.0.0
                    resources:
                      limits:
                        cpu: 100m
                        memory: 128Mi
      permissions:
        - serviceAccountName: database-operator
          rules:
            - apiGroups: ["mycompany.io"]
              resources: ["databases"]
              verbs: ["*"]
            - apiGroups: ["apps"]
              resources: ["statefulsets"]
              verbs: ["*"]
  customresourcedefinitions:
    owned:
      - name: databases.mycompany.io
        version: v1
        kind: Database
        displayName: Database
        description: A database instance
```

---

## 3. Admission Controllers

### Что такое Admission Controllers?

Admission Controllers — это плагины, которые перехватывают запросы к API серверу после аутентификации и авторизации, но до сохранения объекта в etcd. Они могут модифицировать или отклонять запросы.

### Типы Admission Controllers

```
                          API Request
                               │
                               ▼
                    ┌──────────────────┐
                    │  Authentication  │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │  Authorization   │
                    └────────┬─────────┘
                             │
                             ▼
          ┌──────────────────────────────────────┐
          │         Admission Controllers         │
          │                                       │
          │  ┌─────────────────────────────────┐ │
          │  │     Mutating Admission          │ │
          │  │  (Modify request, add defaults) │ │
          │  └───────────────┬─────────────────┘ │
          │                  │                   │
          │                  ▼                   │
          │  ┌─────────────────────────────────┐ │
          │  │    Validating Admission         │ │
          │  │  (Validate, accept or reject)   │ │
          │  └─────────────────────────────────┘ │
          └──────────────────┬───────────────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │   Persist to     │
                    │      etcd        │
                    └──────────────────┘
```

### Встроенные Admission Controllers

```bash
# Посмотреть включенные admission controllers
kubectl exec -n kube-system <api-server-pod> -- kube-apiserver --help | grep admission-plugins

# Часто используемые:
# - NamespaceLifecycle: предотвращает создание объектов в несуществующих namespace
# - LimitRanger: применяет resource limits
# - ResourceQuota: проверяет квоты
# - PodSecurityPolicy (deprecated): проверяет политики безопасности
# - PodSecurity: замена PSP
# - MutatingAdmissionWebhook: вызывает внешние webhooks для мутации
# - ValidatingAdmissionWebhook: вызывает внешние webhooks для валидации
```

### Включение Admission Controllers

```yaml
# В манифесте kube-apiserver (или kubeadm конфигурации)
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
apiServer:
  extraArgs:
    enable-admission-plugins: >-
      NamespaceLifecycle,
      LimitRanger,
      ServiceAccount,
      DefaultStorageClass,
      ResourceQuota,
      PodSecurity,
      MutatingAdmissionWebhook,
      ValidatingAdmissionWebhook
    disable-admission-plugins: ""
```

---

## 4. Webhooks (Validating и Mutating)

### Mutating Webhook

Mutating webhooks модифицируют входящие запросы. Типичные use cases:
- Добавление sidecar контейнеров (например, Istio)
- Установка default значений
- Инъекция environment variables

#### Конфигурация Mutating Webhook

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: pod-defaults-webhook
webhooks:
  - name: pod-defaults.mycompany.io
    admissionReviewVersions: ["v1", "v1beta1"]
    sideEffects: None
    timeoutSeconds: 10
    reinvocationPolicy: IfNeeded

    clientConfig:
      service:
        name: webhook-service
        namespace: webhook-system
        path: /mutate
        port: 443
      caBundle: <base64-encoded-ca-cert>

    rules:
      - operations: ["CREATE", "UPDATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
        scope: Namespaced

    namespaceSelector:
      matchLabels:
        webhook-enabled: "true"

    objectSelector:
      matchExpressions:
        - key: skip-webhook
          operator: NotIn
          values: ["true"]

    failurePolicy: Fail  # или Ignore
    matchPolicy: Equivalent
```

#### Пример Mutating Webhook Server на Go

```go
package main

import (
    "encoding/json"
    "fmt"
    "io"
    "net/http"

    admissionv1 "k8s.io/api/admission/v1"
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/apimachinery/pkg/runtime/serializer"
)

var (
    scheme = runtime.NewScheme()
    codecs = serializer.NewCodecFactory(scheme)
)

func init() {
    _ = admissionv1.AddToScheme(scheme)
    _ = corev1.AddToScheme(scheme)
}

type patchOperation struct {
    Op    string      `json:"op"`
    Path  string      `json:"path"`
    Value interface{} `json:"value,omitempty"`
}

func handleMutate(w http.ResponseWriter, r *http.Request) {
    // Читаем тело запроса
    body, err := io.ReadAll(r.Body)
    if err != nil {
        http.Error(w, "Failed to read request body", http.StatusBadRequest)
        return
    }

    // Декодируем AdmissionReview
    admissionReview := &admissionv1.AdmissionReview{}
    deserializer := codecs.UniversalDeserializer()
    if _, _, err := deserializer.Decode(body, nil, admissionReview); err != nil {
        http.Error(w, fmt.Sprintf("Failed to decode: %v", err), http.StatusBadRequest)
        return
    }

    // Обрабатываем запрос
    response := mutate(admissionReview.Request)

    // Формируем ответ
    admissionReview.Response = response
    admissionReview.Response.UID = admissionReview.Request.UID

    respBytes, err := json.Marshal(admissionReview)
    if err != nil {
        http.Error(w, fmt.Sprintf("Failed to marshal response: %v", err), http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    w.Write(respBytes)
}

func mutate(request *admissionv1.AdmissionRequest) *admissionv1.AdmissionResponse {
    // Декодируем Pod
    pod := &corev1.Pod{}
    if err := json.Unmarshal(request.Object.Raw, pod); err != nil {
        return &admissionv1.AdmissionResponse{
            Allowed: false,
            Result: &metav1.Status{
                Message: fmt.Sprintf("Failed to unmarshal pod: %v", err),
            },
        }
    }

    // Создаем патчи
    var patches []patchOperation

    // Добавляем labels
    if pod.Labels == nil {
        patches = append(patches, patchOperation{
            Op:    "add",
            Path:  "/metadata/labels",
            Value: map[string]string{},
        })
    }

    // Добавляем label с именем namespace
    patches = append(patches, patchOperation{
        Op:    "add",
        Path:  "/metadata/labels/namespace",
        Value: request.Namespace,
    })

    // Добавляем default resource limits если не заданы
    for i, container := range pod.Spec.Containers {
        if container.Resources.Limits == nil {
            patches = append(patches, patchOperation{
                Op:   "add",
                Path: fmt.Sprintf("/spec/containers/%d/resources/limits", i),
                Value: map[string]string{
                    "cpu":    "500m",
                    "memory": "256Mi",
                },
            })
        }
        if container.Resources.Requests == nil {
            patches = append(patches, patchOperation{
                Op:   "add",
                Path: fmt.Sprintf("/spec/containers/%d/resources/requests", i),
                Value: map[string]string{
                    "cpu":    "100m",
                    "memory": "128Mi",
                },
            })
        }
    }

    // Добавляем sidecar контейнер (пример)
    sidecar := corev1.Container{
        Name:  "log-collector",
        Image: "mycompany/log-collector:latest",
        VolumeMounts: []corev1.VolumeMount{
            {
                Name:      "logs",
                MountPath: "/var/log/app",
            },
        },
    }

    patches = append(patches, patchOperation{
        Op:    "add",
        Path:  "/spec/containers/-",
        Value: sidecar,
    })

    // Сериализуем патчи
    patchBytes, err := json.Marshal(patches)
    if err != nil {
        return &admissionv1.AdmissionResponse{
            Allowed: false,
            Result: &metav1.Status{
                Message: fmt.Sprintf("Failed to marshal patches: %v", err),
            },
        }
    }

    patchType := admissionv1.PatchTypeJSONPatch

    return &admissionv1.AdmissionResponse{
        Allowed:   true,
        Patch:     patchBytes,
        PatchType: &patchType,
    }
}

func main() {
    http.HandleFunc("/mutate", handleMutate)
    http.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
    })

    fmt.Println("Starting webhook server on :8443")
    if err := http.ListenAndServeTLS(":8443", "/certs/tls.crt", "/certs/tls.key", nil); err != nil {
        panic(err)
    }
}
```

### Validating Webhook

Validating webhooks проверяют запросы и могут их отклонять:

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: pod-policy-webhook
webhooks:
  - name: pod-policy.mycompany.io
    admissionReviewVersions: ["v1"]
    sideEffects: None

    clientConfig:
      service:
        name: webhook-service
        namespace: webhook-system
        path: /validate
        port: 443
      caBundle: <base64-encoded-ca-cert>

    rules:
      - operations: ["CREATE", "UPDATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]

    failurePolicy: Fail
```

#### Пример Validating Webhook

```go
func validate(request *admissionv1.AdmissionRequest) *admissionv1.AdmissionResponse {
    pod := &corev1.Pod{}
    if err := json.Unmarshal(request.Object.Raw, pod); err != nil {
        return &admissionv1.AdmissionResponse{
            Allowed: false,
            Result: &metav1.Status{
                Message: fmt.Sprintf("Failed to unmarshal pod: %v", err),
            },
        }
    }

    var errors []string

    // Проверка 1: Запрещаем использование latest tag
    for _, container := range pod.Spec.Containers {
        if strings.HasSuffix(container.Image, ":latest") || !strings.Contains(container.Image, ":") {
            errors = append(errors,
                fmt.Sprintf("Container %s uses latest or untagged image: %s",
                    container.Name, container.Image))
        }
    }

    // Проверка 2: Требуем resource limits
    for _, container := range pod.Spec.Containers {
        if container.Resources.Limits == nil ||
           container.Resources.Limits.Cpu().IsZero() ||
           container.Resources.Limits.Memory().IsZero() {
            errors = append(errors,
                fmt.Sprintf("Container %s must have CPU and memory limits",
                    container.Name))
        }
    }

    // Проверка 3: Запрещаем privileged containers
    if pod.Spec.SecurityContext != nil {
        for _, container := range pod.Spec.Containers {
            if container.SecurityContext != nil &&
               container.SecurityContext.Privileged != nil &&
               *container.SecurityContext.Privileged {
                errors = append(errors,
                    fmt.Sprintf("Container %s is privileged, which is not allowed",
                        container.Name))
            }
        }
    }

    // Проверка 4: Требуем определенные labels
    requiredLabels := []string{"app", "version", "owner"}
    for _, label := range requiredLabels {
        if _, ok := pod.Labels[label]; !ok {
            errors = append(errors,
                fmt.Sprintf("Required label '%s' is missing", label))
        }
    }

    if len(errors) > 0 {
        return &admissionv1.AdmissionResponse{
            Allowed: false,
            Result: &metav1.Status{
                Message: "Pod validation failed:\n" + strings.Join(errors, "\n"),
                Code:    http.StatusForbidden,
            },
        }
    }

    return &admissionv1.AdmissionResponse{
        Allowed: true,
    }
}
```

### ValidatingAdmissionPolicy (Kubernetes 1.26+)

Новый встроенный механизм без webhook server:

```yaml
apiVersion: admissionregistration.k8s.io/v1beta1
kind: ValidatingAdmissionPolicy
metadata:
  name: require-labels
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["pods"]
  validations:
    - expression: "object.metadata.labels.?app.hasValue()"
      message: "All pods must have an 'app' label"
    - expression: "object.spec.containers.all(c, !c.image.endsWith(':latest'))"
      message: "Using 'latest' tag is not allowed"
    - expression: "object.spec.containers.all(c, c.resources.?limits.?memory.hasValue())"
      message: "All containers must have memory limits"
---
apiVersion: admissionregistration.k8s.io/v1beta1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: require-labels-binding
spec:
  policyName: require-labels
  validationActions: ["Deny"]
  matchResources:
    namespaceSelector:
      matchLabels:
        environment: production
```

---

## 5. Service Mesh (Istio, Linkerd)

### Что такое Service Mesh?

Service Mesh — это инфраструктурный слой для управления межсервисным взаимодействием. Он обеспечивает:

- **Traffic Management** — routing, load balancing, retries
- **Security** — mTLS, authorization policies
- **Observability** — metrics, tracing, logging
- **Reliability** — circuit breaking, timeouts, fault injection

### Архитектура Service Mesh

```
┌─────────────────────────────────────────────────────────────────┐
│                         Control Plane                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │   Config    │  │  Service    │  │      Certificate        │  │
│  │   Store     │  │  Discovery  │  │      Authority          │  │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                    Configuration Push
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                          Data Plane                              │
│                                                                  │
│  ┌──────────────────────┐     ┌──────────────────────┐         │
│  │        Pod A         │     │        Pod B         │         │
│  │  ┌────────────────┐  │     │  ┌────────────────┐  │         │
│  │  │   Application  │  │     │  │   Application  │  │         │
│  │  └───────┬────────┘  │     │  └───────┬────────┘  │         │
│  │          │           │     │          │           │         │
│  │  ┌───────▼────────┐  │     │  ┌───────▼────────┐  │         │
│  │  │  Sidecar Proxy │◀─┼─────┼──│  Sidecar Proxy │  │         │
│  │  │   (Envoy)      │──┼─────┼──▶   (Envoy)      │  │         │
│  │  └────────────────┘  │     │  └────────────────┘  │         │
│  └──────────────────────┘     └──────────────────────┘         │
└─────────────────────────────────────────────────────────────────┘
```

### Istio

Istio — наиболее популярный и функциональный service mesh.

#### Установка Istio

```bash
# Скачать istioctl
curl -L https://istio.io/downloadIstio | sh -

# Установить Istio с профилем
istioctl install --set profile=demo

# Включить автоматическую инъекцию sidecar
kubectl label namespace default istio-injection=enabled
```

#### Компоненты Istio

```yaml
# VirtualService - правила маршрутизации
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews-route
spec:
  hosts:
    - reviews
  http:
    - match:
        - headers:
            end-user:
              exact: jason
      route:
        - destination:
            host: reviews
            subset: v2
    - route:
        - destination:
            host: reviews
            subset: v1
          weight: 90
        - destination:
            host: reviews
            subset: v2
          weight: 10
    retries:
      attempts: 3
      perTryTimeout: 2s
    timeout: 10s
---
# DestinationRule - конфигурация нагрузки
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews-destination
spec:
  host: reviews
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: UPGRADE
        http1MaxPendingRequests: 100
    loadBalancer:
      simple: ROUND_ROBIN
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
      trafficPolicy:
        loadBalancer:
          simple: LEAST_CONN
---
# Gateway - входная точка
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "bookinfo.example.com"
    - port:
        number: 443
        name: https
        protocol: HTTPS
      tls:
        mode: SIMPLE
        credentialName: bookinfo-credential
      hosts:
        - "bookinfo.example.com"
---
# AuthorizationPolicy - контроль доступа
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: reviews-policy
  namespace: default
spec:
  selector:
    matchLabels:
      app: reviews
  action: ALLOW
  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/default/sa/productpage"]
      to:
        - operation:
            methods: ["GET"]
            paths: ["/reviews/*"]
---
# PeerAuthentication - mTLS конфигурация
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
---
# ServiceEntry - внешние сервисы
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: external-api
spec:
  hosts:
    - api.external-service.com
  location: MESH_EXTERNAL
  ports:
    - number: 443
      name: https
      protocol: TLS
  resolution: DNS
```

### Linkerd

Linkerd — легковесный и простой в использовании service mesh.

#### Установка Linkerd

```bash
# Установить CLI
curl -sL https://run.linkerd.io/install | sh

# Проверить prerequisites
linkerd check --pre

# Установить control plane
linkerd install | kubectl apply -f -

# Проверить установку
linkerd check

# Добавить mesh к namespace
kubectl get deploy -o yaml | linkerd inject - | kubectl apply -f -
```

#### Конфигурация Linkerd

```yaml
# ServiceProfile - правила маршрутизации
apiVersion: linkerd.io/v1alpha2
kind: ServiceProfile
metadata:
  name: webapp.default.svc.cluster.local
  namespace: default
spec:
  routes:
    - name: GET /api/users/{id}
      condition:
        method: GET
        pathRegex: /api/users/[^/]+
      responseClasses:
        - condition:
            status:
              min: 500
              max: 599
          isFailure: true
  retryBudget:
    retryRatio: 0.2
    minRetriesPerSecond: 10
    ttl: 10s
---
# TrafficSplit - канареечные деплойменты
apiVersion: split.smi-spec.io/v1alpha1
kind: TrafficSplit
metadata:
  name: webapp-split
  namespace: default
spec:
  service: webapp
  backends:
    - service: webapp-stable
      weight: 900m
    - service: webapp-canary
      weight: 100m
---
# Server - политики авторизации
apiVersion: policy.linkerd.io/v1beta1
kind: Server
metadata:
  name: webapp-server
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: webapp
  port: http
  proxyProtocol: HTTP/1
---
# ServerAuthorization
apiVersion: policy.linkerd.io/v1beta1
kind: ServerAuthorization
metadata:
  name: webapp-authz
  namespace: default
spec:
  server:
    name: webapp-server
  client:
    meshTLS:
      serviceAccounts:
        - name: nginx
        - name: frontend
```

### Сравнение Istio и Linkerd

| Характеристика | Istio | Linkerd |
|---------------|-------|---------|
| **Сложность** | Высокая | Низкая |
| **Ресурсы** | Больше (~100MB+ на sidecar) | Меньше (~20MB на sidecar) |
| **Функциональность** | Очень богатая | Базовая, но достаточная |
| **Proxy** | Envoy | linkerd2-proxy (Rust) |
| **mTLS** | Да | Да (по умолчанию) |
| **Traffic management** | Продвинутый | Базовый |
| **Multi-cluster** | Да | Да |
| **Производительность** | Хорошая | Отличная |

---

## 6. Multi-cluster Setups

### Зачем нужны мульти-кластерные конфигурации?

- **Высокая доступность** — отказоустойчивость на уровне регионов
- **Geo-distribution** — размещение близко к пользователям
- **Isolation** — разделение environments (dev/staging/prod)
- **Compliance** — соответствие требованиям data locality
- **Scale** — преодоление лимитов одного кластера

### Паттерны Multi-cluster

```
┌─────────────────────────────────────────────────────────────────┐
│                    Pattern 1: Replicated                        │
│                                                                  │
│  ┌──────────────────┐         ┌──────────────────┐             │
│  │   Cluster A      │         │   Cluster B      │             │
│  │ ┌──────────────┐ │         │ ┌──────────────┐ │             │
│  │ │   App + DB   │ │◄───────▶│ │   App + DB   │ │             │
│  │ └──────────────┘ │ Sync    │ └──────────────┘ │             │
│  └──────────────────┘         └──────────────────┘             │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    Pattern 2: Failover                          │
│                                                                  │
│  ┌──────────────────┐         ┌──────────────────┐             │
│  │   Primary        │         │   Secondary      │             │
│  │ ┌──────────────┐ │  Async  │ ┌──────────────┐ │             │
│  │ │   Active     │ │────────▶│ │   Standby    │ │             │
│  │ └──────────────┘ │  Repl   │ └──────────────┘ │             │
│  └──────────────────┘         └──────────────────┘             │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│               Pattern 3: Split by Function                      │
│                                                                  │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐                │
│  │  Frontend  │  │  Backend   │  │  Database  │                │
│  │  Cluster   │──│  Cluster   │──│  Cluster   │                │
│  └────────────┘  └────────────┘  └────────────┘                │
└─────────────────────────────────────────────────────────────────┘
```

### Submariner - Cross-cluster Networking

```bash
# Установка Submariner
curl -Ls https://get.submariner.io | bash

# Подготовка брокера
subctl deploy-broker --kubeconfig /path/to/broker/kubeconfig

# Подключение кластеров
subctl join --kubeconfig /path/to/cluster-a/kubeconfig broker-info.subm
subctl join --kubeconfig /path/to/cluster-b/kubeconfig broker-info.subm
```

```yaml
# ServiceExport - экспорт сервиса
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: ServiceExport
metadata:
  name: nginx
  namespace: default
---
# ServiceImport - импорт сервиса
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: ServiceImport
metadata:
  name: nginx
  namespace: default
spec:
  type: ClusterSetIP
  ports:
    - port: 80
      protocol: TCP
```

### Cilium Cluster Mesh

```yaml
# ClusterMesh конфигурация
apiVersion: cilium.io/v2
kind: CiliumClusterConfig
metadata:
  name: cluster-mesh-config
spec:
  clusters:
    - name: cluster-1
      address: cluster-1-api.example.com
    - name: cluster-2
      address: cluster-2-api.example.com
---
# Global Service - сервис доступный во всех кластерах
apiVersion: v1
kind: Service
metadata:
  name: global-service
  annotations:
    io.cilium/global-service: "true"
    io.cilium/shared-service: "true"
spec:
  selector:
    app: myapp
  ports:
    - port: 80
```

---

## 7. Kubernetes Federation

### Kubefed (Federation v2)

Kubefed позволяет управлять несколькими кластерами как единым целым.

#### Установка Kubefed

```bash
# Установить Kubefed controller
kubectl apply -f https://github.com/kubernetes-sigs/kubefed/releases/download/v0.9.0/kubefed.yaml

# Добавить кластеры в федерацию
kubefedctl join cluster1 --cluster-context cluster1-context \
  --host-cluster-context host-cluster-context
kubefedctl join cluster2 --cluster-context cluster2-context \
  --host-cluster-context host-cluster-context
```

#### Federated Resources

```yaml
# FederatedDeployment
apiVersion: types.kubefed.io/v1beta1
kind: FederatedDeployment
metadata:
  name: nginx
  namespace: default
spec:
  template:
    metadata:
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
              image: nginx:1.21
              ports:
                - containerPort: 80
  placement:
    clusters:
      - name: cluster1
      - name: cluster2
  overrides:
    - clusterName: cluster1
      clusterOverrides:
        - path: "/spec/replicas"
          value: 5
    - clusterName: cluster2
      clusterOverrides:
        - path: "/spec/replicas"
          value: 2
---
# FederatedService
apiVersion: types.kubefed.io/v1beta1
kind: FederatedService
metadata:
  name: nginx
  namespace: default
spec:
  template:
    spec:
      selector:
        app: nginx
      ports:
        - port: 80
          targetPort: 80
  placement:
    clusters:
      - name: cluster1
      - name: cluster2
---
# FederatedNamespace
apiVersion: types.kubefed.io/v1beta1
kind: FederatedNamespace
metadata:
  name: production
  namespace: production
spec:
  placement:
    clusters:
      - name: cluster1
      - name: cluster2
      - name: cluster3
---
# ReplicaSchedulingPreference
apiVersion: scheduling.kubefed.io/v1alpha1
kind: ReplicaSchedulingPreference
metadata:
  name: nginx-rsp
  namespace: default
spec:
  targetKind: FederatedDeployment
  totalReplicas: 10
  clusters:
    cluster1:
      minReplicas: 2
      maxReplicas: 6
      weight: 2
    cluster2:
      minReplicas: 1
      maxReplicas: 4
      weight: 1
```

---

## 8. API Aggregation Layer

### Что такое API Aggregation?

API Aggregation позволяет расширять Kubernetes API, добавляя кастомные API серверы. В отличие от CRD, aggregated APIs запускаются как отдельные серверы.

### Когда использовать Aggregation vs CRD?

| Критерий | CRD | Aggregation |
|----------|-----|-------------|
| Простота | Проще | Сложнее |
| Производительность | etcd-based | Произвольный backend |
| Валидация | OpenAPI Schema | Полная кастомизация |
| Подресурсы | Ограниченные | Любые |
| Версионирование | Webhook conversion | Полный контроль |

### Создание Aggregated API Server

```yaml
# APIService регистрация
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  name: v1alpha1.metrics.example.com
spec:
  group: metrics.example.com
  version: v1alpha1
  service:
    name: custom-metrics-apiserver
    namespace: custom-metrics
    port: 443
  groupPriorityMinimum: 100
  versionPriority: 100
  caBundle: <base64-encoded-ca-cert>
---
# Service для API server
apiVersion: v1
kind: Service
metadata:
  name: custom-metrics-apiserver
  namespace: custom-metrics
spec:
  ports:
    - port: 443
      targetPort: 8443
  selector:
    app: custom-metrics-apiserver
---
# Deployment API server
apiVersion: apps/v1
kind: Deployment
metadata:
  name: custom-metrics-apiserver
  namespace: custom-metrics
spec:
  replicas: 1
  selector:
    matchLabels:
      app: custom-metrics-apiserver
  template:
    metadata:
      labels:
        app: custom-metrics-apiserver
    spec:
      serviceAccountName: custom-metrics-apiserver
      containers:
        - name: apiserver
          image: mycompany/custom-metrics-apiserver:v1.0.0
          args:
            - --secure-port=8443
            - --tls-cert-file=/certs/tls.crt
            - --tls-private-key-file=/certs/tls.key
            - --etcd-servers=http://etcd:2379
          ports:
            - containerPort: 8443
          volumeMounts:
            - name: certs
              mountPath: /certs
              readOnly: true
      volumes:
        - name: certs
          secret:
            secretName: custom-metrics-apiserver-tls
```

### Пример Aggregated API Server на Go

```go
package main

import (
    "os"

    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/apimachinery/pkg/runtime/schema"
    "k8s.io/apiserver/pkg/registry/rest"
    genericapiserver "k8s.io/apiserver/pkg/server"
    "k8s.io/apiserver/pkg/server/options"

    metricsv1alpha1 "example.com/custom-metrics/pkg/apis/metrics/v1alpha1"
)

func main() {
    // Создаем scheme
    scheme := runtime.NewScheme()
    metricsv1alpha1.AddToScheme(scheme)

    // Настраиваем сервер
    serverOptions := options.NewRecommendedOptions("", nil)

    config := &genericapiserver.RecommendedConfig{
        Config: genericapiserver.Config{
            // ... конфигурация
        },
    }

    if err := serverOptions.ApplyTo(config); err != nil {
        os.Exit(1)
    }

    // Создаем сервер
    server, err := config.Complete().New("custom-metrics-server", genericapiserver.NewEmptyDelegate())
    if err != nil {
        os.Exit(1)
    }

    // Регистрируем API группу
    apiGroupInfo := genericapiserver.NewDefaultAPIGroupInfo(
        "metrics.example.com",
        scheme,
        runtime.NewParameterCodec(scheme),
        runtime.NewCodecFactory(scheme),
    )

    // Добавляем версию и storage
    v1alpha1Storage := map[string]rest.Storage{
        "custommetrics": &MetricsStorage{},
    }
    apiGroupInfo.VersionedResourcesStorageMap["v1alpha1"] = v1alpha1Storage

    if err := server.InstallAPIGroup(&apiGroupInfo); err != nil {
        os.Exit(1)
    }

    // Запускаем сервер
    stopCh := genericapiserver.SetupSignalHandler()
    server.PrepareRun().Run(stopCh)
}

// MetricsStorage реализует REST интерфейс
type MetricsStorage struct{}

func (s *MetricsStorage) New() runtime.Object {
    return &metricsv1alpha1.CustomMetric{}
}

func (s *MetricsStorage) Get(ctx context.Context, name string, options *metav1.GetOptions) (runtime.Object, error) {
    // Реализация получения метрики
    return &metricsv1alpha1.CustomMetric{
        ObjectMeta: metav1.ObjectMeta{
            Name: name,
        },
        Value: 42,
    }, nil
}

// ... другие методы REST интерфейса
```

---

## 9. Custom Controllers

### Архитектура Custom Controller

```go
package main

import (
    "context"
    "time"

    "k8s.io/apimachinery/pkg/util/wait"
    "k8s.io/client-go/informers"
    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/tools/cache"
    "k8s.io/client-go/util/workqueue"
)

type Controller struct {
    clientset     kubernetes.Interface
    informer      cache.SharedIndexInformer
    workqueue     workqueue.RateLimitingInterface
    hasSynced     cache.InformerSynced
}

func NewController(clientset kubernetes.Interface) *Controller {
    // Создаем informer factory
    factory := informers.NewSharedInformerFactory(clientset, time.Hour*12)

    // Получаем informer для нужного ресурса
    podInformer := factory.Core().V1().Pods()

    controller := &Controller{
        clientset: clientset,
        informer:  podInformer.Informer(),
        workqueue: workqueue.NewRateLimitingQueue(
            workqueue.DefaultControllerRateLimiter(),
        ),
        hasSynced: podInformer.Informer().HasSynced,
    }

    // Регистрируем event handlers
    podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: func(obj interface{}) {
            key, err := cache.MetaNamespaceKeyFunc(obj)
            if err == nil {
                controller.workqueue.Add(key)
            }
        },
        UpdateFunc: func(old, new interface{}) {
            key, err := cache.MetaNamespaceKeyFunc(new)
            if err == nil {
                controller.workqueue.Add(key)
            }
        },
        DeleteFunc: func(obj interface{}) {
            key, err := cache.DeletionHandlingMetaNamespaceKeyFunc(obj)
            if err == nil {
                controller.workqueue.Add(key)
            }
        },
    })

    return controller
}

func (c *Controller) Run(workers int, stopCh <-chan struct{}) error {
    defer c.workqueue.ShutDown()

    // Запускаем informer
    go c.informer.Run(stopCh)

    // Ждем синхронизации кэша
    if !cache.WaitForCacheSync(stopCh, c.hasSynced) {
        return fmt.Errorf("failed to wait for caches to sync")
    }

    // Запускаем worker'ов
    for i := 0; i < workers; i++ {
        go wait.Until(c.runWorker, time.Second, stopCh)
    }

    <-stopCh
    return nil
}

func (c *Controller) runWorker() {
    for c.processNextItem() {
    }
}

func (c *Controller) processNextItem() bool {
    key, quit := c.workqueue.Get()
    if quit {
        return false
    }
    defer c.workqueue.Done(key)

    err := c.syncHandler(key.(string))
    if err == nil {
        c.workqueue.Forget(key)
        return true
    }

    // Retry с exponential backoff
    if c.workqueue.NumRequeues(key) < 5 {
        c.workqueue.AddRateLimited(key)
        return true
    }

    c.workqueue.Forget(key)
    return true
}

func (c *Controller) syncHandler(key string) error {
    namespace, name, err := cache.SplitMetaNamespaceKey(key)
    if err != nil {
        return err
    }

    // Получаем объект из кэша
    obj, exists, err := c.informer.GetIndexer().GetByKey(key)
    if err != nil {
        return err
    }

    if !exists {
        // Объект удален
        return c.handleDelete(namespace, name)
    }

    // Обрабатываем объект
    return c.handleAddOrUpdate(obj)
}

func (c *Controller) handleAddOrUpdate(obj interface{}) error {
    pod := obj.(*corev1.Pod)

    // Реализация бизнес-логики
    fmt.Printf("Processing pod: %s/%s\n", pod.Namespace, pod.Name)

    return nil
}

func (c *Controller) handleDelete(namespace, name string) error {
    fmt.Printf("Pod deleted: %s/%s\n", namespace, name)
    return nil
}
```

### Leader Election

```go
package main

import (
    "context"
    "os"
    "time"

    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/tools/leaderelection"
    "k8s.io/client-go/tools/leaderelection/resourcelock"
)

func runWithLeaderElection(clientset kubernetes.Interface, run func(context.Context)) {
    // Получаем identity
    id, err := os.Hostname()
    if err != nil {
        panic(err)
    }

    // Создаем resource lock
    lock := &resourcelock.LeaseLock{
        LeaseMeta: metav1.ObjectMeta{
            Name:      "my-controller-lock",
            Namespace: "default",
        },
        Client: clientset.CoordinationV1(),
        LockConfig: resourcelock.ResourceLockConfig{
            Identity: id,
        },
    }

    ctx := context.Background()

    // Запускаем leader election
    leaderelection.RunOrDie(ctx, leaderelection.LeaderElectionConfig{
        Lock:            lock,
        ReleaseOnCancel: true,
        LeaseDuration:   15 * time.Second,
        RenewDeadline:   10 * time.Second,
        RetryPeriod:     2 * time.Second,
        Callbacks: leaderelection.LeaderCallbacks{
            OnStartedLeading: func(ctx context.Context) {
                // Стали лидером - запускаем контроллер
                run(ctx)
            },
            OnStoppedLeading: func() {
                // Потеряли лидерство
                os.Exit(0)
            },
            OnNewLeader: func(identity string) {
                if identity == id {
                    return
                }
                // Новый лидер - другой инстанс
            },
        },
    })
}
```

---

## 10. Extended Resources

### Что такое Extended Resources?

Extended Resources позволяют объявлять и использовать кастомные ресурсы на узлах, помимо стандартных CPU и Memory.

### Добавление Extended Resource на Node

```bash
# Через kubectl patch
kubectl patch node <node-name> --type='json' -p='[
  {
    "op": "add",
    "path": "/status/capacity/example.com~1gpu",
    "value": "4"
  }
]'

# Через API (для automation)
curl -X PATCH \
  -H "Content-Type: application/json-patch+json" \
  -d '[{"op": "add", "path": "/status/capacity/example.com~1special-hardware", "value": "10"}]' \
  https://<api-server>/api/v1/nodes/<node-name>/status
```

### Использование Extended Resources в Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  containers:
    - name: gpu-container
      image: nvidia/cuda:11.0-base
      resources:
        limits:
          example.com/gpu: "2"
        requests:
          example.com/gpu: "2"
---
# Пример с несколькими extended resources
apiVersion: v1
kind: Pod
metadata:
  name: specialized-pod
spec:
  containers:
    - name: main
      image: myapp:latest
      resources:
        requests:
          cpu: "500m"
          memory: "256Mi"
          example.com/fpga: "1"
          example.com/special-memory: "4"
        limits:
          cpu: "1"
          memory: "512Mi"
          example.com/fpga: "1"
          example.com/special-memory: "4"
```

### Node Extended Resource через DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: extended-resource-advertiser
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: resource-advertiser
  template:
    metadata:
      labels:
        app: resource-advertiser
    spec:
      serviceAccountName: resource-advertiser
      containers:
        - name: advertiser
          image: mycompany/resource-advertiser:v1.0.0
          env:
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
          volumeMounts:
            - name: device-dir
              mountPath: /dev
              readOnly: true
      volumes:
        - name: device-dir
          hostPath:
            path: /dev
```

---

## 11. Device Plugins

### Что такое Device Plugins?

Device Plugins позволяют vendors предоставлять доступ к специализированному оборудованию (GPU, FPGA, network devices) без модификации кода Kubernetes.

### Архитектура Device Plugin

```
┌─────────────────────────────────────────────────────────────────┐
│                          Node                                    │
│                                                                  │
│  ┌──────────────────┐      ┌─────────────────────────────────┐  │
│  │     Kubelet      │◀────▶│        Device Plugin            │  │
│  │                  │ gRPC │                                 │  │
│  │  Device Manager  │      │  - ListAndWatch                 │  │
│  │                  │      │  - Allocate                     │  │
│  └──────────────────┘      │  - GetDevicePluginOptions       │  │
│          │                 │  - PreStartContainer            │  │
│          ▼                 └─────────────────────────────────┘  │
│  ┌──────────────────┐                    │                      │
│  │       Pod        │                    ▼                      │
│  │  ┌────────────┐  │      ┌─────────────────────────────────┐  │
│  │  │ Container  │◀─┼──────│         Hardware                │  │
│  │  │            │  │      │    (GPU, FPGA, etc.)            │  │
│  │  └────────────┘  │      └─────────────────────────────────┘  │
│  └──────────────────┘                                           │
└─────────────────────────────────────────────────────────────────┘
```

### Реализация Device Plugin на Go

```go
package main

import (
    "context"
    "net"
    "os"
    "path"
    "time"

    "google.golang.org/grpc"
    pluginapi "k8s.io/kubelet/pkg/apis/deviceplugin/v1beta1"
)

const (
    resourceName = "example.com/custom-device"
    socketPath   = pluginapi.DevicePluginPath + "custom-device.sock"
)

type CustomDevicePlugin struct {
    devices []*pluginapi.Device
    server  *grpc.Server
    stop    chan struct{}
}

func NewCustomDevicePlugin() *CustomDevicePlugin {
    return &CustomDevicePlugin{
        devices: discoverDevices(),
        stop:    make(chan struct{}),
    }
}

func discoverDevices() []*pluginapi.Device {
    // Обнаружение устройств на хосте
    devices := []*pluginapi.Device{}

    // Пример: находим устройства в /dev
    for i := 0; i < 4; i++ {
        devices = append(devices, &pluginapi.Device{
            ID:     fmt.Sprintf("device-%d", i),
            Health: pluginapi.Healthy,
        })
    }

    return devices
}

// GetDevicePluginOptions возвращает опции плагина
func (p *CustomDevicePlugin) GetDevicePluginOptions(ctx context.Context, empty *pluginapi.Empty) (*pluginapi.DevicePluginOptions, error) {
    return &pluginapi.DevicePluginOptions{
        PreStartRequired:                true,
        GetPreferredAllocationAvailable: true,
    }, nil
}

// ListAndWatch отправляет список устройств и обновления
func (p *CustomDevicePlugin) ListAndWatch(empty *pluginapi.Empty, stream pluginapi.DevicePlugin_ListAndWatchServer) error {
    // Отправляем начальный список
    resp := &pluginapi.ListAndWatchResponse{
        Devices: p.devices,
    }
    if err := stream.Send(resp); err != nil {
        return err
    }

    // Слушаем изменения
    ticker := time.NewTicker(time.Second * 10)
    defer ticker.Stop()

    for {
        select {
        case <-p.stop:
            return nil
        case <-ticker.C:
            // Проверяем состояние устройств
            p.checkDeviceHealth()

            resp := &pluginapi.ListAndWatchResponse{
                Devices: p.devices,
            }
            if err := stream.Send(resp); err != nil {
                return err
            }
        }
    }
}

// Allocate выделяет устройства для контейнера
func (p *CustomDevicePlugin) Allocate(ctx context.Context, req *pluginapi.AllocateRequest) (*pluginapi.AllocateResponse, error) {
    response := &pluginapi.AllocateResponse{}

    for _, containerReq := range req.ContainerRequests {
        containerResp := &pluginapi.ContainerAllocateResponse{
            Envs: map[string]string{
                "CUSTOM_DEVICES": strings.Join(containerReq.DevicesIDs, ","),
            },
            Devices: []*pluginapi.DeviceSpec{},
            Mounts:  []*pluginapi.Mount{},
        }

        for _, deviceID := range containerReq.DevicesIDs {
            // Добавляем device mapping
            containerResp.Devices = append(containerResp.Devices, &pluginapi.DeviceSpec{
                HostPath:      fmt.Sprintf("/dev/custom-device%s", deviceID),
                ContainerPath: fmt.Sprintf("/dev/custom-device%s", deviceID),
                Permissions:   "rw",
            })

            // Добавляем volume mount если нужно
            containerResp.Mounts = append(containerResp.Mounts, &pluginapi.Mount{
                HostPath:      fmt.Sprintf("/var/lib/custom-device/%s", deviceID),
                ContainerPath: fmt.Sprintf("/var/lib/custom-device/%s", deviceID),
                ReadOnly:      false,
            })
        }

        response.ContainerResponses = append(response.ContainerResponses, containerResp)
    }

    return response, nil
}

// PreStartContainer вызывается перед запуском контейнера
func (p *CustomDevicePlugin) PreStartContainer(ctx context.Context, req *pluginapi.PreStartContainerRequest) (*pluginapi.PreStartContainerResponse, error) {
    // Инициализация устройства перед использованием
    for _, deviceID := range req.DevicesIDs {
        if err := initializeDevice(deviceID); err != nil {
            return nil, err
        }
    }
    return &pluginapi.PreStartContainerResponse{}, nil
}

// GetPreferredAllocation определяет предпочтительное распределение устройств
func (p *CustomDevicePlugin) GetPreferredAllocation(ctx context.Context, req *pluginapi.PreferredAllocationRequest) (*pluginapi.PreferredAllocationResponse, error) {
    response := &pluginapi.PreferredAllocationResponse{}

    for _, containerReq := range req.ContainerRequests {
        // Логика выбора оптимальных устройств
        // Например, устройства на одной шине, в одной NUMA зоне
        preferred := selectOptimalDevices(containerReq.AvailableDeviceIDs, int(containerReq.AllocationSize))

        response.ContainerResponses = append(response.ContainerResponses, &pluginapi.ContainerPreferredAllocationResponse{
            DeviceIDs: preferred,
        })
    }

    return response, nil
}

func (p *CustomDevicePlugin) Start() error {
    // Удаляем старый socket
    os.Remove(socketPath)

    // Создаем gRPC сервер
    p.server = grpc.NewServer()
    pluginapi.RegisterDevicePluginServer(p.server, p)

    // Слушаем на socket
    sock, err := net.Listen("unix", socketPath)
    if err != nil {
        return err
    }

    go p.server.Serve(sock)

    // Регистрируемся в kubelet
    return p.register()
}

func (p *CustomDevicePlugin) register() error {
    conn, err := grpc.Dial(
        pluginapi.KubeletSocket,
        grpc.WithInsecure(),
        grpc.WithDialer(func(addr string, timeout time.Duration) (net.Conn, error) {
            return net.DialTimeout("unix", addr, timeout)
        }),
    )
    if err != nil {
        return err
    }
    defer conn.Close()

    client := pluginapi.NewRegistrationClient(conn)

    req := &pluginapi.RegisterRequest{
        Version:      pluginapi.Version,
        Endpoint:     path.Base(socketPath),
        ResourceName: resourceName,
        Options: &pluginapi.DevicePluginOptions{
            PreStartRequired: true,
        },
    }

    _, err = client.Register(context.Background(), req)
    return err
}

func main() {
    plugin := NewCustomDevicePlugin()

    if err := plugin.Start(); err != nil {
        os.Exit(1)
    }

    // Держим процесс запущенным
    select {}
}
```

### NVIDIA GPU Device Plugin (пример использования)

```yaml
# DaemonSet для NVIDIA device plugin
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nvidia-device-plugin-daemonset
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: nvidia-device-plugin-ds
  template:
    metadata:
      labels:
        name: nvidia-device-plugin-ds
    spec:
      tolerations:
        - key: nvidia.com/gpu
          operator: Exists
          effect: NoSchedule
      priorityClassName: system-node-critical
      containers:
        - name: nvidia-device-plugin-ctr
          image: nvcr.io/nvidia/k8s-device-plugin:v0.14.0
          env:
            - name: FAIL_ON_INIT_ERROR
              value: "false"
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: ["ALL"]
          volumeMounts:
            - name: device-plugin
              mountPath: /var/lib/kubelet/device-plugins
      volumes:
        - name: device-plugin
          hostPath:
            path: /var/lib/kubelet/device-plugins
---
# Pod использующий GPU
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  containers:
    - name: cuda-container
      image: nvidia/cuda:11.0-base
      command: ["nvidia-smi"]
      resources:
        limits:
          nvidia.com/gpu: 2  # запрашиваем 2 GPU
```

---

## 12. Практические примеры

### Пример 1: Полный Operator с Webhook

```yaml
# Структура проекта:
# my-operator/
# ├── api/
# │   └── v1/
# │       └── database_types.go
# ├── controllers/
# │   └── database_controller.go
# ├── webhooks/
# │   └── database_webhook.go
# ├── config/
# │   ├── crd/
# │   ├── rbac/
# │   └── webhook/
# └── main.go
```

```go
// webhooks/database_webhook.go
package webhooks

import (
    "context"
    "fmt"

    "k8s.io/apimachinery/pkg/runtime"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/webhook"
    "sigs.k8s.io/controller-runtime/pkg/webhook/admission"

    mycompanyv1 "github.com/mycompany/my-operator/api/v1"
)

// +kubebuilder:webhook:path=/mutate-mycompany-io-v1-database,mutating=true,failurePolicy=fail,sideEffects=None,groups=mycompany.io,resources=databases,verbs=create;update,versions=v1,name=mdatabase.kb.io,admissionReviewVersions=v1

type DatabaseDefaulter struct{}

var _ webhook.CustomDefaulter = &DatabaseDefaulter{}

func (d *DatabaseDefaulter) Default(ctx context.Context, obj runtime.Object) error {
    database := obj.(*mycompanyv1.Database)

    // Устанавливаем defaults
    if database.Spec.Replicas == 0 {
        database.Spec.Replicas = 1
    }

    if database.Spec.Storage.Size == "" {
        database.Spec.Storage.Size = "10Gi"
    }

    // Добавляем labels
    if database.Labels == nil {
        database.Labels = make(map[string]string)
    }
    database.Labels["managed-by"] = "database-operator"

    return nil
}

// +kubebuilder:webhook:path=/validate-mycompany-io-v1-database,mutating=false,failurePolicy=fail,sideEffects=None,groups=mycompany.io,resources=databases,verbs=create;update;delete,versions=v1,name=vdatabase.kb.io,admissionReviewVersions=v1

type DatabaseValidator struct{}

var _ webhook.CustomValidator = &DatabaseValidator{}

func (v *DatabaseValidator) ValidateCreate(ctx context.Context, obj runtime.Object) (warnings admission.Warnings, err error) {
    database := obj.(*mycompanyv1.Database)
    return v.validate(database)
}

func (v *DatabaseValidator) ValidateUpdate(ctx context.Context, oldObj, newObj runtime.Object) (warnings admission.Warnings, err error) {
    oldDatabase := oldObj.(*mycompanyv1.Database)
    newDatabase := newObj.(*mycompanyv1.Database)

    // Запрещаем изменение engine
    if oldDatabase.Spec.Engine != newDatabase.Spec.Engine {
        return nil, fmt.Errorf("engine change is not allowed: %s -> %s",
            oldDatabase.Spec.Engine, newDatabase.Spec.Engine)
    }

    return v.validate(newDatabase)
}

func (v *DatabaseValidator) ValidateDelete(ctx context.Context, obj runtime.Object) (warnings admission.Warnings, err error) {
    database := obj.(*mycompanyv1.Database)

    // Запрещаем удаление production баз без подтверждения
    if database.Labels["environment"] == "production" {
        if database.Annotations["confirm-delete"] != "yes" {
            return nil, fmt.Errorf("cannot delete production database without confirmation annotation")
        }
    }

    return nil, nil
}

func (v *DatabaseValidator) validate(database *mycompanyv1.Database) (warnings admission.Warnings, err error) {
    var errs []string
    var warns admission.Warnings

    // Проверяем версию
    validVersions := map[string][]string{
        "postgresql": {"14", "15", "16"},
        "mysql":      {"8.0", "8.1"},
        "mongodb":    {"6.0", "7.0"},
    }

    versions, ok := validVersions[database.Spec.Engine]
    if !ok {
        errs = append(errs, fmt.Sprintf("unknown engine: %s", database.Spec.Engine))
    } else {
        valid := false
        for _, v := range versions {
            if database.Spec.Version == v {
                valid = true
                break
            }
        }
        if !valid {
            errs = append(errs,
                fmt.Sprintf("invalid version %s for engine %s, valid versions: %v",
                    database.Spec.Version, database.Spec.Engine, versions))
        }
    }

    // Предупреждение о replicas
    if database.Spec.Replicas < 3 {
        warns = append(warns, "less than 3 replicas is not recommended for production")
    }

    if len(errs) > 0 {
        return warns, fmt.Errorf("validation failed: %v", errs)
    }

    return warns, nil
}
```

### Пример 2: Multi-cluster Service Discovery

```yaml
# ClusterSet configuration
apiVersion: about.k8s.io/v1alpha1
kind: ClusterProperty
metadata:
  name: cluster.clusterset.k8s.io
  namespace: default
spec:
  value: my-clusterset
---
apiVersion: about.k8s.io/v1alpha1
kind: ClusterProperty
metadata:
  name: id.k8s.io
  namespace: default
spec:
  value: cluster-west
---
# Export service to other clusters
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: ServiceExport
metadata:
  name: my-service
  namespace: production
---
# Consumer cluster - headless service for discovery
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: production
spec:
  type: ClusterIP
  clusterIP: None
  ports:
    - port: 80
      name: http
---
# DNS format: <service>.<namespace>.svc.clusterset.local
# Example: my-service.production.svc.clusterset.local
```

### Пример 3: Custom Scheduler Extender

```go
// Extender для кастомного скоринга нод
package main

import (
    "encoding/json"
    "net/http"

    v1 "k8s.io/api/core/v1"
    schedulerapi "k8s.io/kube-scheduler/extender/v1"
)

type Extender struct{}

func (e *Extender) Filter(w http.ResponseWriter, r *http.Request) {
    var args schedulerapi.ExtenderArgs
    if err := json.NewDecoder(r.Body).Decode(&args); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    filteredNodes := []v1.Node{}
    failedNodes := make(schedulerapi.FailedNodesMap)

    for _, node := range args.Nodes.Items {
        // Кастомная логика фильтрации
        if e.isNodeSuitable(args.Pod, &node) {
            filteredNodes = append(filteredNodes, node)
        } else {
            failedNodes[node.Name] = "failed custom filter"
        }
    }

    result := &schedulerapi.ExtenderFilterResult{
        Nodes: &v1.NodeList{
            Items: filteredNodes,
        },
        FailedNodes: failedNodes,
    }

    json.NewEncoder(w).Encode(result)
}

func (e *Extender) Prioritize(w http.ResponseWriter, r *http.Request) {
    var args schedulerapi.ExtenderArgs
    if err := json.NewDecoder(r.Body).Decode(&args); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    priorities := schedulerapi.HostPriorityList{}

    for _, node := range args.Nodes.Items {
        score := e.calculateScore(args.Pod, &node)
        priorities = append(priorities, schedulerapi.HostPriority{
            Host:  node.Name,
            Score: score,
        })
    }

    json.NewEncoder(w).Encode(priorities)
}

func (e *Extender) isNodeSuitable(pod *v1.Pod, node *v1.Node) bool {
    // Проверяем кастомные условия
    requiredZone := pod.Labels["required-zone"]
    if requiredZone != "" && node.Labels["topology.kubernetes.io/zone"] != requiredZone {
        return false
    }
    return true
}

func (e *Extender) calculateScore(pod *v1.Pod, node *v1.Node) int64 {
    score := int64(50) // базовый скор

    // Предпочитаем ноды в той же зоне
    if pod.Labels["preferred-zone"] == node.Labels["topology.kubernetes.io/zone"] {
        score += 25
    }

    // Предпочитаем ноды с SSD
    if node.Labels["storage-type"] == "ssd" {
        score += 25
    }

    return score
}

func main() {
    extender := &Extender{}

    http.HandleFunc("/filter", extender.Filter)
    http.HandleFunc("/prioritize", extender.Prioritize)
    http.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
    })

    http.ListenAndServe(":8888", nil)
}
```

```yaml
# Scheduler configuration с extender
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: custom-scheduler
extenders:
  - urlPrefix: "http://scheduler-extender.kube-system.svc:8888"
    filterVerb: "filter"
    prioritizeVerb: "prioritize"
    weight: 5
    enableHTTPS: false
    nodeCacheCapable: false
```

---

## Заключение

Продвинутые возможности Kubernetes открывают широкие возможности для расширения и кастомизации платформы:

1. **CRDs и Operators** позволяют создавать доменно-специфичные абстракции и автоматизировать управление сложными приложениями

2. **Admission Controllers и Webhooks** обеспечивают контроль над всеми изменениями в кластере

3. **Service Mesh** решает задачи observability, security и traffic management на уровне инфраструктуры

4. **Multi-cluster и Federation** необходимы для масштабирования и обеспечения отказоустойчивости

5. **API Aggregation** дает полный контроль над расширением API

6. **Device Plugins** позволяют использовать специализированное оборудование

При использовании этих возможностей важно:
- Начинать с простых решений (CRD + controller-runtime)
- Использовать готовые инструменты (Kubebuilder, Operator SDK)
- Тестировать в изолированных средах
- Следить за производительностью и надежностью

---

## Дополнительные ресурсы

- [Kubernetes Documentation - Extending Kubernetes](https://kubernetes.io/docs/concepts/extend-kubernetes/)
- [Kubebuilder Book](https://book.kubebuilder.io/)
- [Operator SDK](https://sdk.operatorframework.io/)
- [Istio Documentation](https://istio.io/latest/docs/)
- [Linkerd Documentation](https://linkerd.io/docs/)
- [Kubernetes API Aggregation](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/)
