# Strangler Pattern (Паттерн "Удушитель")

## Определение

**Strangler Pattern** (паттерн "Удушитель" или "Удавка") — это архитектурный паттерн постепенной миграции legacy-системы на новую архитектуру путём инкрементальной замены отдельных компонентов. Название происходит от Strangler Fig (фикус-душитель) — растения, которое обвивает дерево-хозяина и постепенно замещает его.

Паттерн был впервые описан Мартином Фаулером в 2004 году и стал стандартом де-факто для миграции монолитных приложений на микросервисную архитектуру.

```
┌─────────────────────────────────────────────────────────────────┐
│                    ЭВОЛЮЦИЯ STRANGLER PATTERN                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Этап 1: Начало          Этап 2: Миграция      Этап 3: Финал   │
│  ┌─────────────┐         ┌─────────────┐       ┌─────────────┐ │
│  │   Legacy    │         │ ┌─────────┐ │       │             │ │
│  │   System    │    →    │ │ New     │ │   →   │ New System  │ │
│  │             │         │ │ System  │ │       │             │ │
│  │ ┌─────────┐ │         │ └────┬────┘ │       │ ┌─────────┐ │ │
│  │ │ Module  │ │         │      ↓      │       │ │ Service │ │ │
│  │ │   A     │ │         │ ┌────┴────┐ │       │ │    A    │ │ │
│  │ ├─────────┤ │         │ │ Legacy  │ │       │ ├─────────┤ │ │
│  │ │ Module  │ │         │ │ System  │ │       │ │ Service │ │ │
│  │ │   B     │ │         │ │(частич.)│ │       │ │    B    │ │ │
│  │ ├─────────┤ │         │ └─────────┘ │       │ ├─────────┤ │ │
│  │ │ Module  │ │         │             │       │ │ Service │ │ │
│  │ │   C     │ │         │             │       │ │    C    │ │ │
│  │ └─────────┘ │         │             │       │ └─────────┘ │ │
│  └─────────────┘         └─────────────┘       └─────────────┘ │
│                                                                 │
│  100% Legacy             50% Legacy             0% Legacy       │
│                          50% New               100% New         │
└─────────────────────────────────────────────────────────────────┘
```

## Ключевые характеристики

### 1. Инкрементальная миграция

Система мигрирует по частям, а не целиком ("Big Bang" подход). Это снижает риски и позволяет получать обратную связь на каждом этапе.

### 2. Фасадный слой (Strangler Facade)

Прокси или API Gateway, который маршрутизирует запросы между старой и новой системами:

```
┌───────────────────────────────────────────────────────────────┐
│                     STRANGLER FACADE                          │
│                                                               │
│    ┌─────────┐                                                │
│    │ Client  │                                                │
│    └────┬────┘                                                │
│         │                                                     │
│         ▼                                                     │
│    ┌─────────────────────────────────────────┐               │
│    │           API Gateway / Proxy           │               │
│    │         (Strangler Facade)              │               │
│    │                                         │               │
│    │  ┌─────────────────────────────────┐   │               │
│    │  │     Routing Rules               │   │               │
│    │  │  /api/users/*  → New Service    │   │               │
│    │  │  /api/orders/* → New Service    │   │               │
│    │  │  /api/legacy/* → Legacy App     │   │               │
│    │  │  /api/*        → Legacy App     │   │               │
│    │  └─────────────────────────────────┘   │               │
│    └──────────────┬──────────────┬──────────┘               │
│                   │              │                           │
│         ┌─────────┴───┐    ┌─────┴─────────┐                │
│         ▼             │    ▼               │                │
│    ┌─────────┐        │  ┌─────────────┐   │                │
│    │  New    │        │  │   Legacy    │   │                │
│    │ Services│        │  │   System    │   │                │
│    └─────────┘        │  └─────────────┘   │                │
│                       │                     │                │
└───────────────────────┴─────────────────────┘                │
```

### 3. Coexistence (Сосуществование)

Старая и новая системы работают параллельно, обеспечивая непрерывность бизнес-процессов.

### 4. Feature Toggles

Использование флагов для переключения трафика между системами:

```python
# Пример использования feature toggles для маршрутизации
class FeatureToggleRouter:
    def __init__(self):
        self.toggles = {
            'users_service': True,      # Мигрирован
            'orders_service': True,     # Мигрирован
            'payments_service': False,  # Ещё в legacy
            'reports_service': False,   # Ещё в legacy
        }

    def route_request(self, service_name: str, request):
        if self.toggles.get(service_name, False):
            return self.route_to_new_service(service_name, request)
        else:
            return self.route_to_legacy(request)

    def route_to_new_service(self, service_name: str, request):
        """Направить запрос в новый микросервис"""
        service_url = self.get_service_url(service_name)
        return requests.forward(service_url, request)

    def route_to_legacy(self, request):
        """Направить запрос в legacy-систему"""
        return requests.forward(LEGACY_URL, request)
```

## Когда использовать

### Идеальные сценарии применения

1. **Миграция монолита на микросервисы**
   - Большая legacy-система с множеством модулей
   - Невозможность остановить бизнес на время миграции

2. **Модернизация устаревшего стека**
   - Переход с устаревших технологий (COBOL, старые версии Java)
   - Обновление фреймворков (Legacy Django → FastAPI)

3. **Рефакторинг при активной разработке**
   - Команда продолжает добавлять фичи
   - Нет возможности заморозить разработку

4. **Снижение технического долга**
   - Постепенная замена проблемных компонентов
   - Улучшение архитектуры по частям

### Когда НЕ использовать

```
┌─────────────────────────────────────────────────────────────────┐
│                    ПРОТИВОПОКАЗАНИЯ                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ Маленькая система                                          │
│     → Проще переписать с нуля                                  │
│                                                                 │
│  ❌ Тесно связанные компоненты                                 │
│     → Невозможно выделить модули                               │
│                                                                 │
│  ❌ Отсутствие чётких границ                                   │
│     → Нужен предварительный рефакторинг                        │
│                                                                 │
│  ❌ Критически синхронные транзакции                           │
│     → Сложно обеспечить консистентность                        │
│                                                                 │
│  ❌ Отсутствие тестов                                          │
│     → Высокий риск регрессий                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Преимущества и недостатки

### Преимущества

| Преимущество | Описание |
|-------------|----------|
| **Низкий риск** | Постепенная миграция позволяет откатить изменения |
| **Непрерывность бизнеса** | Система работает во время миграции |
| **Быстрая обратная связь** | Проблемы выявляются на ранних этапах |
| **Гибкость** | Можно менять приоритеты миграции |
| **Обучение команды** | Постепенное освоение новых технологий |
| **Измеримый прогресс** | Легко отслеживать процент миграции |

### Недостатки

| Недостаток | Описание |
|-----------|----------|
| **Сложность инфраструктуры** | Нужно поддерживать две системы |
| **Overhead маршрутизации** | Дополнительная задержка через facade |
| **Дублирование данных** | Часто требуется синхронизация БД |
| **Длительный процесс** | Миграция может занять годы |
| **Стоимость** | Двойные затраты на поддержку |
| **Сложность тестирования** | Нужно тестировать обе системы |

## Примеры реализации

### Пример 1: Nginx как Strangler Facade

```nginx
# nginx.conf - Конфигурация для Strangler Pattern

upstream legacy_backend {
    server legacy-app:8080;
}

upstream users_service {
    server users-service:3000;
}

upstream orders_service {
    server orders-service:3001;
}

upstream payments_service {
    server payments-service:3002;
}

server {
    listen 80;
    server_name api.example.com;

    # ===============================================
    # Мигрированные сервисы - направляем в новые
    # ===============================================

    # Users service - полностью мигрирован
    location /api/v2/users {
        proxy_pass http://users_service;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Request-ID $request_id;

        # Логирование для мониторинга миграции
        access_log /var/log/nginx/users_service.log;
    }

    # Orders service - полностью мигрирован
    location /api/v2/orders {
        proxy_pass http://orders_service;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # Payments service - частично мигрирован (A/B тестирование)
    location /api/v2/payments {
        # 20% трафика на новый сервис
        split_clients $request_id $payments_backend {
            20%     payments_service;
            *       legacy_backend;
        }

        proxy_pass http://$payments_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Migration-Target $payments_backend;
    }

    # ===============================================
    # Всё остальное - в legacy
    # ===============================================

    location / {
        proxy_pass http://legacy_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### Пример 2: Python FastAPI Gateway

```python
# strangler_gateway.py
from fastapi import FastAPI, Request, HTTPException
from fastapi.responses import StreamingResponse
import httpx
from dataclasses import dataclass
from enum import Enum
from typing import Optional
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class MigrationStatus(Enum):
    LEGACY = "legacy"
    MIGRATING = "migrating"  # A/B тестирование
    MIGRATED = "migrated"

@dataclass
class ServiceConfig:
    name: str
    status: MigrationStatus
    new_url: Optional[str] = None
    legacy_url: str = "http://legacy-app:8080"
    traffic_percentage: int = 0  # % трафика на новый сервис

# Конфигурация миграции
MIGRATION_CONFIG = {
    "/api/users": ServiceConfig(
        name="users",
        status=MigrationStatus.MIGRATED,
        new_url="http://users-service:3000"
    ),
    "/api/orders": ServiceConfig(
        name="orders",
        status=MigrationStatus.MIGRATING,
        new_url="http://orders-service:3001",
        traffic_percentage=50
    ),
    "/api/payments": ServiceConfig(
        name="payments",
        status=MigrationStatus.LEGACY
    ),
    "/api/reports": ServiceConfig(
        name="reports",
        status=MigrationStatus.LEGACY
    ),
}

app = FastAPI(title="Strangler Gateway")
client = httpx.AsyncClient(timeout=30.0)

def get_service_config(path: str) -> ServiceConfig:
    """Найти конфигурацию сервиса по пути"""
    for prefix, config in MIGRATION_CONFIG.items():
        if path.startswith(prefix):
            return config
    # По умолчанию - legacy
    return ServiceConfig(
        name="default",
        status=MigrationStatus.LEGACY
    )

def should_use_new_service(config: ServiceConfig, request_id: str) -> bool:
    """Определить, использовать ли новый сервис"""
    if config.status == MigrationStatus.MIGRATED:
        return True
    if config.status == MigrationStatus.LEGACY:
        return False

    # Для MIGRATING - используем hash request_id для consistent routing
    hash_value = hash(request_id) % 100
    return hash_value < config.traffic_percentage

@app.middleware("http")
async def strangler_middleware(request: Request, call_next):
    """Middleware для маршрутизации между legacy и новыми сервисами"""
    path = request.url.path
    request_id = request.headers.get("X-Request-ID", str(id(request)))

    config = get_service_config(path)
    use_new = should_use_new_service(config, request_id)

    # Определяем целевой URL
    if use_new and config.new_url:
        target_url = f"{config.new_url}{path}"
        target_system = "new"
    else:
        target_url = f"{config.legacy_url}{path}"
        target_system = "legacy"

    # Логируем решение маршрутизации
    logger.info(
        f"Routing {path} to {target_system} "
        f"(service: {config.name}, status: {config.status.value})"
    )

    # Проксируем запрос
    try:
        # Собираем headers
        headers = dict(request.headers)
        headers["X-Forwarded-For"] = request.client.host
        headers["X-Original-Path"] = path
        headers["X-Target-System"] = target_system

        # Выполняем запрос
        body = await request.body()
        response = await client.request(
            method=request.method,
            url=target_url,
            headers=headers,
            content=body,
            params=dict(request.query_params)
        )

        # Возвращаем ответ с метаданными
        return StreamingResponse(
            iter([response.content]),
            status_code=response.status_code,
            headers={
                **dict(response.headers),
                "X-Routed-To": target_system,
                "X-Service": config.name
            }
        )

    except httpx.RequestError as e:
        logger.error(f"Error proxying to {target_url}: {e}")

        # Fallback на legacy при ошибке нового сервиса
        if use_new:
            logger.warning(f"Falling back to legacy for {path}")
            return await proxy_to_legacy(request, config, path)

        raise HTTPException(status_code=503, detail="Service unavailable")

async def proxy_to_legacy(request: Request, config: ServiceConfig, path: str):
    """Fallback на legacy-систему"""
    target_url = f"{config.legacy_url}{path}"
    body = await request.body()

    response = await client.request(
        method=request.method,
        url=target_url,
        headers=dict(request.headers),
        content=body,
        params=dict(request.query_params)
    )

    return StreamingResponse(
        iter([response.content]),
        status_code=response.status_code,
        headers={
            **dict(response.headers),
            "X-Routed-To": "legacy-fallback"
        }
    )

# Эндпоинты для мониторинга миграции
@app.get("/migration/status")
async def migration_status():
    """Получить статус миграции всех сервисов"""
    return {
        service_path: {
            "name": config.name,
            "status": config.status.value,
            "traffic_to_new": config.traffic_percentage
                if config.status == MigrationStatus.MIGRATING else
                (100 if config.status == MigrationStatus.MIGRATED else 0)
        }
        for service_path, config in MIGRATION_CONFIG.items()
    }

@app.post("/migration/update/{service_name}")
async def update_migration(service_name: str, traffic_percentage: int):
    """Обновить процент трафика для сервиса"""
    for path, config in MIGRATION_CONFIG.items():
        if config.name == service_name:
            config.traffic_percentage = min(100, max(0, traffic_percentage))
            if traffic_percentage >= 100:
                config.status = MigrationStatus.MIGRATED
            elif traffic_percentage > 0:
                config.status = MigrationStatus.MIGRATING
            else:
                config.status = MigrationStatus.LEGACY
            return {"message": f"Updated {service_name} to {traffic_percentage}%"}

    raise HTTPException(status_code=404, detail="Service not found")
```

### Пример 3: Синхронизация данных между системами

```python
# data_sync.py - Синхронизация данных между legacy и новой системой

import asyncio
from datetime import datetime
from typing import Dict, Any, List
from dataclasses import dataclass
import json
import logging

logger = logging.getLogger(__name__)

@dataclass
class SyncEvent:
    """Событие синхронизации"""
    entity_type: str
    entity_id: str
    operation: str  # create, update, delete
    data: Dict[str, Any]
    timestamp: datetime
    source: str  # legacy или new

class DataSynchronizer:
    """
    Компонент для двунаправленной синхронизации данных
    между legacy и новой системой во время миграции
    """

    def __init__(self, legacy_db, new_db, event_bus):
        self.legacy_db = legacy_db
        self.new_db = new_db
        self.event_bus = event_bus

        # Маппинг полей между системами
        self.field_mappings = {
            'users': {
                'legacy_to_new': {
                    'user_id': 'id',
                    'user_name': 'username',
                    'user_email': 'email',
                    'created_date': 'created_at',
                    'status_flag': 'is_active'
                },
                'new_to_legacy': {
                    'id': 'user_id',
                    'username': 'user_name',
                    'email': 'user_email',
                    'created_at': 'created_date',
                    'is_active': 'status_flag'
                }
            }
        }

    def transform_legacy_to_new(self, entity_type: str, data: Dict) -> Dict:
        """Преобразование данных из legacy формата в новый"""
        mapping = self.field_mappings.get(entity_type, {}).get('legacy_to_new', {})

        transformed = {}
        for old_key, new_key in mapping.items():
            if old_key in data:
                value = data[old_key]
                # Преобразование типов при необходимости
                if new_key == 'is_active' and isinstance(value, int):
                    value = bool(value)
                if new_key == 'created_at' and isinstance(value, str):
                    value = datetime.fromisoformat(value)
                transformed[new_key] = value

        return transformed

    def transform_new_to_legacy(self, entity_type: str, data: Dict) -> Dict:
        """Преобразование данных из нового формата в legacy"""
        mapping = self.field_mappings.get(entity_type, {}).get('new_to_legacy', {})

        transformed = {}
        for new_key, old_key in mapping.items():
            if new_key in data:
                value = data[new_key]
                # Обратное преобразование типов
                if old_key == 'status_flag' and isinstance(value, bool):
                    value = 1 if value else 0
                if old_key == 'created_date' and isinstance(value, datetime):
                    value = value.isoformat()
                transformed[old_key] = value

        return transformed

    async def sync_from_legacy(self, event: SyncEvent):
        """Синхронизация изменений из legacy в новую систему"""
        try:
            # Проверяем, не является ли это echo нашего же изменения
            if await self.is_echo_event(event):
                logger.debug(f"Skipping echo event for {event.entity_type}:{event.entity_id}")
                return

            transformed_data = self.transform_legacy_to_new(
                event.entity_type,
                event.data
            )

            if event.operation == 'create':
                await self.new_db.insert(event.entity_type, transformed_data)
            elif event.operation == 'update':
                await self.new_db.update(
                    event.entity_type,
                    event.entity_id,
                    transformed_data
                )
            elif event.operation == 'delete':
                await self.new_db.delete(event.entity_type, event.entity_id)

            # Помечаем синхронизацию как выполненную
            await self.mark_synced(event, 'legacy_to_new')

            logger.info(
                f"Synced {event.operation} {event.entity_type}:{event.entity_id} "
                f"from legacy to new"
            )

        except Exception as e:
            logger.error(f"Failed to sync from legacy: {e}")
            await self.handle_sync_error(event, 'legacy_to_new', str(e))

    async def sync_from_new(self, event: SyncEvent):
        """Синхронизация изменений из новой системы в legacy"""
        try:
            if await self.is_echo_event(event):
                return

            transformed_data = self.transform_new_to_legacy(
                event.entity_type,
                event.data
            )

            if event.operation == 'create':
                await self.legacy_db.insert(event.entity_type, transformed_data)
            elif event.operation == 'update':
                await self.legacy_db.update(
                    event.entity_type,
                    event.entity_id,
                    transformed_data
                )
            elif event.operation == 'delete':
                await self.legacy_db.delete(event.entity_type, event.entity_id)

            await self.mark_synced(event, 'new_to_legacy')

            logger.info(
                f"Synced {event.operation} {event.entity_type}:{event.entity_id} "
                f"from new to legacy"
            )

        except Exception as e:
            logger.error(f"Failed to sync from new: {e}")
            await self.handle_sync_error(event, 'new_to_legacy', str(e))

class ConflictResolver:
    """Разрешение конфликтов при двунаправленной синхронизации"""

    async def resolve(
        self,
        entity_type: str,
        entity_id: str,
        legacy_data: Dict,
        new_data: Dict
    ) -> Dict:
        """
        Стратегия разрешения конфликтов:
        - Для мигрированных сервисов: новая система приоритетнее
        - Для не мигрированных: legacy приоритетнее
        - Для полей: используем последнее изменение
        """
        migration_status = await self.get_migration_status(entity_type)

        if migration_status == 'migrated':
            # Новая система является source of truth
            return new_data
        elif migration_status == 'legacy':
            # Legacy является source of truth
            return legacy_data
        else:
            # Частичная миграция - merge по timestamp
            return await self.merge_by_timestamp(legacy_data, new_data)
```

## Best practices и антипаттерны

### Best Practices

```
┌─────────────────────────────────────────────────────────────────┐
│                      BEST PRACTICES                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ Начинайте с периферийных модулей                           │
│     Мигрируйте сначала модули с минимальными зависимостями     │
│                                                                 │
│  ✅ Создайте чёткие API контракты                              │
│     Определите интерфейсы до начала миграции                   │
│                                                                 │
│  ✅ Инвестируйте в мониторинг                                  │
│     Сравнивайте метрики legacy и новых сервисов                │
│                                                                 │
│  ✅ Автоматизируйте развёртывание                              │
│     CI/CD для обеих систем с возможностью отката               │
│                                                                 │
│  ✅ Документируйте прогресс                                    │
│     Ведите реестр мигрированных компонентов                    │
│                                                                 │
│  ✅ Планируйте откат                                           │
│     Каждый шаг миграции должен быть обратимым                  │
│                                                                 │
│  ✅ Тестируйте в продакшене                                    │
│     Canary deployments, shadow traffic                         │
│                                                                 │
│  ✅ Установите deadline                                        │
│     Без дедлайна миграция может длиться бесконечно             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Антипаттерны

| Антипаттерн | Проблема | Решение |
|------------|----------|---------|
| **"Big Bang" внутри Strangler** | Попытка мигрировать большой модуль целиком | Разбейте на подмодули |
| **Двойная запись везде** | Синхронизация всех данных между системами | Определите source of truth |
| **Отсутствие feature flags** | Невозможность быстрого отката | Внедрите систему флагов |
| **Игнорирование legacy** | Прекращение поддержки до миграции | Продолжайте фиксить баги |
| **Бесконечная миграция** | Нет плана завершения | Установите KPI и сроки |
| **Copy-paste кода** | Дублирование логики в обеих системах | Выделите общие библиотеки |

### Чек-лист успешной миграции

```python
# migration_checklist.py

class MigrationChecklist:
    """Чек-лист для проверки готовности к миграции модуля"""

    checks = [
        # Подготовка
        ("Определены границы модуля", "preparation"),
        ("Составлен список зависимостей", "preparation"),
        ("Создан API контракт", "preparation"),
        ("Настроен мониторинг", "preparation"),
        ("Написаны интеграционные тесты", "preparation"),

        # Реализация
        ("Реализован новый сервис", "implementation"),
        ("Настроена маршрутизация в facade", "implementation"),
        ("Настроена синхронизация данных", "implementation"),
        ("Проведено нагрузочное тестирование", "implementation"),

        # Rollout
        ("Развёрнут canary (5% трафика)", "rollout"),
        ("Проверены метрики canary", "rollout"),
        ("Увеличен трафик до 50%", "rollout"),
        ("Проверена производительность", "rollout"),
        ("Переключено 100% трафика", "rollout"),

        # Завершение
        ("Удалены правила маршрутизации legacy", "cleanup"),
        ("Отключена синхронизация данных", "cleanup"),
        ("Удалён код из legacy", "cleanup"),
        ("Обновлена документация", "cleanup"),
    ]

    def get_progress(self, completed: list) -> dict:
        """Расчёт прогресса миграции"""
        total = len(self.checks)
        done = len([c for c in self.checks if c[0] in completed])

        by_phase = {}
        for check, phase in self.checks:
            if phase not in by_phase:
                by_phase[phase] = {'total': 0, 'done': 0}
            by_phase[phase]['total'] += 1
            if check in completed:
                by_phase[phase]['done'] += 1

        return {
            'overall': f"{done}/{total} ({done/total*100:.0f}%)",
            'by_phase': {
                phase: f"{data['done']}/{data['total']}"
                for phase, data in by_phase.items()
            }
        }
```

## Связанные паттерны

```
┌─────────────────────────────────────────────────────────────────┐
│                   СВЯЗАННЫЕ ПАТТЕРНЫ                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────┐     ┌─────────────────┐                   │
│  │ Branch by       │     │ Anti-Corruption │                   │
│  │ Abstraction     │     │ Layer           │                   │
│  └────────┬────────┘     └────────┬────────┘                   │
│           │                       │                             │
│           ▼                       ▼                             │
│  Изоляция изменений      Защита от legacy API                  │
│  через интерфейсы        и моделей данных                      │
│                                                                 │
│  ┌─────────────────┐     ┌─────────────────┐                   │
│  │ Feature Toggles │     │ Parallel Run    │                   │
│  └────────┬────────┘     └────────┬────────┘                   │
│           │                       │                             │
│           ▼                       ▼                             │
│  Переключение между      Одновременное выполнение              │
│  старой и новой логикой  в обеих системах                      │
│                                                                 │
│  ┌─────────────────┐     ┌─────────────────┐                   │
│  │ Backend for     │     │ API Gateway     │                   │
│  │ Frontend (BFF)  │     │                 │                   │
│  └────────┬────────┘     └────────┬────────┘                   │
│           │                       │                             │
│           ▼                       ▼                             │
│  Отдельный API для       Единая точка входа                    │
│  каждого клиента         для маршрутизации                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Ресурсы для изучения

### Книги
- **"Building Microservices"** - Sam Newman (глава про миграцию)
- **"Monolith to Microservices"** - Sam Newman (полностью посвящена теме)
- **"Working Effectively with Legacy Code"** - Michael Feathers

### Статьи
- [StranglerFigApplication](https://martinfowler.com/bliki/StranglerFigApplication.html) - Martin Fowler
- [Strangler Pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/strangler-fig) - Microsoft Azure Patterns
- [How to break a Monolith into Microservices](https://martinfowler.com/articles/break-monolith-into-microservices.html)

### Примеры из практики
- **Amazon** - миграция с монолита (2001-2006)
- **Netflix** - переход на микросервисы и облако (2008-2016)
- **Spotify** - постепенная декомпозиция монолита

### Инструменты
- **Nginx** / **Kong** / **Envoy** - для реализации Strangler Facade
- **Istio** / **Linkerd** - service mesh для маршрутизации
- **LaunchDarkly** / **Unleash** - для feature toggles
- **Debezium** - для CDC и синхронизации данных
