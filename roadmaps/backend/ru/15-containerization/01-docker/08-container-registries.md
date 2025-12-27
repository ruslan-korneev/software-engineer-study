# Container Registries (Реестры контейнеров)

## Введение

**Container Registry** (реестр контейнеров) — это централизованное хранилище для Docker-образов, которое позволяет хранить, версионировать и распространять образы контейнеров.

### Зачем нужен Container Registry?

1. **Централизованное хранение** — все образы в одном месте
2. **Версионирование** — отслеживание версий через теги
3. **Распространение** — доставка образов на различные серверы и окружения
4. **Совместная работа** — команда может использовать общие образы
5. **Интеграция с CI/CD** — автоматическая сборка и развёртывание
6. **Контроль доступа** — управление правами на чтение и запись

### Как работает Container Registry

```
┌─────────────────┐         ┌─────────────────┐         ┌─────────────────┐
│   Developer     │  push   │    Registry     │  pull   │   Production    │
│   (build)       │ ──────► │   (storage)     │ ◄────── │   Server        │
└─────────────────┘         └─────────────────┘         └─────────────────┘
```

---

## Docker Hub

**Docker Hub** — это официальный публичный реестр Docker, крупнейший репозиторий контейнерных образов в мире.

### Регистрация и настройка

1. Зарегистрируйтесь на [hub.docker.com](https://hub.docker.com)
2. Подтвердите email
3. Авторизуйтесь через CLI:

```bash
# Вход в Docker Hub
docker login

# Или с указанием учётных данных
docker login -u username -p password

# Использование token вместо пароля (рекомендуется)
docker login -u username --password-stdin < token.txt
```

### Публичные и приватные репозитории

| Тип репозитория | Доступ | Использование |
|-----------------|--------|---------------|
| **Public** | Все могут pull | Open-source проекты, общедоступные образы |
| **Private** | Только авторизованные | Проприетарный код, внутренние проекты |

```bash
# Создание публичного образа
docker build -t username/myapp:latest .
docker push username/myapp:latest

# Приватный репозиторий создаётся автоматически при первом push
# Настройка приватности — через веб-интерфейс Docker Hub
```

### Лимиты и тарифы Docker Hub

| План | Pull-лимиты | Приватные репозитории | Цена |
|------|-------------|----------------------|------|
| **Anonymous** | 100 pulls / 6 часов | 0 | Бесплатно |
| **Free (авторизован)** | 200 pulls / 6 часов | 1 | Бесплатно |
| **Pro** | Без лимитов | Безлимитно | $5/мес |
| **Team** | Без лимитов | Безлимитно | $7/user/мес |
| **Business** | Без лимитов | Безлимитно | $24/user/мес |

> **Важно:** Анонимные pull-запросы лимитируются по IP-адресу, что может вызвать проблемы в CI/CD.

---

## Другие популярные реестры

### GitHub Container Registry (ghcr.io)

GitHub предоставляет собственный реестр контейнеров, интегрированный с репозиториями.

```bash
# Авторизация с Personal Access Token
echo $GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin

# Push образа
docker tag myapp:latest ghcr.io/username/myapp:latest
docker push ghcr.io/username/myapp:latest
```

**Преимущества:**
- Интеграция с GitHub Actions
- Привязка к репозиториям
- Бесплатные приватные репозитории для публичных проектов
- Гранулярные права доступа

**Лимиты бесплатного плана:**
- Storage: 500 MB
- Data transfer: 1 GB/месяц

### Google Container Registry (GCR) / Artifact Registry

Google Cloud предлагает GCR и более новый Artifact Registry.

```bash
# Настройка аутентификации
gcloud auth configure-docker

# Для Artifact Registry
gcloud auth configure-docker us-docker.pkg.dev

# Push образа
docker tag myapp:latest gcr.io/project-id/myapp:latest
docker push gcr.io/project-id/myapp:latest

# Artifact Registry (рекомендуется)
docker tag myapp:latest us-docker.pkg.dev/project-id/repo-name/myapp:latest
docker push us-docker.pkg.dev/project-id/repo-name/myapp:latest
```

**Особенности:**
- Интеграция с GKE (Google Kubernetes Engine)
- Vulnerability scanning
- Binary Authorization
- Региональное хранение

### Amazon Elastic Container Registry (ECR)

AWS ECR — полностью управляемый реестр для AWS-инфраструктуры.

```bash
# Получение токена аутентификации
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

# Создание репозитория
aws ecr create-repository --repository-name myapp

# Push образа
docker tag myapp:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
```

**Особенности:**
- Интеграция с ECS, EKS, Lambda
- Lifecycle policies для автоматической очистки
- Cross-region и cross-account replication
- Image scanning с Amazon Inspector

### Azure Container Registry (ACR)

Microsoft Azure Container Registry для Azure-инфраструктуры.

```bash
# Вход через Azure CLI
az acr login --name myregistry

# Push образа
docker tag myapp:latest myregistry.azurecr.io/myapp:latest
docker push myregistry.azurecr.io/myapp:latest
```

**Особенности:**
- Интеграция с AKS (Azure Kubernetes Service)
- Geo-replication
- Azure Active Directory аутентификация
- ACR Tasks для автоматической сборки

### GitLab Container Registry

Встроенный реестр в GitLab, доступный для каждого проекта.

```bash
# Авторизация
docker login registry.gitlab.com

# Push образа
docker tag myapp:latest registry.gitlab.com/username/project/myapp:latest
docker push registry.gitlab.com/username/project/myapp:latest
```

**Преимущества:**
- Интеграция с GitLab CI/CD
- Автоматическая очистка старых образов
- Привязка к проектам и группам
- Бесплатно на gitlab.com

---

## Работа с реестрами

### docker login

Авторизация в реестре контейнеров.

```bash
# Docker Hub (по умолчанию)
docker login

# Конкретный реестр
docker login registry.example.com

# С указанием credentials
docker login -u username -p password registry.example.com

# Безопасный ввод пароля через stdin
echo $PASSWORD | docker login -u username --password-stdin registry.example.com

# Выход из реестра
docker logout
docker logout registry.example.com
```

**Где хранятся credentials:**
```bash
# Файл конфигурации
cat ~/.docker/config.json
```

```json
{
  "auths": {
    "https://index.docker.io/v1/": {
      "auth": "base64_encoded_credentials"
    },
    "ghcr.io": {
      "auth": "base64_encoded_credentials"
    }
  }
}
```

> **Внимание:** По умолчанию credentials хранятся в base64 (не зашифрованы). Используйте credential helpers для безопасного хранения.

### docker tag

Создание тега (alias) для образа.

```bash
# Синтаксис
docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]

# Примеры
# Локальный образ -> Docker Hub
docker tag myapp:latest username/myapp:latest

# Добавление версии
docker tag myapp:latest username/myapp:v1.0.0

# Для другого реестра
docker tag myapp:latest ghcr.io/username/myapp:latest

# Множественные теги
docker tag myapp:latest username/myapp:latest
docker tag myapp:latest username/myapp:v1.0.0
docker tag myapp:latest username/myapp:$(git rev-parse --short HEAD)
```

### docker push

Отправка образа в реестр.

```bash
# Синтаксис
docker push [OPTIONS] NAME[:TAG]

# Push одного тега
docker push username/myapp:latest

# Push всех тегов образа
docker push --all-tags username/myapp

# Push в приватный реестр
docker push registry.example.com/myapp:v1.0.0
```

**Процесс push:**
1. Docker проверяет, какие слои уже есть в реестре
2. Загружаются только новые/изменённые слои
3. Создаётся manifest с метаданными образа

### docker pull

Загрузка образа из реестра.

```bash
# Синтаксис
docker pull [OPTIONS] NAME[:TAG|@DIGEST]

# Последняя версия
docker pull nginx

# Конкретная версия
docker pull nginx:1.24

# По digest (неизменяемая ссылка)
docker pull nginx@sha256:abc123...

# Из другого реестра
docker pull ghcr.io/username/myapp:latest

# Все теги
docker pull --all-tags nginx

# Конкретная платформа
docker pull --platform linux/arm64 nginx
```

---

## Приватный Docker Registry

### Развёртывание своего Registry

Docker предоставляет официальный образ `registry` для создания собственного реестра.

#### Базовое развёртывание

```bash
# Запуск простого registry
docker run -d \
  --name registry \
  -p 5000:5000 \
  registry:2

# Проверка работы
curl http://localhost:5000/v2/_catalog

# Push образа в локальный registry
docker tag myapp:latest localhost:5000/myapp:latest
docker push localhost:5000/myapp:latest
```

#### Развёртывание с Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  registry:
    image: registry:2
    ports:
      - "5000:5000"
    volumes:
      - registry-data:/var/lib/registry
      - ./config.yml:/etc/docker/registry/config.yml:ro
    environment:
      REGISTRY_STORAGE_DELETE_ENABLED: "true"
    restart: unless-stopped

volumes:
  registry-data:
```

### Настройка и конфигурация

#### Конфигурационный файл

```yaml
# config.yml
version: 0.1

log:
  level: info
  formatter: text

storage:
  filesystem:
    rootdirectory: /var/lib/registry
  delete:
    enabled: true

http:
  addr: :5000
  headers:
    X-Content-Type-Options: [nosniff]

health:
  storagedriver:
    enabled: true
    interval: 10s
    threshold: 3
```

#### Registry с TLS (HTTPS)

```yaml
# docker-compose.yml
version: '3.8'

services:
  registry:
    image: registry:2
    ports:
      - "443:443"
    volumes:
      - registry-data:/var/lib/registry
      - ./certs:/certs:ro
    environment:
      REGISTRY_HTTP_ADDR: 0.0.0.0:443
      REGISTRY_HTTP_TLS_CERTIFICATE: /certs/domain.crt
      REGISTRY_HTTP_TLS_KEY: /certs/domain.key
    restart: unless-stopped

volumes:
  registry-data:
```

#### Registry с аутентификацией

```bash
# Создание htpasswd файла
mkdir auth
docker run --rm --entrypoint htpasswd httpd:2 \
  -Bbn admin secretpassword > auth/htpasswd
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  registry:
    image: registry:2
    ports:
      - "5000:5000"
    volumes:
      - registry-data:/var/lib/registry
      - ./auth:/auth:ro
    environment:
      REGISTRY_AUTH: htpasswd
      REGISTRY_AUTH_HTPASSWD_REALM: "Registry Realm"
      REGISTRY_AUTH_HTPASSWD_PATH: /auth/htpasswd
    restart: unless-stopped

volumes:
  registry-data:
```

#### S3-совместимое хранилище

```yaml
# config.yml
version: 0.1

storage:
  s3:
    accesskey: AKIAIOSFODNN7EXAMPLE
    secretkey: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
    region: us-east-1
    bucket: my-registry-bucket
    encrypt: true
    secure: true
```

#### Web UI для Registry

```yaml
# docker-compose.yml
version: '3.8'

services:
  registry:
    image: registry:2
    ports:
      - "5000:5000"
    volumes:
      - registry-data:/var/lib/registry
    environment:
      REGISTRY_STORAGE_DELETE_ENABLED: "true"
    restart: unless-stopped

  registry-ui:
    image: joxit/docker-registry-ui:latest
    ports:
      - "8080:80"
    environment:
      REGISTRY_TITLE: "My Private Registry"
      REGISTRY_URL: http://registry:5000
      SINGLE_REGISTRY: "true"
      DELETE_IMAGES: "true"
    depends_on:
      - registry

volumes:
  registry-data:
```

---

## Безопасность реестров

### Аутентификация

#### Token-based аутентификация

```yaml
# config.yml
auth:
  token:
    realm: https://auth.example.com/token
    service: registry.example.com
    issuer: "Auth Service"
    rootcertbundle: /certs/bundle.crt
```

#### OAuth2 / OIDC

```yaml
# Пример для Harbor (enterprise registry)
auth_mode: oidc_auth
oidc_name: "Google"
oidc_endpoint: "https://accounts.google.com"
oidc_client_id: "your-client-id"
oidc_client_secret: "your-client-secret"
oidc_scope: "openid,profile,email"
```

### Credential Helpers

Безопасное хранение учётных данных.

```bash
# Установка credential helper для macOS
brew install docker-credential-helper

# Для Linux (pass)
sudo apt install pass docker-credential-pass

# Настройка в ~/.docker/config.json
{
  "credsStore": "osxkeychain"  # или "pass", "secretservice"
}
```

### Сканирование образов

#### Trivy (рекомендуется)

```bash
# Установка
brew install trivy  # macOS
apt install trivy   # Debian/Ubuntu

# Сканирование образа
trivy image myapp:latest

# Сканирование с отчётом
trivy image --format json --output report.json myapp:latest

# Только критические и высокие уязвимости
trivy image --severity CRITICAL,HIGH myapp:latest

# Сканирование в CI/CD (выход с ошибкой при уязвимостях)
trivy image --exit-code 1 --severity CRITICAL myapp:latest
```

#### Docker Scout (встроенный в Docker)

```bash
# Включение Docker Scout
docker scout enroll

# Сканирование образа
docker scout cves myapp:latest

# Рекомендации по исправлению
docker scout recommendations myapp:latest

# Сравнение образов
docker scout compare myapp:v1 myapp:v2
```

#### Сканирование в GitHub Actions

```yaml
# .github/workflows/scan.yml
name: Security Scan

on:
  push:
    branches: [main]

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'

      - name: Upload Trivy scan results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'
```

### Подпись образов

#### Docker Content Trust (DCT)

```bash
# Включение DCT
export DOCKER_CONTENT_TRUST=1

# Push подписанного образа
docker push username/myapp:latest
# Docker автоматически подпишет образ

# Генерация ключей
docker trust key generate mykey
docker trust signer add --key mykey.pub mykey username/myapp
```

#### Cosign (Sigstore)

```bash
# Установка cosign
brew install cosign  # macOS

# Генерация ключевой пары
cosign generate-key-pair

# Подпись образа
cosign sign --key cosign.key username/myapp:latest

# Проверка подписи
cosign verify --key cosign.pub username/myapp:latest
```

#### Подпись в CI/CD (GitHub Actions)

```yaml
# .github/workflows/sign.yml
jobs:
  build-and-sign:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      id-token: write  # Для keyless signing

    steps:
      - uses: actions/checkout@v4

      - name: Install cosign
        uses: sigstore/cosign-installer@v3

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push
        id: build
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:latest

      - name: Sign image (keyless)
        run: |
          cosign sign --yes ghcr.io/${{ github.repository }}@${{ steps.build.outputs.digest }}
```

---

## Best Practices

### Именование образов

```bash
# Формат: registry/namespace/repository:tag

# Хорошие примеры
mycompany/backend-api:v1.2.3
mycompany/backend-api:1.2.3-alpine
ghcr.io/myorg/myapp:sha-abc1234

# Плохие примеры
myapp              # Нет тега
myapp:latest       # Неинформативный тег
app:v1             # Слишком общее имя
```

### Стратегия тегирования

```bash
# Рекомендуемые теги

# 1. Semantic Versioning
docker tag myapp:latest mycompany/myapp:1.2.3
docker tag myapp:latest mycompany/myapp:1.2
docker tag myapp:latest mycompany/myapp:1

# 2. Git-based теги
docker tag myapp:latest mycompany/myapp:$(git rev-parse --short HEAD)
docker tag myapp:latest mycompany/myapp:$(git describe --tags)

# 3. Timestamp
docker tag myapp:latest mycompany/myapp:$(date +%Y%m%d-%H%M%S)

# 4. Environment
docker tag myapp:latest mycompany/myapp:staging
docker tag myapp:latest mycompany/myapp:production
```

### Автоматическая очистка

```bash
# Docker Hub — через веб-интерфейс или API

# ECR Lifecycle Policy
aws ecr put-lifecycle-policy \
  --repository-name myapp \
  --lifecycle-policy-text '{
    "rules": [
      {
        "rulePriority": 1,
        "description": "Keep last 10 images",
        "selection": {
          "tagStatus": "any",
          "countType": "imageCountMoreThan",
          "countNumber": 10
        },
        "action": {
          "type": "expire"
        }
      }
    ]
  }'

# Локальный registry — garbage collection
docker exec registry bin/registry garbage-collect /etc/docker/registry/config.yml
```

### Multi-platform образы

```bash
# Создание multi-platform образа
docker buildx create --name multiplatform --use

docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --tag username/myapp:latest \
  --push \
  .
```

### Организация репозиториев

```
# Структура для проекта
mycompany/
├── backend-api           # Backend сервис
├── frontend              # Frontend приложение
├── worker                # Background workers
├── migrations            # Database migrations
└── base-images/
    ├── python            # Базовый Python образ
    ├── node              # Базовый Node.js образ
    └── golang            # Базовый Go образ
```

---

## Типичные ошибки

### 1. Использование latest в production

```bash
# ❌ Плохо — непредсказуемо
docker pull myapp:latest

# ✅ Хорошо — конкретная версия
docker pull myapp:v1.2.3
docker pull myapp@sha256:abc123...
```

### 2. Хранение секретов в образе

```dockerfile
# ❌ Плохо — секреты в образе
ENV AWS_SECRET_KEY=secret123

# ✅ Хорошо — секреты через runtime
# Передавать через -e или secrets в docker-compose/k8s
```

### 3. Push без тегирования

```bash
# ❌ Плохо — push несуществующего образа
docker push myapp:latest
# Error: An image does not exist locally with the tag: myapp

# ✅ Хорошо — сначала тег, потом push
docker tag myapp:latest username/myapp:latest
docker push username/myapp:latest
```

### 4. Незашифрованные credentials

```bash
# ❌ Плохо — пароль в командной строке
docker login -u user -p password

# ✅ Хорошо — использование stdin
echo $TOKEN | docker login -u user --password-stdin

# ✅ Ещё лучше — credential helper
# Настроить в ~/.docker/config.json
```

### 5. Отсутствие сканирования

```bash
# ❌ Плохо — push без проверки
docker build -t myapp:latest .
docker push myapp:latest

# ✅ Хорошо — сканирование перед push
docker build -t myapp:latest .
trivy image --exit-code 1 --severity CRITICAL myapp:latest
docker push myapp:latest
```

### 6. Неправильный формат имени

```bash
# ❌ Плохо — неверный формат
docker push MyApp:Latest  # Uppercase не допускается
docker push my app:v1     # Пробелы не допускаются

# ✅ Хорошо — lowercase и дефисы
docker push my-app:v1
docker push myapp:latest
```

### 7. Rate Limiting в CI/CD

```bash
# ❌ Плохо — анонимный pull в CI
docker pull nginx:latest  # 100 pulls / 6 часов

# ✅ Хорошо — авторизованный pull
echo $DOCKER_TOKEN | docker login -u $DOCKER_USER --password-stdin
docker pull nginx:latest  # 200+ pulls

# ✅ Ещё лучше — использовать свой registry mirror
docker pull registry.mycompany.com/library/nginx:latest
```

---

## Сравнительная таблица реестров

| Функция | Docker Hub | GHCR | ECR | GCR | ACR |
|---------|------------|------|-----|-----|-----|
| Бесплатный tier | ✅ | ✅ | ✅ | ✅ | ❌ |
| Приватные репо (бесплатно) | 1 | ∞* | ✅ | ✅ | ❌ |
| Vulnerability scanning | ✅ | ✅ | ✅ | ✅ | ✅ |
| Geo-replication | ❌ | ❌ | ✅ | ✅ | ✅ |
| CI/CD интеграция | GitHub Actions | GitHub Actions | CodePipeline | Cloud Build | Azure Pipelines |
| OIDC auth | ❌ | ✅ | ✅ | ✅ | ✅ |

*Для публичных репозиториев

---

## Полезные команды

```bash
# Просмотр всех образов для реестра
docker images "registry.example.com/*"

# Удаление всех тегов для образа
docker images -q username/myapp | xargs docker rmi

# Просмотр manifest образа
docker manifest inspect nginx:latest

# Проверка доступных платформ
docker manifest inspect --verbose nginx:latest

# Копирование образов между реестрами (с помощью crane)
crane copy source-registry/image:tag target-registry/image:tag

# Просмотр layers образа
docker history username/myapp:latest

# Проверка размера образа
docker images --format "{{.Repository}}:{{.Tag}} {{.Size}}" | grep myapp
```

---

## Заключение

Container Registry — критически важный компонент инфраструктуры контейнеризации. Правильный выбор и настройка реестра обеспечивают:

- **Надёжное хранение** образов
- **Быструю доставку** в production
- **Безопасность** через сканирование и подписи
- **Эффективную работу** команды

Для начала достаточно Docker Hub, но для production-окружения рекомендуется рассмотреть:
- Облачные реестры (ECR, GCR, ACR) при использовании соответствующего облака
- GitHub Container Registry для open-source проектов
- Приватный Registry для полного контроля над инфраструктурой
