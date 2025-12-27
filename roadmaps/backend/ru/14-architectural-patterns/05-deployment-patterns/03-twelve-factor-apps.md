# Twelve-Factor App (12-факторное приложение)

## Определение

**Twelve-Factor App** — это методология разработки SaaS-приложений, состоящая из 12 принципов, которые обеспечивают:
- Портативность между средами выполнения
- Горизонтальное масштабирование
- Непрерывное развёртывание (CI/CD)
- Минимизацию различий между development и production

Методология была сформулирована инженерами Heroku в 2011 году на основе опыта разработки и эксплуатации сотен приложений в облаке.

```
┌─────────────────────────────────────────────────────────────────┐
│                    12 ФАКТОРОВ                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│    ┌─────────────────────────────────────────────────────┐     │
│    │  1. Codebase          │  7. Port Binding            │     │
│    │  2. Dependencies      │  8. Concurrency             │     │
│    │  3. Config            │  9. Disposability           │     │
│    │  4. Backing Services  │ 10. Dev/Prod Parity         │     │
│    │  5. Build, Release,   │ 11. Logs                    │     │
│    │     Run               │ 12. Admin Processes         │     │
│    │  6. Processes         │                             │     │
│    └─────────────────────────────────────────────────────┘     │
│                                                                 │
│              Облако-ориентированная архитектура                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 12 факторов: детальный разбор

### Фактор 1: Codebase (Кодовая база)

**Принцип**: Один репозиторий — одно приложение. Множество деплоев из одной кодовой базы.

```
┌─────────────────────────────────────────────────────────────────┐
│                      CODEBASE                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│    ┌─────────────────────┐                                     │
│    │    Git Repository   │                                     │
│    │    (one codebase)   │                                     │
│    └──────────┬──────────┘                                     │
│               │                                                 │
│     ┌─────────┼─────────┐                                      │
│     │         │         │                                       │
│     ▼         ▼         ▼                                       │
│  ┌─────┐  ┌─────┐  ┌─────┐                                     │
│  │ Dev │  │Stage│  │Prod │                                     │
│  └─────┘  └─────┘  └─────┘                                     │
│                                                                 │
│  ✅ Правильно:                                                 │
│     1 repo = 1 app = N deploys                                 │
│                                                                 │
│  ❌ Неправильно:                                               │
│     N repos = 1 app (распределённая система)                   │
│     1 repo = N apps (монорепо с разными приложениями)          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Практика**:
```bash
# Правильная структура
my-app/
├── .git/
├── src/
├── tests/
├── Dockerfile
└── README.md

# Разные окружения - разные ветки или теги
git checkout main      # Production
git checkout develop   # Staging
git checkout feature/* # Development
```

### Фактор 2: Dependencies (Зависимости)

**Принцип**: Явное объявление и изоляция зависимостей. Приложение никогда не полагается на неявное существование системных пакетов.

```python
# ✅ Правильно: все зависимости в requirements.txt или pyproject.toml

# pyproject.toml
[project]
name = "my-app"
version = "1.0.0"
dependencies = [
    "fastapi>=0.100.0",
    "uvicorn>=0.23.0",
    "sqlalchemy>=2.0.0",
    "pydantic>=2.0.0",
    "redis>=4.5.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.4.0",
    "black>=23.7.0",
    "mypy>=1.5.0",
]
```

```dockerfile
# Dockerfile с явными зависимостями
FROM python:3.12-slim

WORKDIR /app

# Системные зависимости (явно объявлены)
RUN apt-get update && apt-get install -y \
    libpq-dev \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# Python зависимости
COPY pyproject.toml .
RUN pip install --no-cache-dir .

COPY . .

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Антипаттерны**:
```python
# ❌ Неправильно: зависимость от системных утилит без объявления
import subprocess
subprocess.run(["curl", "https://example.com"])  # curl может не быть установлен

# ❌ Неправильно: импорт без объявления в зависимостях
import numpy  # Если numpy нет в requirements.txt
```

### Фактор 3: Config (Конфигурация)

**Принцип**: Конфигурация хранится в переменных окружения. Никакой конфигурации в коде.

```python
# config.py - Правильный подход с pydantic-settings

from pydantic_settings import BaseSettings
from pydantic import Field, PostgresDsn, RedisDsn
from functools import lru_cache

class Settings(BaseSettings):
    """
    Конфигурация приложения через переменные окружения
    """
    # Application
    app_name: str = Field(default="MyApp", alias="APP_NAME")
    debug: bool = Field(default=False, alias="DEBUG")
    log_level: str = Field(default="INFO", alias="LOG_LEVEL")

    # Database
    database_url: PostgresDsn = Field(..., alias="DATABASE_URL")
    db_pool_size: int = Field(default=10, alias="DB_POOL_SIZE")
    db_pool_overflow: int = Field(default=20, alias="DB_POOL_OVERFLOW")

    # Redis
    redis_url: RedisDsn = Field(..., alias="REDIS_URL")
    redis_ttl: int = Field(default=3600, alias="REDIS_TTL")

    # External Services
    api_key: str = Field(..., alias="API_KEY")
    webhook_secret: str = Field(..., alias="WEBHOOK_SECRET")

    # Feature Flags
    feature_new_ui: bool = Field(default=False, alias="FEATURE_NEW_UI")
    feature_beta_api: bool = Field(default=False, alias="FEATURE_BETA_API")

    class Config:
        env_file = ".env"  # Только для локальной разработки
        env_file_encoding = "utf-8"
        case_sensitive = True

    def is_production(self) -> bool:
        return not self.debug

@lru_cache()
def get_settings() -> Settings:
    """Singleton для настроек"""
    return Settings()
```

```bash
# .env.example - шаблон для разработчиков
APP_NAME=MyApp
DEBUG=true
LOG_LEVEL=DEBUG

DATABASE_URL=postgresql://user:pass@localhost:5432/myapp
DB_POOL_SIZE=5

REDIS_URL=redis://localhost:6379/0

API_KEY=your-api-key-here
WEBHOOK_SECRET=your-secret-here

FEATURE_NEW_UI=false
FEATURE_BETA_API=true
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    КОНФИГУРАЦИЯ                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ Хранить в env:                 ❌ НЕ хранить в env:        │
│  ─────────────────                 ────────────────────         │
│  • DATABASE_URL                    • Маршруты (routes)          │
│  • API_KEY                         • Структуры данных           │
│  • SECRET_KEY                      • Бизнес-логика              │
│  • REDIS_URL                       • Константы приложения       │
│  • LOG_LEVEL                       • Порядок middleware         │
│  • Feature flags                                                │
│                                                                 │
│  Тест: Можно ли опубликовать код в open source                 │
│        без изменений? Если да — конфиг правильный.             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Фактор 4: Backing Services (Вспомогательные сервисы)

**Принцип**: Backing services (БД, очереди, кэш, email) — это подключаемые ресурсы, взаимозаменяемые без изменения кода.

```python
# services/database.py - Абстракция над backing service

from abc import ABC, abstractmethod
from typing import Any, Generic, TypeVar
from contextlib import contextmanager

T = TypeVar("T")

class DatabaseService(ABC):
    """Абстрактный интерфейс для базы данных"""

    @abstractmethod
    def connect(self) -> None:
        pass

    @abstractmethod
    def disconnect(self) -> None:
        pass

    @abstractmethod
    def execute(self, query: str, params: dict = None) -> Any:
        pass

class PostgresService(DatabaseService):
    """Реализация для PostgreSQL"""

    def __init__(self, database_url: str):
        self.database_url = database_url
        self.engine = None

    def connect(self) -> None:
        from sqlalchemy import create_engine
        self.engine = create_engine(self.database_url)

    def disconnect(self) -> None:
        if self.engine:
            self.engine.dispose()

    def execute(self, query: str, params: dict = None) -> Any:
        with self.engine.connect() as conn:
            return conn.execute(query, params)

class CacheService(ABC):
    """Абстрактный интерфейс для кэша"""

    @abstractmethod
    async def get(self, key: str) -> Any:
        pass

    @abstractmethod
    async def set(self, key: str, value: Any, ttl: int = 3600) -> None:
        pass

    @abstractmethod
    async def delete(self, key: str) -> None:
        pass

class RedisCache(CacheService):
    """Redis реализация кэша"""

    def __init__(self, redis_url: str):
        self.redis_url = redis_url
        self.client = None

    async def connect(self):
        import redis.asyncio as redis
        self.client = redis.from_url(self.redis_url)

    async def get(self, key: str) -> Any:
        import json
        value = await self.client.get(key)
        return json.loads(value) if value else None

    async def set(self, key: str, value: Any, ttl: int = 3600) -> None:
        import json
        await self.client.setex(key, ttl, json.dumps(value))

    async def delete(self, key: str) -> None:
        await self.client.delete(key)

# Dependency Injection
class ServiceContainer:
    """Контейнер для backing services"""

    def __init__(self, settings):
        self.settings = settings
        self._services = {}

    def database(self) -> DatabaseService:
        if "database" not in self._services:
            self._services["database"] = PostgresService(
                str(self.settings.database_url)
            )
        return self._services["database"]

    def cache(self) -> CacheService:
        if "cache" not in self._services:
            self._services["cache"] = RedisCache(
                str(self.settings.redis_url)
            )
        return self._services["cache"]
```

```
┌─────────────────────────────────────────────────────────────────┐
│                BACKING SERVICES                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                    ┌──────────────────┐                        │
│                    │   Application    │                        │
│                    └────────┬─────────┘                        │
│                             │                                   │
│    ┌────────────────────────┼────────────────────────┐         │
│    │                        │                        │         │
│    ▼                        ▼                        ▼         │
│ ┌──────────┐         ┌──────────┐         ┌──────────┐        │
│ │ Database │         │  Cache   │         │  Queue   │        │
│ │ (attach) │         │ (attach) │         │ (attach) │        │
│ └──────────┘         └──────────┘         └──────────┘        │
│                                                                 │
│ Легко заменить:                                                │
│ • Локальный PostgreSQL → AWS RDS                               │
│ • Redis local → ElastiCache                                    │
│ • RabbitMQ → Amazon SQS                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Фактор 5: Build, Release, Run (Сборка, релиз, запуск)

**Принцип**: Строгое разделение этапов: сборка → релиз (сборка + конфиг) → запуск.

```
┌─────────────────────────────────────────────────────────────────┐
│               BUILD → RELEASE → RUN                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────┐      ┌─────────┐      ┌─────────┐      ┌────────┐ │
│  │  Code   │ ──▶  │  Build  │ ──▶  │ Release │ ──▶  │  Run   │ │
│  │ (git)   │      │ (image) │      │ (deploy)│      │(process│ │
│  └─────────┘      └─────────┘      └─────────┘      └────────┘ │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ BUILD STAGE                                              │   │
│  │ • Компиляция кода                                        │   │
│  │ • Установка зависимостей                                 │   │
│  │ • Создание артефактов (Docker image, binary)             │   │
│  │ • Результат: неизменяемый build artifact                 │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ RELEASE STAGE                                            │   │
│  │ • Build artifact + Config (env vars)                     │   │
│  │ • Уникальный release ID (v1.2.3, git sha)               │   │
│  │ • Должен быть неизменяемым                              │   │
│  │ • Возможность rollback                                   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ RUN STAGE                                                │   │
│  │ • Запуск процессов из release                           │   │
│  │ • Управляется платформой (Kubernetes, Docker)            │   │
│  │ • Минимальные moving parts                              │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```yaml
# .github/workflows/build-release-run.yml

name: Build, Release, Run

on:
  push:
    tags: ['v*']

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ========================================
  # BUILD STAGE
  # ========================================
  build:
    runs-on: ubuntu-latest
    outputs:
      image_tag: ${{ steps.meta.outputs.version }}

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Container Registry
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
            type=semver,pattern={{version}}
            type=sha,prefix=

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # ========================================
  # RELEASE STAGE (per environment)
  # ========================================
  release-staging:
    needs: build
    runs-on: ubuntu-latest
    environment: staging

    steps:
      - name: Create Release Manifest
        run: |
          cat > release.yaml << EOF
          apiVersion: v1
          kind: Release
          metadata:
            name: myapp-${{ needs.build.outputs.image_tag }}
            environment: staging
          spec:
            image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ needs.build.outputs.image_tag }}
            config:
              DATABASE_URL: \${{ secrets.STAGING_DATABASE_URL }}
              REDIS_URL: \${{ secrets.STAGING_REDIS_URL }}
              LOG_LEVEL: DEBUG
          EOF

      - name: Deploy to Staging
        uses: azure/k8s-deploy@v4
        with:
          manifests: release.yaml
          namespace: staging

  release-production:
    needs: [build, release-staging]
    runs-on: ubuntu-latest
    environment: production

    steps:
      - name: Create Release Manifest
        run: |
          cat > release.yaml << EOF
          apiVersion: v1
          kind: Release
          metadata:
            name: myapp-${{ needs.build.outputs.image_tag }}
            environment: production
          spec:
            image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ needs.build.outputs.image_tag }}
            config:
              DATABASE_URL: \${{ secrets.PROD_DATABASE_URL }}
              REDIS_URL: \${{ secrets.PROD_REDIS_URL }}
              LOG_LEVEL: INFO
          EOF

      - name: Deploy to Production
        uses: azure/k8s-deploy@v4
        with:
          manifests: release.yaml
          namespace: production
```

### Фактор 6: Processes (Процессы)

**Принцип**: Приложение выполняется как один или несколько stateless процессов. Данные хранятся в backing services.

```python
# ❌ Неправильно: хранение состояния в памяти процесса

class BadSessionStore:
    """Сессии потеряются при перезапуске или масштабировании"""

    def __init__(self):
        self.sessions = {}  # Состояние в памяти!

    def set(self, session_id: str, data: dict):
        self.sessions[session_id] = data

    def get(self, session_id: str) -> dict:
        return self.sessions.get(session_id)


# ✅ Правильно: stateless процесс, состояние в Redis

class GoodSessionStore:
    """Сессии хранятся во внешнем сервисе"""

    def __init__(self, redis_client):
        self.redis = redis_client

    async def set(self, session_id: str, data: dict, ttl: int = 3600):
        await self.redis.setex(
            f"session:{session_id}",
            ttl,
            json.dumps(data)
        )

    async def get(self, session_id: str) -> dict | None:
        data = await self.redis.get(f"session:{session_id}")
        return json.loads(data) if data else None


# ❌ Неправильно: локальные файлы

class BadFileUploader:
    """Файлы недоступны другим инстансам"""

    def upload(self, file, filename: str):
        path = f"/tmp/uploads/{filename}"
        with open(path, "wb") as f:
            f.write(file.read())
        return path


# ✅ Правильно: файлы в object storage

class GoodFileUploader:
    """Файлы доступны всем инстансам"""

    def __init__(self, s3_client, bucket: str):
        self.s3 = s3_client
        self.bucket = bucket

    async def upload(self, file, filename: str) -> str:
        key = f"uploads/{filename}"
        await self.s3.put_object(
            Bucket=self.bucket,
            Key=key,
            Body=file.read()
        )
        return f"s3://{self.bucket}/{key}"
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    STATELESS PROCESSES                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Share-Nothing Architecture:                                   │
│                                                                 │
│   ┌─────────┐  ┌─────────┐  ┌─────────┐                        │
│   │Process 1│  │Process 2│  │Process 3│                        │
│   │(no state)│  │(no state)│  │(no state)│                       │
│   └────┬────┘  └────┬────┘  └────┬────┘                        │
│        │            │            │                              │
│        └────────────┼────────────┘                              │
│                     │                                           │
│                     ▼                                           │
│              ┌──────────────┐                                   │
│              │ Shared State │                                   │
│              │ (Redis, S3,  │                                   │
│              │  Database)   │                                   │
│              └──────────────┘                                   │
│                                                                 │
│   ✅ Любой процесс может обработать любой запрос               │
│   ✅ Процессы легко масштабировать горизонтально               │
│   ✅ Падение одного процесса не теряет данные                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Фактор 7: Port Binding (Привязка к порту)

**Принцип**: Приложение — полностью автономный сервис, экспортирующий HTTP через привязку к порту.

```python
# main.py - Приложение само является веб-сервером

import uvicorn
from fastapi import FastAPI
from config import get_settings

app = FastAPI()

@app.get("/health")
async def health():
    return {"status": "healthy"}

@app.get("/")
async def root():
    return {"message": "Hello, World!"}

if __name__ == "__main__":
    settings = get_settings()

    # Приложение само слушает порт
    uvicorn.run(
        "main:app",
        host="0.0.0.0",
        port=int(settings.port),  # Порт из env: PORT=8000
        workers=settings.workers,
        log_level=settings.log_level.lower()
    )
```

```dockerfile
# Dockerfile - Контейнер экспортирует порт

FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# Порт определяется переменной окружения
ENV PORT=8000
EXPOSE $PORT

CMD ["python", "main.py"]
```

```yaml
# kubernetes/deployment.yaml

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
      containers:
        - name: myapp
          image: myapp:latest
          env:
            - name: PORT
              value: "8000"
          ports:
            - containerPort: 8000
          livenessProbe:
            httpGet:
              path: /health
              port: 8000
          readinessProbe:
            httpGet:
              path: /health
              port: 8000
```

### Фактор 8: Concurrency (Параллелизм)

**Принцип**: Масштабирование через модель процессов. Разные типы работы — разные типы процессов.

```
┌─────────────────────────────────────────────────────────────────┐
│                    PROCESS MODEL                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Горизонтальное масштабирование по типам процессов:           │
│                                                                 │
│   ┌────────────────────────────────────────────────────────┐   │
│   │                                                        │   │
│   │  web (HTTP)     ████████████████  (8 процессов)       │   │
│   │  worker (BG)    ████████  (4 процесса)                │   │
│   │  scheduler      ██  (1 процесс)                        │   │
│   │  urgent-worker  ████  (2 процесса)                     │   │
│   │                                                        │   │
│   └────────────────────────────────────────────────────────┘   │
│                                                                 │
│   Каждый тип процесса масштабируется независимо                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```python
# Procfile - определение типов процессов

# Procfile
# web: uvicorn main:app --host 0.0.0.0 --port $PORT
# worker: celery -A tasks worker --loglevel=info
# scheduler: celery -A tasks beat --loglevel=info
# urgent-worker: celery -A tasks worker -Q urgent --loglevel=info
```

```yaml
# docker-compose.yml - Масштабирование типов процессов

version: "3.9"

services:
  web:
    build: .
    command: uvicorn main:app --host 0.0.0.0 --port 8000
    deploy:
      replicas: 4
    environment:
      - PORT=8000
      - DATABASE_URL=${DATABASE_URL}
    ports:
      - "8000-8003:8000"

  worker:
    build: .
    command: celery -A tasks worker --loglevel=info -Q default
    deploy:
      replicas: 2
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - CELERY_BROKER_URL=${REDIS_URL}

  urgent-worker:
    build: .
    command: celery -A tasks worker --loglevel=info -Q urgent
    deploy:
      replicas: 1
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - CELERY_BROKER_URL=${REDIS_URL}

  scheduler:
    build: .
    command: celery -A tasks beat --loglevel=info
    deploy:
      replicas: 1  # Только один scheduler!
    environment:
      - CELERY_BROKER_URL=${REDIS_URL}
```

### Фактор 9: Disposability (Утилизируемость)

**Принцип**: Процессы могут быть запущены или остановлены в любой момент. Быстрый запуск, корректное завершение.

```python
# graceful_shutdown.py - Корректное завершение

import signal
import asyncio
from contextlib import asynccontextmanager
from fastapi import FastAPI
import logging

logger = logging.getLogger(__name__)

class GracefulShutdown:
    """Управление корректным завершением процесса"""

    def __init__(self):
        self.shutdown_event = asyncio.Event()
        self.active_connections = 0
        self.active_tasks = set()

    def register_signals(self):
        """Регистрация обработчиков сигналов"""
        for sig in (signal.SIGTERM, signal.SIGINT):
            signal.signal(sig, self._handle_signal)

    def _handle_signal(self, signum, frame):
        """Обработчик сигнала завершения"""
        logger.info(f"Received signal {signum}, initiating graceful shutdown")
        self.shutdown_event.set()

    async def wait_for_shutdown(self, timeout: float = 30.0):
        """Ожидание завершения активных задач"""
        logger.info(f"Waiting for {len(self.active_tasks)} tasks to complete")

        try:
            # Ждём завершения всех задач с таймаутом
            if self.active_tasks:
                done, pending = await asyncio.wait(
                    self.active_tasks,
                    timeout=timeout
                )

                # Отменяем незавершённые задачи
                for task in pending:
                    logger.warning(f"Cancelling task: {task}")
                    task.cancel()

        except Exception as e:
            logger.error(f"Error during shutdown: {e}")

    @asynccontextmanager
    async def track_task(self, task_name: str):
        """Контекстный менеджер для отслеживания задач"""
        task = asyncio.current_task()
        self.active_tasks.add(task)
        logger.debug(f"Started task: {task_name}")

        try:
            yield
        finally:
            self.active_tasks.discard(task)
            logger.debug(f"Completed task: {task_name}")


shutdown_manager = GracefulShutdown()

@asynccontextmanager
async def lifespan(app: FastAPI):
    """FastAPI lifespan для startup/shutdown"""

    # STARTUP
    logger.info("Starting application...")
    shutdown_manager.register_signals()

    # Инициализация ресурсов
    await init_database()
    await init_cache()
    logger.info("Application started")

    yield

    # SHUTDOWN
    logger.info("Shutting down application...")

    # Ждём завершения активных задач
    await shutdown_manager.wait_for_shutdown()

    # Закрываем ресурсы
    await close_database()
    await close_cache()

    logger.info("Application stopped")

app = FastAPI(lifespan=lifespan)

@app.get("/long-task")
async def long_task():
    """Пример долгой задачи с корректным завершением"""
    async with shutdown_manager.track_task("long_task"):
        for i in range(10):
            if shutdown_manager.shutdown_event.is_set():
                logger.info("Task interrupted by shutdown")
                break
            await asyncio.sleep(1)

        return {"status": "completed"}
```

```yaml
# kubernetes/deployment.yaml - Настройки для graceful shutdown

apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 30  # Время на graceful shutdown
      containers:
        - name: app
          lifecycle:
            preStop:
              exec:
                # Дать время для drain connections
                command: ["sh", "-c", "sleep 5"]
```

### Фактор 10: Dev/Prod Parity (Паритет сред)

**Принцип**: Минимизация различий между development, staging и production.

```
┌─────────────────────────────────────────────────────────────────┐
│                    DEV/PROD PARITY                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │             Традиционный подход (плохо)                 │   │
│  │                                                         │   │
│  │  Gap          Dev              Prod                     │   │
│  │  ────         ───              ────                     │   │
│  │  Time         Weeks            Instant                  │   │
│  │  Personnel    Developer        Ops team                 │   │
│  │  Tools        SQLite           PostgreSQL               │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │             12-Factor подход (хорошо)                   │   │
│  │                                                         │   │
│  │  Gap          Dev              Prod                     │   │
│  │  ────         ───              ────                     │   │
│  │  Time         Hours            Instant                  │   │
│  │  Personnel    Same developer   Same developer           │   │
│  │  Tools        PostgreSQL       PostgreSQL               │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```yaml
# docker-compose.yml - Локальная среда идентична production

version: "3.9"

services:
  app:
    build: .
    environment:
      - DATABASE_URL=postgresql://user:pass@postgres:5432/myapp
      - REDIS_URL=redis://redis:6379/0
    depends_on:
      - postgres
      - redis

  # Те же backing services, что и в production
  postgres:
    image: postgres:16  # Та же версия, что в production
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: myapp
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7  # Та же версия

  # Имитация production инфраструктуры
  nginx:
    image: nginx:latest
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - app

volumes:
  postgres_data:
```

```python
# Избегаем различий в коде

# ❌ Неправильно: разный код для разных сред
if settings.environment == "development":
    from sqlite3 import connect
    db = connect("dev.db")
else:
    import psycopg2
    db = psycopg2.connect(settings.database_url)

# ✅ Правильно: одинаковый код, разная конфигурация
from sqlalchemy import create_engine
engine = create_engine(settings.database_url)  # postgresql:// везде
```

### Фактор 11: Logs (Логи)

**Принцип**: Логи — это поток событий. Приложение пишет в stdout, платформа собирает и обрабатывает.

```python
# logging_config.py - Структурированное логирование

import logging
import sys
import json
from datetime import datetime
from typing import Any

class JSONFormatter(logging.Formatter):
    """Форматтер для структурированных JSON логов"""

    def format(self, record: logging.LogRecord) -> str:
        log_entry = {
            "timestamp": datetime.utcnow().isoformat() + "Z",
            "level": record.levelname,
            "logger": record.name,
            "message": record.getMessage(),
            "module": record.module,
            "function": record.funcName,
            "line": record.lineno,
        }

        # Добавляем extra поля
        if hasattr(record, "request_id"):
            log_entry["request_id"] = record.request_id
        if hasattr(record, "user_id"):
            log_entry["user_id"] = record.user_id
        if hasattr(record, "duration_ms"):
            log_entry["duration_ms"] = record.duration_ms

        # Добавляем исключение, если есть
        if record.exc_info:
            log_entry["exception"] = self.formatException(record.exc_info)

        return json.dumps(log_entry)


def setup_logging(log_level: str = "INFO"):
    """Настройка логирования для 12-factor app"""

    # Корневой логгер
    root_logger = logging.getLogger()
    root_logger.setLevel(getattr(logging, log_level.upper()))

    # Handler для stdout (единственный!)
    handler = logging.StreamHandler(sys.stdout)
    handler.setFormatter(JSONFormatter())
    root_logger.addHandler(handler)

    # Отключаем логи сторонних библиотек
    logging.getLogger("uvicorn").setLevel(logging.WARNING)
    logging.getLogger("sqlalchemy").setLevel(logging.WARNING)


# middleware/logging.py - Request logging

from starlette.middleware.base import BaseHTTPMiddleware
import time
import uuid

class RequestLoggingMiddleware(BaseHTTPMiddleware):
    """Middleware для логирования всех запросов"""

    async def dispatch(self, request, call_next):
        request_id = request.headers.get("X-Request-ID", str(uuid.uuid4()))
        start_time = time.time()

        # Добавляем request_id в контекст
        logger = logging.getLogger("api")

        # Логируем начало запроса
        logger.info(
            f"Request started: {request.method} {request.url.path}",
            extra={
                "request_id": request_id,
                "method": request.method,
                "path": request.url.path,
                "client_ip": request.client.host,
            }
        )

        try:
            response = await call_next(request)
            duration_ms = (time.time() - start_time) * 1000

            # Логируем завершение
            logger.info(
                f"Request completed: {response.status_code}",
                extra={
                    "request_id": request_id,
                    "status_code": response.status_code,
                    "duration_ms": round(duration_ms, 2),
                }
            )

            response.headers["X-Request-ID"] = request_id
            return response

        except Exception as e:
            duration_ms = (time.time() - start_time) * 1000
            logger.exception(
                f"Request failed: {str(e)}",
                extra={
                    "request_id": request_id,
                    "duration_ms": round(duration_ms, 2),
                }
            )
            raise
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    LOG AGGREGATION                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐                         │
│  │ App 1   │  │ App 2   │  │ App 3   │                         │
│  │ stdout  │  │ stdout  │  │ stdout  │                         │
│  └────┬────┘  └────┬────┘  └────┬────┘                         │
│       │            │            │                               │
│       └────────────┼────────────┘                               │
│                    │                                            │
│                    ▼                                            │
│           ┌────────────────┐                                    │
│           │  Log Collector │                                    │
│           │  (Fluentd,     │                                    │
│           │   Filebeat)    │                                    │
│           └───────┬────────┘                                    │
│                   │                                             │
│                   ▼                                             │
│           ┌────────────────┐                                    │
│           │  Log Storage   │                                    │
│           │ (Elasticsearch,│                                    │
│           │  Loki, etc.)   │                                    │
│           └───────┬────────┘                                    │
│                   │                                             │
│                   ▼                                             │
│           ┌────────────────┐                                    │
│           │  Visualization │                                    │
│           │  (Kibana,      │                                    │
│           │   Grafana)     │                                    │
│           └────────────────┘                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Фактор 12: Admin Processes (Административные процессы)

**Принцип**: Административные задачи (миграции, консоль) запускаются как одноразовые процессы в идентичном окружении.

```python
# cli.py - Административные команды

import click
import asyncio
from app.database import engine, get_session
from app.models import User
from app.config import get_settings

@click.group()
def cli():
    """Административные команды приложения"""
    pass

@cli.command()
def migrate():
    """Выполнить миграции базы данных"""
    from alembic.config import Config
    from alembic import command

    click.echo("Running database migrations...")
    alembic_cfg = Config("alembic.ini")
    command.upgrade(alembic_cfg, "head")
    click.echo("Migrations completed successfully")

@cli.command()
@click.argument("email")
@click.option("--admin", is_flag=True, help="Create admin user")
def create_user(email: str, admin: bool):
    """Создать нового пользователя"""
    async def _create():
        async with get_session() as session:
            user = User(email=email, is_admin=admin)
            session.add(user)
            await session.commit()
            return user

    user = asyncio.run(_create())
    click.echo(f"Created user: {user.id} ({user.email})")

@cli.command()
def shell():
    """Интерактивная консоль с контекстом приложения"""
    import IPython
    from app.models import User, Order, Product
    from app.services import UserService, OrderService

    # Инициализируем контекст
    settings = get_settings()

    # Добавляем полезные объекты в namespace
    namespace = {
        "settings": settings,
        "User": User,
        "Order": Order,
        "Product": Product,
        "UserService": UserService,
        "OrderService": OrderService,
    }

    click.echo("Starting interactive shell...")
    click.echo("Available objects: settings, User, Order, Product, UserService, OrderService")
    IPython.start_ipython(argv=[], user_ns=namespace)

@cli.command()
@click.option("--dry-run", is_flag=True, help="Show what would be done")
def cleanup_old_data(dry_run: bool):
    """Удалить устаревшие данные"""
    async def _cleanup():
        async with get_session() as session:
            # Найти устаревшие записи
            old_records = await session.execute(
                "SELECT id FROM logs WHERE created_at < NOW() - INTERVAL '90 days'"
            )
            count = len(old_records.all())

            if dry_run:
                click.echo(f"Would delete {count} old log records")
            else:
                await session.execute(
                    "DELETE FROM logs WHERE created_at < NOW() - INTERVAL '90 days'"
                )
                await session.commit()
                click.echo(f"Deleted {count} old log records")

    asyncio.run(_cleanup())

if __name__ == "__main__":
    cli()
```

```bash
# Запуск административных команд

# Локально
python cli.py migrate
python cli.py create-user admin@example.com --admin
python cli.py shell

# В Kubernetes - как Job
kubectl run --rm -it migration \
  --image=myapp:latest \
  --restart=Never \
  -- python cli.py migrate

# В Docker
docker run --rm \
  --env-file .env \
  myapp:latest \
  python cli.py migrate
```

```yaml
# kubernetes/migration-job.yaml

apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  template:
    spec:
      containers:
        - name: migration
          image: myapp:latest
          command: ["python", "cli.py", "migrate"]
          envFrom:
            - secretRef:
                name: app-secrets
            - configMapRef:
                name: app-config
      restartPolicy: Never
  backoffLimit: 3
```

## Когда использовать

### Идеальные сценарии

1. **SaaS приложения** — методология создана для них
2. **Микросервисы** — каждый сервис следует 12 факторам
3. **Облачные развёртывания** — AWS, GCP, Azure, Heroku
4. **Контейнеризация** — Docker, Kubernetes
5. **CI/CD пайплайны** — автоматизация сборки и развёртывания

### Когда адаптировать

- **Legacy системы** — постепенное внедрение принципов
- **Монолиты** — применимо, но требует адаптации
- **Embedded системы** — некоторые факторы неприменимы
- **Batch processing** — требует модификации подхода

## Best practices и антипаттерны

```
┌─────────────────────────────────────────────────────────────────┐
│                      ЧЕКЛИСТ 12-FACTOR                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ Один репозиторий = одно приложение                         │
│  ✅ Все зависимости объявлены явно (requirements.txt)          │
│  ✅ Конфигурация через переменные окружения                    │
│  ✅ Backing services подключаются через URL                    │
│  ✅ Строгое разделение build/release/run                       │
│  ✅ Процессы stateless, состояние в backing services           │
│  ✅ Приложение экспортирует HTTP через порт                    │
│  ✅ Масштабирование через добавление процессов                 │
│  ✅ Быстрый запуск и graceful shutdown                         │
│  ✅ Dev/staging/prod максимально похожи                        │
│  ✅ Логи в stdout как поток событий                            │
│  ✅ Admin-задачи как одноразовые процессы                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Антипаттерны

| Антипаттерн | Нарушенный фактор | Решение |
|------------|-------------------|---------|
| Хардкод конфигурации | 3. Config | Env variables |
| Локальное хранение сессий | 6. Processes | Redis/DB |
| Разные БД в dev/prod | 10. Dev/Prod Parity | Docker Compose |
| Логи в файлы | 11. Logs | stdout + агрегатор |
| Миграции при старте app | 12. Admin | Отдельный job |
| Неявные зависимости | 2. Dependencies | requirements.txt |

## Связанные паттерны

```
┌─────────────────────────────────────────────────────────────────┐
│                   СВЯЗАННЫЕ КОНЦЕПЦИИ                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────┐     ┌─────────────────┐                   │
│  │ Cloud Native    │     │ Microservices   │                   │
│  └────────┬────────┘     └────────┬────────┘                   │
│           │                       │                             │
│           ▼                       ▼                             │
│  CNCF практики и            Архитектура                        │
│  технологии                 независимых сервисов               │
│                                                                 │
│  ┌─────────────────┐     ┌─────────────────┐                   │
│  │ DevOps          │     │ GitOps          │                   │
│  └────────┬────────┘     └────────┬────────┘                   │
│           │                       │                             │
│           ▼                       ▼                             │
│  Культура и практики       Декларативная                       │
│  автоматизации             инфраструктура                      │
│                                                                 │
│  ┌─────────────────┐     ┌─────────────────┐                   │
│  │ Containers      │     │ Serverless      │                   │
│  └────────┬────────┘     └────────┬────────┘                   │
│           │                       │                             │
│           ▼                       ▼                             │
│  Docker, Kubernetes        FaaS платформы                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Beyond Twelve-Factor

Современные дополнения к оригинальным 12 факторам:

| Фактор | Описание |
|--------|----------|
| **API First** | Проектирование начинается с API |
| **Telemetry** | Metrics, tracing, logging |
| **Security** | Security by design, zero trust |
| **Resilience** | Circuit breakers, retries, timeouts |

## Ресурсы для изучения

### Официальные ресурсы
- [12factor.net](https://12factor.net/) — оригинальный манифест
- [12factor.net/ru](https://12factor.net/ru/) — русский перевод

### Книги
- **"Cloud Native Patterns"** — Cornelia Davis
- **"Production-Ready Microservices"** — Susan Fowler
- **"Building Microservices"** — Sam Newman

### Статьи
- [Beyond the Twelve-Factor App](https://tanzu.vmware.com/content/blog/beyond-the-twelve-factor-app) — Kevin Hoffman
- [The Twelve-Factor App methodology revisited](https://architecturenotes.co/12-factor-app-revisited/)

### Инструменты
- **Docker** — контейнеризация
- **Kubernetes** — оркестрация
- **Terraform** — Infrastructure as Code
- **GitHub Actions / GitLab CI** — CI/CD
- **Prometheus + Grafana** — мониторинг
- **ELK / Loki** — логирование
