# Деплой контейнеров Docker

## Введение в деплой контейнеров

Деплой контейнеров — это процесс развёртывания Docker-образов в production-среде. В отличие от традиционного развёртывания приложений, контейнеризация обеспечивает:

- **Консистентность окружения** — образ работает одинаково везде
- **Изоляция** — приложения не конфликтуют друг с другом
- **Масштабируемость** — легко запустить несколько копий контейнера
- **Быстрый откат** — возврат к предыдущей версии за секунды
- **Упрощённый CI/CD** — автоматизация сборки и деплоя

### Типичный workflow деплоя

```
1. Разработка → 2. Сборка образа → 3. Тестирование → 4. Push в Registry → 5. Deploy на сервер
```

---

## Подготовка образов к продакшену

### Оптимизация размера образа

Меньший размер образа означает:
- Быстрее загрузка и деплой
- Меньше поверхность атаки
- Экономия дискового пространства и трафика

#### Используйте минимальные базовые образы

```dockerfile
# Плохо - 900+ MB
FROM python:3.11

# Лучше - ~120 MB
FROM python:3.11-slim

# Ещё лучше для Go/Rust - ~5 MB
FROM alpine:3.19

# Идеально для статически скомпилированных приложений - 0 MB
FROM scratch
```

#### Объединяйте RUN команды

```dockerfile
# Плохо - каждая команда создаёт новый слой
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get clean

# Хорошо - один слой, меньше размер
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

#### Используйте .dockerignore

```dockerignore
# .dockerignore
.git
.gitignore
node_modules
*.md
*.log
.env
.env.*
__pycache__
*.pyc
tests/
docs/
.coverage
.pytest_cache
```

### Multi-stage builds

Multi-stage сборка позволяет использовать несколько FROM в одном Dockerfile и копировать только нужные артефакты в финальный образ.

#### Пример для Go-приложения

```dockerfile
# Этап сборки
FROM golang:1.22-alpine AS builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main .

# Финальный образ
FROM alpine:3.19

RUN apk --no-cache add ca-certificates
WORKDIR /root/

COPY --from=builder /app/main .

EXPOSE 8080
CMD ["./main"]
```

#### Пример для Python-приложения

```dockerfile
# Этап сборки зависимостей
FROM python:3.11-slim AS builder

WORKDIR /app

RUN pip install --no-cache-dir poetry
COPY pyproject.toml poetry.lock ./
RUN poetry export -f requirements.txt --output requirements.txt --without-hashes

# Финальный образ
FROM python:3.11-slim

WORKDIR /app

# Копируем requirements из builder
COPY --from=builder /app/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Копируем код приложения
COPY . .

# Создаём непривилегированного пользователя
RUN useradd -m appuser && chown -R appuser:appuser /app
USER appuser

EXPOSE 8000
CMD ["gunicorn", "-b", "0.0.0.0:8000", "app:app"]
```

#### Пример для Node.js

```dockerfile
# Этап сборки
FROM node:20-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

COPY . .
RUN npm run build

# Финальный образ
FROM node:20-alpine

WORKDIR /app
ENV NODE_ENV=production

COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./

USER node
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

### Тегирование и версионирование

#### Стратегии тегирования

```bash
# Semantic versioning
docker build -t myapp:1.2.3 .
docker build -t myapp:1.2 .
docker build -t myapp:1 .
docker build -t myapp:latest .

# Git-based тегирование
docker build -t myapp:$(git rev-parse --short HEAD) .
docker build -t myapp:$(git describe --tags --always) .

# Дата + коммит
docker build -t myapp:2024.01.15-abc1234 .

# Branch-based (для dev/staging)
docker build -t myapp:develop .
docker build -t myapp:feature-auth .
```

#### Пример скрипта тегирования

```bash
#!/bin/bash
# build.sh

IMAGE_NAME="myapp"
REGISTRY="ghcr.io/myorg"
GIT_SHA=$(git rev-parse --short HEAD)
VERSION=$(git describe --tags --always --dirty)
DATE=$(date +%Y%m%d)

# Сборка с несколькими тегами
docker build \
    -t ${REGISTRY}/${IMAGE_NAME}:${VERSION} \
    -t ${REGISTRY}/${IMAGE_NAME}:${GIT_SHA} \
    -t ${REGISTRY}/${IMAGE_NAME}:${DATE}-${GIT_SHA} \
    -t ${REGISTRY}/${IMAGE_NAME}:latest \
    --label "org.opencontainers.image.revision=${GIT_SHA}" \
    --label "org.opencontainers.image.created=$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
    .

# Push всех тегов
docker push ${REGISTRY}/${IMAGE_NAME} --all-tags
```

---

## Container Registry

Container Registry — хранилище Docker-образов, откуда они загружаются на production-серверы.

### Docker Hub

Официальный registry от Docker.

```bash
# Логин
docker login

# Push образа
docker tag myapp:latest username/myapp:latest
docker push username/myapp:latest

# Pull образа
docker pull username/myapp:latest
```

**Особенности:**
- Бесплатные публичные репозитории
- 1 приватный репозиторий бесплатно
- Rate limiting для анонимных пользователей

### GitHub Container Registry (GHCR)

Интегрирован с GitHub Actions.

```bash
# Логин с Personal Access Token
echo $GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin

# Push образа
docker tag myapp:latest ghcr.io/username/myapp:latest
docker push ghcr.io/username/myapp:latest
```

### AWS ECR (Elastic Container Registry)

```bash
# Логин в ECR
aws ecr get-login-password --region eu-west-1 | \
    docker login --username AWS --password-stdin 123456789.dkr.ecr.eu-west-1.amazonaws.com

# Создание репозитория
aws ecr create-repository --repository-name myapp

# Push образа
docker tag myapp:latest 123456789.dkr.ecr.eu-west-1.amazonaws.com/myapp:latest
docker push 123456789.dkr.ecr.eu-west-1.amazonaws.com/myapp:latest
```

### Google Container Registry (GCR) / Artifact Registry

```bash
# Логин
gcloud auth configure-docker

# Push образа
docker tag myapp:latest gcr.io/project-id/myapp:latest
docker push gcr.io/project-id/myapp:latest

# Artifact Registry (новый формат)
docker tag myapp:latest europe-west1-docker.pkg.dev/project-id/repo/myapp:latest
docker push europe-west1-docker.pkg.dev/project-id/repo/myapp:latest
```

### Azure Container Registry (ACR)

```bash
# Логин
az acr login --name myregistry

# Push образа
docker tag myapp:latest myregistry.azurecr.io/myapp:latest
docker push myregistry.azurecr.io/myapp:latest
```

### Self-hosted Registry

Для внутренних нужд можно поднять собственный registry.

```yaml
# docker-compose.yml для self-hosted registry
version: "3.8"

services:
  registry:
    image: registry:2
    ports:
      - "5000:5000"
    volumes:
      - registry-data:/var/lib/registry
    environment:
      REGISTRY_STORAGE_DELETE_ENABLED: "true"
      REGISTRY_HTTP_HEADERS_Access-Control-Allow-Origin: '[*]'

  registry-ui:
    image: joxit/docker-registry-ui:latest
    ports:
      - "8080:80"
    environment:
      - REGISTRY_TITLE=My Registry
      - REGISTRY_URL=http://registry:5000
    depends_on:
      - registry

volumes:
  registry-data:
```

```bash
# Использование self-hosted registry
docker tag myapp:latest localhost:5000/myapp:latest
docker push localhost:5000/myapp:latest
```

---

## Способы деплоя

### Ручной деплой на VPS/VM

Простейший способ для небольших проектов.

```bash
# На локальной машине: собираем и пушим
docker build -t ghcr.io/myorg/myapp:v1.0.0 .
docker push ghcr.io/myorg/myapp:v1.0.0

# На сервере: логинимся и запускаем
ssh user@server

docker login ghcr.io
docker pull ghcr.io/myorg/myapp:v1.0.0

# Останавливаем старый контейнер
docker stop myapp && docker rm myapp

# Запускаем новый
docker run -d \
    --name myapp \
    --restart unless-stopped \
    -p 80:8000 \
    -e DATABASE_URL="postgres://..." \
    -v /app/data:/data \
    ghcr.io/myorg/myapp:v1.0.0
```

#### Скрипт автоматизации ручного деплоя

```bash
#!/bin/bash
# deploy.sh

set -e

SERVER="user@192.168.1.100"
IMAGE="ghcr.io/myorg/myapp"
TAG="${1:-latest}"
CONTAINER_NAME="myapp"

echo "Deploying ${IMAGE}:${TAG} to ${SERVER}..."

ssh ${SERVER} << EOF
    set -e

    # Pull новый образ
    docker pull ${IMAGE}:${TAG}

    # Останавливаем и удаляем старый контейнер
    docker stop ${CONTAINER_NAME} 2>/dev/null || true
    docker rm ${CONTAINER_NAME} 2>/dev/null || true

    # Запускаем новый контейнер
    docker run -d \
        --name ${CONTAINER_NAME} \
        --restart unless-stopped \
        -p 80:8000 \
        --env-file /home/user/.env \
        ${IMAGE}:${TAG}

    # Проверяем здоровье
    sleep 5
    docker ps | grep ${CONTAINER_NAME}

    # Очистка старых образов
    docker image prune -f
EOF

echo "Deployment complete!"
```

### Docker Compose в продакшене

Для приложений с несколькими сервисами.

```yaml
# docker-compose.prod.yml
version: "3.8"

services:
  app:
    image: ghcr.io/myorg/myapp:${TAG:-latest}
    restart: unless-stopped
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=${REDIS_URL}
      - SECRET_KEY=${SECRET_KEY}
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 1G
        reservations:
          cpus: '0.5'
          memory: 256M

  db:
    image: postgres:16-alpine
    restart: unless-stopped
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=${POSTGRES_DB}
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  nginx:
    image: nginx:alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
    depends_on:
      - app

volumes:
  postgres_data:
  redis_data:
```

```bash
# Деплой с Compose
TAG=v1.0.0 docker compose -f docker-compose.prod.yml up -d

# Обновление одного сервиса
TAG=v1.1.0 docker compose -f docker-compose.prod.yml up -d --no-deps app

# Просмотр логов
docker compose -f docker-compose.prod.yml logs -f app
```

### Docker Swarm

Нативная оркестрация Docker для кластеров.

```bash
# Инициализация Swarm
docker swarm init --advertise-addr 192.168.1.100

# Добавление worker-нод
docker swarm join --token SWMTKN-... 192.168.1.100:2377
```

```yaml
# stack.yml для Docker Swarm
version: "3.8"

services:
  app:
    image: ghcr.io/myorg/myapp:latest
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
        order: start-first
      rollback_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
    ports:
      - "8000:8000"
    networks:
      - webnet
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  visualizer:
    image: dockersamples/visualizer:stable
    ports:
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    deploy:
      placement:
        constraints:
          - node.role == manager

networks:
  webnet:
```

```bash
# Деплой стека
docker stack deploy -c stack.yml myapp

# Масштабирование
docker service scale myapp_app=5

# Обновление образа
docker service update --image ghcr.io/myorg/myapp:v2.0.0 myapp_app

# Просмотр состояния
docker service ls
docker service ps myapp_app
```

### Kubernetes (обзор)

Kubernetes — наиболее мощная платформа оркестрации контейнеров.

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
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
      containers:
      - name: myapp
        image: ghcr.io/myorg/myapp:v1.0.0
        ports:
        - containerPort: 8000
        resources:
          limits:
            cpu: "500m"
            memory: "512Mi"
          requests:
            cpu: "250m"
            memory: "256Mi"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: database-url
---
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8000
  type: LoadBalancer
```

```bash
# Применение конфигурации
kubectl apply -f deployment.yaml

# Проверка состояния
kubectl get pods
kubectl get deployments

# Обновление образа
kubectl set image deployment/myapp myapp=ghcr.io/myorg/myapp:v2.0.0

# Откат
kubectl rollout undo deployment/myapp

# Масштабирование
kubectl scale deployment/myapp --replicas=5
```

### Managed Container Services

#### AWS ECS (Elastic Container Service)

```json
// task-definition.json
{
  "family": "myapp",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::123456789:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "myapp",
      "image": "123456789.dkr.ecr.eu-west-1.amazonaws.com/myapp:latest",
      "portMappings": [
        {
          "containerPort": 8000,
          "protocol": "tcp"
        }
      ],
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8000/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      },
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/myapp",
          "awslogs-region": "eu-west-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

```bash
# Регистрация task definition
aws ecs register-task-definition --cli-input-json file://task-definition.json

# Обновление сервиса
aws ecs update-service \
    --cluster myapp-cluster \
    --service myapp-service \
    --task-definition myapp:2 \
    --force-new-deployment
```

#### Google Cloud Run

```bash
# Деплой в Cloud Run
gcloud run deploy myapp \
    --image gcr.io/project-id/myapp:latest \
    --platform managed \
    --region europe-west1 \
    --allow-unauthenticated \
    --memory 512Mi \
    --cpu 1 \
    --min-instances 0 \
    --max-instances 10 \
    --port 8000
```

#### Azure Container Instances

```bash
# Создание контейнера
az container create \
    --resource-group myResourceGroup \
    --name myapp \
    --image myregistry.azurecr.io/myapp:latest \
    --cpu 1 \
    --memory 1.5 \
    --registry-login-server myregistry.azurecr.io \
    --registry-username myregistry \
    --registry-password $ACR_PASSWORD \
    --dns-name-label myapp-unique \
    --ports 8000
```

---

## CI/CD для контейнеров

### Сборка образов в CI

Основные принципы:
- Кешируйте слои образа для ускорения сборки
- Используйте multi-stage builds
- Сканируйте образы на уязвимости
- Автоматизируйте тегирование

### GitHub Actions

```yaml
# .github/workflows/docker.yml
name: Build and Deploy

on:
  push:
    branches: [main]
    tags: ['v*']
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Container Registry
        if: github.event_name != 'pull_request'
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Scan for vulnerabilities
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          format: 'table'
          exit-code: '1'
          ignore-unfixed: true
          severity: 'CRITICAL,HIGH'

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production

    steps:
      - name: Deploy to server
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.HOST }}
          username: ${{ secrets.USERNAME }}
          key: ${{ secrets.SSH_KEY }}
          script: |
            docker pull ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:main
            docker stop myapp || true
            docker rm myapp || true
            docker run -d \
              --name myapp \
              --restart unless-stopped \
              -p 8000:8000 \
              --env-file /home/deploy/.env \
              ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:main
```

### GitLab CI

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - deploy

variables:
  DOCKER_TLS_CERTDIR: "/certs"
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  IMAGE_LATEST: $CI_REGISTRY_IMAGE:latest

build:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build --cache-from $IMAGE_LATEST -t $IMAGE_TAG -t $IMAGE_LATEST .
    - docker push $IMAGE_TAG
    - docker push $IMAGE_LATEST
  rules:
    - if: $CI_COMMIT_BRANCH

test:
  stage: test
  image: docker:24
  services:
    - docker:24-dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker pull $IMAGE_TAG
    - docker run --rm $IMAGE_TAG pytest tests/
  rules:
    - if: $CI_COMMIT_BRANCH

scan:
  stage: test
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  script:
    - trivy image --exit-code 1 --severity CRITICAL,HIGH $IMAGE_TAG
  allow_failure: true
  rules:
    - if: $CI_COMMIT_BRANCH

deploy_staging:
  stage: deploy
  image: alpine:latest
  before_script:
    - apk add --no-cache openssh-client
    - eval $(ssh-agent -s)
    - echo "$SSH_PRIVATE_KEY" | ssh-add -
    - mkdir -p ~/.ssh && chmod 700 ~/.ssh
    - echo "$SSH_KNOWN_HOSTS" >> ~/.ssh/known_hosts
  script:
    - |
      ssh $STAGING_USER@$STAGING_HOST << EOF
        docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
        docker pull $IMAGE_TAG
        docker stop myapp || true
        docker rm myapp || true
        docker run -d --name myapp --restart unless-stopped -p 8000:8000 $IMAGE_TAG
      EOF
  environment:
    name: staging
    url: https://staging.example.com
  rules:
    - if: $CI_COMMIT_BRANCH == "develop"

deploy_production:
  stage: deploy
  image: alpine:latest
  before_script:
    - apk add --no-cache openssh-client
    - eval $(ssh-agent -s)
    - echo "$SSH_PRIVATE_KEY" | ssh-add -
    - mkdir -p ~/.ssh && chmod 700 ~/.ssh
    - echo "$SSH_KNOWN_HOSTS" >> ~/.ssh/known_hosts
  script:
    - |
      ssh $PROD_USER@$PROD_HOST << EOF
        docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
        docker pull $IMAGE_TAG
        docker stop myapp || true
        docker rm myapp || true
        docker run -d --name myapp --restart unless-stopped -p 8000:8000 $IMAGE_TAG
      EOF
  environment:
    name: production
    url: https://example.com
  rules:
    - if: $CI_COMMIT_TAG
  when: manual
```

---

## Rolling Updates и Zero-Downtime Deployments

### Rolling Update с Docker Compose

```yaml
# docker-compose.yml
version: "3.8"

services:
  app:
    image: myapp:latest
    deploy:
      replicas: 3
      update_config:
        parallelism: 1      # Обновляем по одному контейнеру
        delay: 10s          # Пауза между обновлениями
        failure_action: rollback
        order: start-first  # Сначала запустить новый, потом остановить старый
```

### Blue-Green Deployment с Nginx

```yaml
# docker-compose.blue-green.yml
version: "3.8"

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - app-blue
      - app-green

  app-blue:
    image: myapp:v1.0.0
    expose:
      - "8000"

  app-green:
    image: myapp:v1.1.0
    expose:
      - "8000"
```

```nginx
# nginx.conf
upstream app {
    # Переключение между blue и green
    server app-green:8000;  # Активный
    # server app-blue:8000;  # Резервный
}

server {
    listen 80;

    location / {
        proxy_pass http://app;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### Скрипт Blue-Green деплоя

```bash
#!/bin/bash
# blue-green-deploy.sh

set -e

NEW_IMAGE=$1
CURRENT_COLOR=$(docker inspect nginx --format '{{range .NetworkSettings.Networks}}{{.Aliases}}{{end}}' 2>/dev/null | grep -o 'blue\|green' || echo "blue")

if [ "$CURRENT_COLOR" == "blue" ]; then
    NEW_COLOR="green"
else
    NEW_COLOR="blue"
fi

echo "Current: $CURRENT_COLOR, Deploying to: $NEW_COLOR"

# Запускаем новую версию
docker compose up -d app-$NEW_COLOR

# Ждём готовности
echo "Waiting for health check..."
for i in {1..30}; do
    if docker exec app-$NEW_COLOR curl -sf http://localhost:8000/health > /dev/null 2>&1; then
        echo "New version is healthy"
        break
    fi
    sleep 2
done

# Переключаем Nginx
sed -i "s/app-$CURRENT_COLOR/app-$NEW_COLOR/g" nginx.conf
docker exec nginx nginx -s reload

echo "Traffic switched to $NEW_COLOR"

# Останавливаем старую версию через 30 секунд
sleep 30
docker compose stop app-$CURRENT_COLOR

echo "Blue-Green deployment complete!"
```

---

## Health Checks и Readiness Probes

### Docker Health Check

```dockerfile
# Dockerfile
FROM python:3.11-slim

# ... остальная конфигурация ...

HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

CMD ["python", "app.py"]
```

### Health Check в docker-compose

```yaml
version: "3.8"

services:
  app:
    image: myapp:latest
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
```

### Реализация Health Endpoint

#### Python (FastAPI)

```python
from fastapi import FastAPI, Response
from datetime import datetime
import asyncpg

app = FastAPI()

@app.get("/health")
async def health_check():
    """Liveness probe - приложение работает"""
    return {"status": "healthy", "timestamp": datetime.utcnow().isoformat()}

@app.get("/ready")
async def readiness_check(response: Response):
    """Readiness probe - приложение готово принимать трафик"""
    checks = {
        "database": await check_database(),
        "redis": await check_redis(),
        "external_api": await check_external_api(),
    }

    all_healthy = all(checks.values())

    if not all_healthy:
        response.status_code = 503

    return {
        "status": "ready" if all_healthy else "not_ready",
        "checks": checks,
        "timestamp": datetime.utcnow().isoformat()
    }

async def check_database():
    try:
        conn = await asyncpg.connect(DATABASE_URL)
        await conn.fetchval("SELECT 1")
        await conn.close()
        return True
    except Exception:
        return False

async def check_redis():
    try:
        redis = await aioredis.from_url(REDIS_URL)
        await redis.ping()
        await redis.close()
        return True
    except Exception:
        return False

async def check_external_api():
    try:
        async with aiohttp.ClientSession() as session:
            async with session.get("https://api.example.com/status", timeout=5) as resp:
                return resp.status == 200
    except Exception:
        return False
```

#### Node.js (Express)

```javascript
const express = require('express');
const { Pool } = require('pg');
const Redis = require('ioredis');

const app = express();

// Liveness probe
app.get('/health', (req, res) => {
    res.json({
        status: 'healthy',
        timestamp: new Date().toISOString()
    });
});

// Readiness probe
app.get('/ready', async (req, res) => {
    const checks = {};

    // Database check
    try {
        const pool = new Pool({ connectionString: process.env.DATABASE_URL });
        await pool.query('SELECT 1');
        await pool.end();
        checks.database = true;
    } catch (e) {
        checks.database = false;
    }

    // Redis check
    try {
        const redis = new Redis(process.env.REDIS_URL);
        await redis.ping();
        await redis.quit();
        checks.redis = true;
    } catch (e) {
        checks.redis = false;
    }

    const allHealthy = Object.values(checks).every(v => v);

    res.status(allHealthy ? 200 : 503).json({
        status: allHealthy ? 'ready' : 'not_ready',
        checks,
        timestamp: new Date().toISOString()
    });
});
```

---

## Мониторинг и логирование в продакшене

### Централизованное логирование

#### Конфигурация Docker для логирования

```yaml
# docker-compose.yml
version: "3.8"

services:
  app:
    image: myapp:latest
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
        labels: "service,environment"
```

#### ELK Stack (Elasticsearch, Logstash, Kibana)

```yaml
# docker-compose.logging.yml
version: "3.8"

services:
  elasticsearch:
    image: elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"

  logstash:
    image: logstash:8.11.0
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf
    ports:
      - "5044:5044"
    depends_on:
      - elasticsearch

  kibana:
    image: kibana:8.11.0
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    depends_on:
      - elasticsearch

  app:
    image: myapp:latest
    logging:
      driver: gelf
      options:
        gelf-address: "udp://localhost:12201"
        tag: "myapp"

volumes:
  elasticsearch_data:
```

### Мониторинг с Prometheus и Grafana

```yaml
# docker-compose.monitoring.yml
version: "3.8"

services:
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=15d'

  grafana:
    image: grafana/grafana:latest
    volumes:
      - grafana_data:/var/lib/grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    ports:
      - "8080:8080"

  node-exporter:
    image: prom/node-exporter:latest
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'
    ports:
      - "9100:9100"

volumes:
  prometheus_data:
  grafana_data:
```

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'app'
    static_configs:
      - targets: ['app:8000']
    metrics_path: /metrics
```

### Метрики приложения (Python + Prometheus)

```python
from prometheus_client import Counter, Histogram, generate_latest, CONTENT_TYPE_LATEST
from fastapi import FastAPI, Response
import time

app = FastAPI()

# Метрики
REQUEST_COUNT = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)

REQUEST_LATENCY = Histogram(
    'http_request_duration_seconds',
    'HTTP request latency',
    ['method', 'endpoint']
)

@app.middleware("http")
async def metrics_middleware(request, call_next):
    start_time = time.time()
    response = await call_next(request)
    duration = time.time() - start_time

    REQUEST_COUNT.labels(
        method=request.method,
        endpoint=request.url.path,
        status=response.status_code
    ).inc()

    REQUEST_LATENCY.labels(
        method=request.method,
        endpoint=request.url.path
    ).observe(duration)

    return response

@app.get("/metrics")
async def metrics():
    return Response(
        content=generate_latest(),
        media_type=CONTENT_TYPE_LATEST
    )
```

---

## Откат на предыдущую версию

### Docker Compose

```bash
# Сохраняем текущую версию
CURRENT_IMAGE=$(docker inspect myapp --format '{{.Config.Image}}')
echo $CURRENT_IMAGE > /backup/last_working_image.txt

# Деплоим новую версию
docker compose up -d

# Если что-то пошло не так - откат
PREVIOUS_IMAGE=$(cat /backup/last_working_image.txt)
docker compose down
docker compose up -d --no-deps -e IMAGE=$PREVIOUS_IMAGE app
```

### Docker Swarm

```bash
# Автоматический откат при сбое health check
docker service update \
    --update-failure-action rollback \
    --rollback-parallelism 1 \
    --rollback-delay 10s \
    myapp

# Ручной откат
docker service rollback myapp

# Просмотр истории
docker service ps myapp --no-trunc
```

### Kubernetes

```bash
# Откат на предыдущую версию
kubectl rollout undo deployment/myapp

# Откат на конкретную ревизию
kubectl rollout history deployment/myapp
kubectl rollout undo deployment/myapp --to-revision=2

# Проверка статуса
kubectl rollout status deployment/myapp
```

### Скрипт отката с валидацией

```bash
#!/bin/bash
# rollback.sh

set -e

CONTAINER_NAME="myapp"
HEALTH_ENDPOINT="http://localhost:8000/health"
MAX_ATTEMPTS=30
WAIT_SECONDS=2

# Получаем текущий образ
CURRENT_IMAGE=$(docker inspect $CONTAINER_NAME --format '{{.Config.Image}}')

# Получаем предыдущий образ из файла или аргумента
PREVIOUS_IMAGE=${1:-$(cat /var/backup/previous_image.txt)}

if [ -z "$PREVIOUS_IMAGE" ]; then
    echo "Error: No previous image specified"
    exit 1
fi

echo "Rolling back from $CURRENT_IMAGE to $PREVIOUS_IMAGE"

# Останавливаем текущий контейнер
docker stop $CONTAINER_NAME
docker rm $CONTAINER_NAME

# Запускаем предыдущую версию
docker run -d \
    --name $CONTAINER_NAME \
    --restart unless-stopped \
    -p 8000:8000 \
    --env-file /etc/myapp/.env \
    $PREVIOUS_IMAGE

# Проверяем здоровье
echo "Checking health..."
for i in $(seq 1 $MAX_ATTEMPTS); do
    if curl -sf $HEALTH_ENDPOINT > /dev/null; then
        echo "Rollback successful! Application is healthy."
        # Сохраняем текущий образ как "предыдущий" для следующего раза
        echo $CURRENT_IMAGE > /var/backup/previous_image.txt
        exit 0
    fi
    echo "Attempt $i/$MAX_ATTEMPTS - waiting..."
    sleep $WAIT_SECONDS
done

echo "Rollback failed! Application is not healthy."
exit 1
```

---

## Best Practices для продакшен-деплоя

### 1. Безопасность

```dockerfile
# Не запускайте контейнеры от root
RUN useradd -r -u 1001 appuser
USER appuser

# Используйте read-only файловую систему где возможно
# docker run --read-only --tmpfs /tmp myapp

# Сканируйте образы на уязвимости
# trivy image myapp:latest
```

### 2. Управление секретами

```yaml
# docker-compose.yml - НЕ храните секреты в файле!
services:
  app:
    image: myapp:latest
    secrets:
      - db_password
      - api_key

secrets:
  db_password:
    external: true  # Создан через docker secret create
  api_key:
    external: true
```

```bash
# Создание секретов
echo "my-password" | docker secret create db_password -

# Для Docker Compose без Swarm - используйте .env файл с ограниченными правами
chmod 600 .env
```

### 3. Ограничение ресурсов

```yaml
services:
  app:
    image: myapp:latest
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 1G
        reservations:
          cpus: '0.5'
          memory: 256M
    # Для обычного docker-compose (не Swarm)
    mem_limit: 1g
    cpus: 2
```

### 4. Graceful Shutdown

```python
# Python пример
import signal
import sys
import time

def graceful_shutdown(signum, frame):
    print("Received shutdown signal, finishing current requests...")
    # Завершаем текущие операции
    time.sleep(5)  # Ждём завершения запросов
    print("Shutdown complete")
    sys.exit(0)

signal.signal(signal.SIGTERM, graceful_shutdown)
signal.signal(signal.SIGINT, graceful_shutdown)
```

```dockerfile
# Dockerfile - используйте exec форму CMD
CMD ["python", "app.py"]  # Правильно - PID 1 получает сигналы

# НЕ используйте shell форму
# CMD python app.py  # Неправильно - shell перехватывает сигналы
```

### 5. Чек-лист перед деплоем

```markdown
## Чек-лист для production-деплоя

### Образ
- [ ] Используется multi-stage build
- [ ] Минимальный базовый образ (alpine, slim, distroless)
- [ ] Образ просканирован на уязвимости
- [ ] Используется конкретный тег (не latest)
- [ ] Размер образа оптимизирован

### Безопасность
- [ ] Контейнер не работает от root
- [ ] Секреты не хранятся в образе
- [ ] Используется read-only файловая система (где возможно)
- [ ] Ограничены capabilities

### Надёжность
- [ ] Настроен health check
- [ ] Настроен graceful shutdown
- [ ] Есть стратегия отката
- [ ] Настроены лимиты ресурсов
- [ ] Настроен restart policy

### Мониторинг
- [ ] Логи отправляются в централизованную систему
- [ ] Метрики экспортируются в Prometheus
- [ ] Настроены алерты

### CI/CD
- [ ] Автоматическая сборка при пуше
- [ ] Автоматические тесты
- [ ] Сканирование безопасности в pipeline
- [ ] Автоматический деплой в staging
- [ ] Ручное подтверждение для production
```

### 6. Пример полной production-ready конфигурации

```yaml
# docker-compose.prod.yml
version: "3.8"

services:
  app:
    image: ${REGISTRY}/${IMAGE_NAME}:${TAG}
    restart: unless-stopped
    read_only: true
    user: "1001:1001"
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    tmpfs:
      - /tmp
    ports:
      - "8000:8000"
    environment:
      - NODE_ENV=production
    secrets:
      - source: db_password
        target: /run/secrets/db_password
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 1G
        reservations:
          cpus: '0.5'
          memory: 256M
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    networks:
      - internal
      - external

  nginx:
    image: nginx:alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
    depends_on:
      app:
        condition: service_healthy
    networks:
      - external

networks:
  internal:
    internal: true
  external:

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

---

## Заключение

Деплой Docker-контейнеров в production требует внимания к множеству аспектов:

1. **Оптимизация образов** — используйте multi-stage builds и минимальные базовые образы
2. **CI/CD** — автоматизируйте сборку, тестирование и деплой
3. **Оркестрация** — выбирайте подходящий инструмент (Compose, Swarm, Kubernetes)
4. **Health checks** — всегда проверяйте готовность приложения
5. **Мониторинг** — собирайте логи и метрики
6. **Безопасность** — не запускайте от root, сканируйте образы
7. **Откат** — всегда имейте возможность быстро вернуться к предыдущей версии

Правильная настройка деплоя контейнеров обеспечивает надёжность, масштабируемость и безопасность вашего приложения в production-среде.
