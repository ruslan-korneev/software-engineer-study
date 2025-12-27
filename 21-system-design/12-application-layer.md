# Application Layer (Слой приложения)

## Содержание
1. [Что такое Application Layer](#что-такое-application-layer)
2. [Web Layer vs Application Layer](#web-layer-vs-application-layer)
3. [Микросервисы vs Монолит](#микросервисы-vs-монолит)
4. [Service Discovery](#service-discovery)
5. [Межсервисная коммуникация](#межсервисная-коммуникация)
6. [API Gateway](#api-gateway)
7. [Sidecar Pattern и Service Mesh](#sidecar-pattern-и-service-mesh)
8. [Примеры архитектур](#примеры-архитектур)

---

## Что такое Application Layer

**Application Layer** (слой приложения) — это уровень системной архитектуры, который содержит бизнес-логику приложения. Это "мозг" системы, где происходит обработка данных, принятие решений и координация между различными компонентами.

### Основные функции Application Layer

```
┌─────────────────────────────────────────────────────────────────┐
│                        Клиенты                                  │
│              (Web, Mobile, IoT, Third-party)                    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Web Layer (Presentation)                     │
│         - Обработка HTTP запросов                               │
│         - Маршрутизация                                         │
│         - Сериализация/десериализация                           │
│         - Аутентификация                                        │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                  APPLICATION LAYER                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Service A  │  │   Service B  │  │   Service C  │          │
│  │  (Заказы)    │  │   (Платежи)  │  │  (Доставка)  │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                 │
│  - Бизнес-логика                                                │
│  - Валидация бизнес-правил                                      │
│  - Оркестрация процессов                                        │
│  - Транзакции                                                   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Data Layer                                 │
│         - Базы данных                                           │
│         - Кэши                                                  │
│         - Файловые хранилища                                    │
└─────────────────────────────────────────────────────────────────┘
```

### Ключевые характеристики

1. **Stateless (без состояния)** — каждый запрос обрабатывается независимо
2. **Горизонтально масштабируемый** — можно добавлять новые экземпляры
3. **Изолированный** — не зависит напрямую от способа представления данных
4. **Тестируемый** — бизнес-логика отделена от инфраструктуры

---

## Web Layer vs Application Layer

### Разделение ответственности

| Аспект | Web Layer | Application Layer |
|--------|-----------|-------------------|
| **Основная задача** | Обработка HTTP | Бизнес-логика |
| **Знание о HTTP** | Полное | Отсутствует |
| **Состояние** | Stateless | Stateless |
| **Зависимости** | Веб-фреймворки | Доменные модели |
| **Пример кода** | Controllers, Handlers | Services, Use Cases |

### Почему важно разделение

```python
# ❌ ПЛОХО: Бизнес-логика в контроллере (Web Layer)
class OrderController:
    def create_order(self, request):
        user_id = request.user.id
        items = request.json['items']

        # Бизнес-логика смешана с обработкой HTTP
        total = 0
        for item in items:
            product = db.get_product(item['id'])
            if product.stock < item['quantity']:
                return Response({"error": "Not enough stock"}, 400)
            total += product.price * item['quantity']

        if total > 10000:
            discount = total * 0.1
            total -= discount

        order = db.create_order(user_id, items, total)
        return Response({"order_id": order.id}, 201)
```

```python
# ✅ ХОРОШО: Разделение на слои

# Application Layer (Service)
class OrderService:
    def __init__(self, product_repo, order_repo, pricing_service):
        self.product_repo = product_repo
        self.order_repo = order_repo
        self.pricing_service = pricing_service

    def create_order(self, user_id: str, items: list[OrderItem]) -> Order:
        """Чистая бизнес-логика без знания о HTTP"""
        # Проверка наличия товаров
        for item in items:
            product = self.product_repo.get(item.product_id)
            if not product.has_stock(item.quantity):
                raise InsufficientStockError(product.id, item.quantity)

        # Расчет стоимости
        total = self.pricing_service.calculate_total(items)

        # Создание заказа
        return self.order_repo.create(user_id, items, total)


# Web Layer (Controller)
class OrderController:
    def __init__(self, order_service: OrderService):
        self.order_service = order_service

    def create_order(self, request):
        """Только обработка HTTP"""
        try:
            items = [OrderItem(**i) for i in request.json['items']]
            order = self.order_service.create_order(
                user_id=request.user.id,
                items=items
            )
            return Response({"order_id": order.id}, 201)
        except InsufficientStockError as e:
            return Response({"error": str(e)}, 400)
        except ValidationError as e:
            return Response({"error": str(e)}, 422)
```

### Архитектурная схема разделения

```
┌────────────────────────────────────────────────────────────────────┐
│                         WEB LAYER                                  │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │
│  │  REST API    │  │   GraphQL    │  │   gRPC       │             │
│  │  Controller  │  │   Resolver   │  │   Handler    │             │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘             │
│         │                 │                 │                      │
│         └─────────────────┼─────────────────┘                      │
│                           ▼                                        │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │              Request/Response Mapping                       │   │
│  │              (DTO ↔ Domain Objects)                         │   │
│  └────────────────────────────────────────────────────────────┘   │
│                           │                                        │
└───────────────────────────┼────────────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────────────┐
│                     APPLICATION LAYER                              │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │                    Use Cases / Services                       │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐           │ │
│  │  │CreateOrder  │  │ProcessPay   │  │SendNotif    │           │ │
│  │  │UseCase      │  │UseCase      │  │UseCase      │           │ │
│  │  └─────────────┘  └─────────────┘  └─────────────┘           │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                           │                                        │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │                   Domain Services                             │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐           │ │
│  │  │PricingServ  │  │InventorySrv│  │FraudDetect  │           │ │
│  │  └─────────────┘  └─────────────┘  └─────────────┘           │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## Микросервисы vs Монолит

### Монолитная архитектура

**Монолит** — это приложение, где все компоненты (UI, бизнес-логика, доступ к данным) развертываются как единый артефакт.

```
┌─────────────────────────────────────────────────────────────┐
│                     MONOLITH                                │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    Единый код                        │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐  │   │
│  │  │  Users   │ │  Orders  │ │ Payments │ │Shipping│  │   │
│  │  │  Module  │ │  Module  │ │  Module  │ │ Module │  │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └────────┘  │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│                           ▼                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                  Общая БД                            │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

#### Плюсы монолита

| Преимущество | Описание |
|-------------|----------|
| **Простота разработки** | Один репозиторий, единый стек технологий |
| **Простота деплоя** | Один артефакт для развертывания |
| **Простота тестирования** | E2E тесты в одном месте |
| **Низкая latency** | Вызовы внутри процесса (in-process) |
| **Простота отладки** | Весь код в одном месте, единый стек вызовов |
| **Транзакции** | ACID транзакции "из коробки" |

#### Минусы монолита

| Недостаток | Описание |
|-----------|----------|
| **Сложность масштабирования** | Нельзя масштабировать отдельные части |
| **Риск деплоя** | Изменение одной функции требует редеплоя всего |
| **Зависимость от стека** | Сложно использовать разные технологии |
| **Coupling (связанность)** | Изменения в одном модуле влияют на другие |
| **Длинные сборки** | Большая кодовая база = долгие билды |
| **Сложность понимания** | Со временем код становится "big ball of mud" |

### Микросервисная архитектура

**Микросервисы** — это архитектурный стиль, где приложение разбито на независимые сервисы, каждый из которых решает одну бизнес-задачу.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         MICROSERVICES                                   │
│                                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │   Users     │  │   Orders    │  │  Payments   │  │  Shipping   │    │
│  │   Service   │  │   Service   │  │   Service   │  │   Service   │    │
│  │             │  │             │  │             │  │             │    │
│  │  Python     │  │    Java     │  │    Go       │  │   Rust      │    │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘    │
│         │                │                │                │           │
│         ▼                ▼                ▼                ▼           │
│     ┌──────┐         ┌──────┐         ┌──────┐         ┌──────┐       │
│     │ DB   │         │ DB   │         │ DB   │         │ DB   │       │
│     │Postgr│         │Mongo │         │Postgr│         │Redis │       │
│     └──────┘         └──────┘         └──────┘         └──────┘       │
│                                                                         │
│  ◄────────────────── Сеть (HTTP/gRPC/MQ) ──────────────────────►       │
└─────────────────────────────────────────────────────────────────────────┘
```

#### Плюсы микросервисов

| Преимущество | Описание |
|-------------|----------|
| **Независимый деплой** | Можно обновлять сервисы по отдельности |
| **Технологическая свобода** | Каждый сервис на своём стеке |
| **Изоляция отказов** | Падение одного сервиса не роняет систему |
| **Масштабируемость** | Можно масштабировать только нужные сервисы |
| **Автономные команды** | Команды владеют своими сервисами |
| **Понятные границы** | Чёткое разделение по бизнес-доменам |

#### Минусы микросервисов

| Недостаток | Описание |
|-----------|----------|
| **Сложность распределённой системы** | Network failures, latency, data consistency |
| **Сложность отладки** | Distributed tracing, логирование |
| **Накладные расходы** | Сеть, сериализация, service discovery |
| **Data consistency** | Нет ACID между сервисами |
| **Операционная сложность** | DevOps, мониторинг, алертинг |
| **Тестирование** | Contract testing, интеграционные тесты |

### Когда что использовать

```
                    Начало проекта
                          │
                          ▼
            ┌───────────────────────┐
            │ Понимаете ли вы домен │
            │     достаточно?       │
            └───────────┬───────────┘
                        │
             ┌──────────┴──────────┐
             │ НЕТ                 │ ДА
             ▼                     ▼
        ┌─────────┐    ┌─────────────────────────┐
        │ МОНОЛИТ │    │ Большая команда (>20)?  │
        │(начните │    └───────────┬─────────────┘
        │ с него) │                │
        └─────────┘     ┌──────────┴──────────┐
                        │ НЕТ                 │ ДА
                        ▼                     ▼
                   ┌─────────┐        ┌──────────────┐
                   │ МОНОЛИТ │        │ Критичны ли  │
                   │ модулярн│        │ независимое  │
                   └─────────┘        │ масштабирова-│
                                      │ ние/деплой?  │
                                      └──────┬───────┘
                                             │
                                  ┌──────────┴──────────┐
                                  │ НЕТ                 │ ДА
                                  ▼                     ▼
                             ┌─────────┐       ┌─────────────────┐
                             │ МОНОЛИТ │       │  МИКРОСЕРВИСЫ   │
                             │ модулярн│       └─────────────────┘
                             └─────────┘
```

### Таблица сравнения

| Критерий | Монолит | Микросервисы |
|----------|---------|--------------|
| **Размер команды** | 1-15 человек | 15+ человек |
| **Зрелость продукта** | MVP, стартап | Зрелый продукт |
| **Понимание домена** | Изучается | Хорошо понятен |
| **Требования к SLA** | Умеренные | Высокие |
| **Бюджет на инфраструктуру** | Ограничен | Достаточный |
| **DevOps экспертиза** | Базовая | Продвинутая |

### Эволюционный подход

```
Этап 1: Монолит                 Этап 2: Модульный монолит
┌─────────────────────┐         ┌─────────────────────────┐
│ ┌─────┐ ┌─────┐    │         │ ┌─────┐ ┌─────┐ ┌─────┐│
│ │Users│ │Order│... │   ──►   │ │Users│ │Order│ │Paym ││
│ └─────┘ └─────┘    │         │ │ Mod │ │ Mod │ │ Mod ││
│    Спагетти-код    │         │ └──┬──┘ └──┬──┘ └──┬──┘│
└─────────────────────┘         │    │      │      │   │
                                │ ┌──┴──────┴──────┴──┐│
                                │ │   Shared Kernel   ││
                                │ └───────────────────┘│
                                └─────────────────────────┘
        │
        │ Этап 3: Выделение сервисов
        ▼
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  ┌─────────┐   ┌────────────────────────────────────┐  │
│  │ Payment │   │          Основной монолит          │  │
│  │ Service │◄──┤  ┌───────┐  ┌───────┐  ┌───────┐  │  │
│  └─────────┘   │  │ Users │  │ Order │  │ Stock │  │  │
│       ▲        │  │  Mod  │  │  Mod  │  │  Mod  │  │  │
│       │        │  └───────┘  └───────┘  └───────┘  │  │
│   Выделили     └────────────────────────────────────┘  │
│   первым,                                              │
│   т.к. критичен                                        │
└─────────────────────────────────────────────────────────┘
```

---

## Service Discovery

### Зачем нужен Service Discovery

В распределённой системе сервисы должны находить друг друга. Статическая конфигурация не работает, потому что:

1. **Динамические IP-адреса** — контейнеры получают новые IP при перезапуске
2. **Auto-scaling** — количество инстансов меняется
3. **Failover** — нездоровые инстансы должны исключаться
4. **Множество окружений** — dev, staging, production

```
Проблема без Service Discovery:

┌────────────┐              ┌────────────┐
│  Service A │──────────────│  Service B │
│            │  Hardcoded   │ IP: ???    │
│            │  IP: 10.0.0.5│            │
└────────────┘              └────────────┘
       │
       │  Service B перезапустился
       │  Новый IP: 10.0.0.8
       ▼
┌────────────┐              ┌────────────┐
│  Service A │──────X───────│  Service B │
│            │  Старый IP   │ IP: 10.0.0.8│
│            │  не работает │            │
└────────────┘              └────────────┘
```

### Компоненты Service Discovery

```
┌─────────────────────────────────────────────────────────────────┐
│                    SERVICE REGISTRY                             │
│  ┌────────────────────────────────────────────────────────────┐│
│  │  Service Name    │    Instances                            ││
│  ├──────────────────┼─────────────────────────────────────────┤│
│  │  order-service   │  10.0.0.5:8080, 10.0.0.6:8080          ││
│  │  user-service    │  10.0.0.10:3000, 10.0.0.11:3000        ││
│  │  payment-service │  10.0.0.20:9000                         ││
│  └────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
         ▲                                    │
         │ Register                           │ Query
         │                                    ▼
┌─────────────────┐                  ┌─────────────────┐
│   order-service │                  │   api-gateway   │
│   (при старте   │                  │   (находит      │
│    регистрируется)                 │    сервисы)     │
└─────────────────┘                  └─────────────────┘
```

### Client-side Discovery

Клиент сам запрашивает реестр сервисов и выбирает инстанс:

```
┌──────────────────┐
│  Service Registry│◄─────────────────────────────────┐
│  (Consul/etcd)   │                                  │
└────────┬─────────┘                                  │
         │                                            │
         │ 1. Получить список                         │ 3. Register
         │    инстансов order-service                 │
         ▼                                            │
┌──────────────────┐         2. Прямой запрос    ┌───────────────┐
│   API Gateway    │─────────────────────────────│ order-service │
│                  │    к выбранному инстансу    │ Instance 1    │
│  Встроенный LB:  │                             └───────────────┘
│  - Round Robin   │                             ┌───────────────┐
│  - Least Conn    │                             │ order-service │
│  - Random        │                             │ Instance 2    │
└──────────────────┘                             └───────────────┘
```

**Плюсы Client-side:**
- Меньше hop'ов (нет промежуточного LB)
- Клиент может использовать умную маршрутизацию
- Меньше точек отказа

**Минусы Client-side:**
- Клиент должен реализовать логику discovery
- Зависимость от библиотеки (Ribbon, go-micro)
- Сложнее управлять централизованно

### Server-side Discovery

Клиент обращается к Load Balancer, который знает о доступных инстансах:

```
┌──────────────────┐
│  Service Registry│◄────────────────────────────────┐
│  (Consul/etcd)   │                                 │
└────────┬─────────┘                                 │
         │                                           │
         │ Sync                                      │ 3. Register
         ▼                                           │
┌──────────────────┐         2. Forward         ┌───────────────┐
│  Load Balancer   │────────────────────────────│ order-service │
│  (Nginx/Envoy)   │                            │ Instance 1    │
└────────▲─────────┘                            └───────────────┘
         │                                      ┌───────────────┐
         │ 1. Request                           │ order-service │
┌────────┴─────────┐                            │ Instance 2    │
│   API Gateway    │                            └───────────────┘
└──────────────────┘
```

**Плюсы Server-side:**
- Клиент простой (не знает о discovery)
- Централизованное управление
- Проще реализовать A/B тестирование, canary

**Минусы Server-side:**
- Дополнительный hop (latency)
- Load Balancer — точка отказа
- Нужна высокая доступность LB

### Инструменты Service Discovery

#### Consul (HashiCorp)

```yaml
# Регистрация сервиса в Consul
service:
  name: "order-service"
  port: 8080
  tags:
    - "v1"
    - "primary"
  check:
    http: "http://localhost:8080/health"
    interval: "10s"
    timeout: "2s"
```

```python
# Получение адресов сервиса
import consul

client = consul.Consul()

# Получить все здоровые инстансы
_, services = client.health.service('order-service', passing=True)

for service in services:
    address = service['Service']['Address']
    port = service['Service']['Port']
    print(f"Found instance: {address}:{port}")
```

**Особенности Consul:**
- Полноценный service mesh (Consul Connect)
- Multi-datacenter из коробки
- Key-Value store
- Встроенный DNS интерфейс

#### etcd

```go
// Регистрация в etcd
import (
    "context"
    clientv3 "go.etcd.io/etcd/client/v3"
)

cli, _ := clientv3.New(clientv3.Config{
    Endpoints: []string{"localhost:2379"},
})

// Создаём lease (TTL для записи)
lease, _ := cli.Grant(context.Background(), 10)

// Регистрируем сервис
cli.Put(context.Background(),
    "/services/order-service/instance-1",
    "10.0.0.5:8080",
    clientv3.WithLease(lease.ID),
)

// Keep-alive для продления lease
cli.KeepAlive(context.Background(), lease.ID)
```

**Особенности etcd:**
- Используется в Kubernetes
- Строгая консистентность (Raft)
- Watch API для real-time обновлений

#### ZooKeeper

```java
// Регистрация в ZooKeeper
import org.apache.zookeeper.*;

ZooKeeper zk = new ZooKeeper("localhost:2181", 3000, null);

// Создаём эфемерный узел (удаляется при отключении)
zk.create(
    "/services/order-service/instance-1",
    "10.0.0.5:8080".getBytes(),
    ZooDefs.Ids.OPEN_ACL_UNSAFE,
    CreateMode.EPHEMERAL
);

// Получение списка инстансов
List<String> children = zk.getChildren("/services/order-service", true);
```

**Особенности ZooKeeper:**
- Проверенный временем (с 2008)
- Используется в Kafka, Hadoop
- Ephemeral nodes для автоматического удаления
- Более сложный в эксплуатации

### Сравнение инструментов

| Характеристика | Consul | etcd | ZooKeeper |
|----------------|--------|------|-----------|
| **Язык** | Go | Go | Java |
| **Консистентность** | Raft (CP) | Raft (CP) | ZAB (CP) |
| **Multi-DC** | Да | Нет (federation) | Нет |
| **Health Checks** | Встроенные | Нет | Нет |
| **DNS Interface** | Да | Нет | Нет |
| **UI** | Да | Нет (сторонние) | Нет |
| **K8s native** | Нет | Да (используется) | Нет |

---

## Межсервисная коммуникация

### Синхронная коммуникация

#### HTTP/REST

```python
# Сервис заказов вызывает сервис пользователей
import httpx

class UserServiceClient:
    def __init__(self, base_url: str):
        self.base_url = base_url
        self.client = httpx.Client(timeout=5.0)

    def get_user(self, user_id: str) -> dict:
        response = self.client.get(f"{self.base_url}/users/{user_id}")
        response.raise_for_status()
        return response.json()

class OrderService:
    def __init__(self, user_client: UserServiceClient):
        self.user_client = user_client

    def create_order(self, user_id: str, items: list) -> Order:
        # Синхронный вызов другого сервиса
        user = self.user_client.get_user(user_id)

        if not user['is_active']:
            raise UserInactiveError(user_id)

        # ... создание заказа
```

**Плюсы REST:**
- Простота и понятность
- Широкая поддержка инструментов
- Человекочитаемый формат (JSON)
- Stateless

**Минусы REST:**
- Overhead на текстовый формат
- Нет строгой типизации (без OpenAPI)
- HTTP/1.1 — одно соединение на запрос

#### gRPC

```protobuf
// user.proto
syntax = "proto3";

package user;

service UserService {
    rpc GetUser(GetUserRequest) returns (User);
    rpc ListUsers(ListUsersRequest) returns (stream User);
}

message GetUserRequest {
    string user_id = 1;
}

message User {
    string id = 1;
    string email = 2;
    string name = 3;
    bool is_active = 4;
}
```

```python
# gRPC клиент
import grpc
import user_pb2
import user_pb2_grpc

class OrderService:
    def __init__(self, user_service_addr: str):
        channel = grpc.insecure_channel(user_service_addr)
        self.user_stub = user_pb2_grpc.UserServiceStub(channel)

    def create_order(self, user_id: str, items: list) -> Order:
        # gRPC вызов
        request = user_pb2.GetUserRequest(user_id=user_id)
        user = self.user_stub.GetUser(request)

        if not user.is_active:
            raise UserInactiveError(user_id)

        # ... создание заказа
```

**Плюсы gRPC:**
- Бинарный формат (Protocol Buffers) — быстрее и компактнее
- Строгая типизация из proto-файлов
- HTTP/2 — мультиплексирование, стриминг
- Кодогенерация для клиентов

**Минусы gRPC:**
- Сложнее отладка (бинарный формат)
- Не работает напрямую из браузера
- Требует proto-файлы

### Сравнение REST vs gRPC

```
REST (JSON over HTTP/1.1):
┌──────────────┐                    ┌──────────────┐
│   Client     │                    │   Server     │
│              │  ──────────────▶   │              │
│              │  HTTP/1.1          │              │
│              │  Content-Type:     │              │
│              │  application/json  │              │
│              │                    │              │
│              │  {"user_id": "123"}│              │
│              │  (15 bytes)        │              │
└──────────────┘                    └──────────────┘

gRPC (Protobuf over HTTP/2):
┌──────────────┐                    ┌──────────────┐
│   Client     │                    │   Server     │
│              │  ══════════════▶   │              │
│              │  HTTP/2            │              │
│              │  Multiplexed       │              │
│              │  Bidirectional     │              │
│              │                    │              │
│              │  [binary: 5 bytes] │              │
└──────────────┘                    └──────────────┘
```

| Аспект | REST | gRPC |
|--------|------|------|
| **Формат** | JSON (текст) | Protobuf (бинарный) |
| **Размер payload** | Больше | Меньше на 30-50% |
| **Скорость** | Медленнее | Быстрее в 2-10 раз |
| **Типизация** | OpenAPI (опционально) | Из proto-файлов |
| **Стриминг** | Нет (или SSE) | Да |
| **Браузер** | Да | gRPC-Web (с прокси) |
| **Отладка** | curl, Postman | grpcurl, Postman |

### Асинхронная коммуникация

#### Message Queues

```
Синхронный подход (проблема):
┌────────────┐  Запрос   ┌────────────┐  Запрос   ┌────────────┐
│   Order    │──────────▶│  Payment   │──────────▶│   Email    │
│  Service   │           │  Service   │           │  Service   │
│            │◀──────────│            │◀──────────│            │
│            │  Ответ    │            │  Ответ    │            │
└────────────┘           └────────────┘           └────────────┘
                                │
                                ▼
                    Если Email Service недоступен,
                    весь запрос падает!


Асинхронный подход (решение):
┌────────────┐  Запрос   ┌────────────┐
│   Order    │──────────▶│  Payment   │
│  Service   │           │  Service   │
│            │◀──────────│            │
│            │  Ответ    │     │      │
└────────────┘           └─────┼──────┘
                               │
                               ▼ Публикует событие
                    ┌────────────────────┐
                    │   Message Queue    │
                    │   (RabbitMQ/Kafka) │
                    └─────────┬──────────┘
                              │
                    ┌─────────┴─────────┐
                    ▼                   ▼
            ┌────────────┐      ┌────────────┐
            │   Email    │      │  Analytics │
            │  Service   │      │  Service   │
            └────────────┘      └────────────┘
               Может обработать позже, ничего не падает
```

```python
# Продюсер (Order Service)
import json
import pika

class OrderService:
    def __init__(self):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters('rabbitmq')
        )
        self.channel = self.connection.channel()
        self.channel.queue_declare(queue='order_events', durable=True)

    def create_order(self, user_id: str, items: list) -> Order:
        order = self._save_order(user_id, items)

        # Публикуем событие (Fire and Forget)
        event = {
            'type': 'ORDER_CREATED',
            'order_id': order.id,
            'user_id': user_id,
            'total': order.total
        }

        self.channel.basic_publish(
            exchange='',
            routing_key='order_events',
            body=json.dumps(event),
            properties=pika.BasicProperties(delivery_mode=2)  # persistent
        )

        return order
```

```python
# Консьюмер (Email Service)
import json
import pika

def process_order_event(ch, method, properties, body):
    event = json.loads(body)

    if event['type'] == 'ORDER_CREATED':
        send_order_confirmation_email(
            order_id=event['order_id'],
            user_id=event['user_id']
        )

    ch.basic_ack(delivery_tag=method.delivery_tag)


connection = pika.BlockingConnection(pika.ConnectionParameters('rabbitmq'))
channel = connection.channel()
channel.queue_declare(queue='order_events', durable=True)

channel.basic_consume(
    queue='order_events',
    on_message_callback=process_order_event
)

channel.start_consuming()
```

### Паттерны межсервисной коммуникации

```
1. Request-Response (синхронный):
┌────────┐  Запрос   ┌────────┐
│  A     │──────────▶│  B     │
│        │◀──────────│        │
└────────┘  Ответ    └────────┘

2. Fire-and-Forget (асинхронный):
┌────────┐  Событие  ┌────────┐
│  A     │──────────▶│ Queue  │──────────▶│  B     │
│        │           └────────┘           └────────┘
└────────┘  (A не ждёт B)

3. Publish-Subscribe:
                              ┌────────┐
                         ┌───▶│  B     │
┌────────┐  Событие  ┌───┴──┐ └────────┘
│  A     │──────────▶│Topic │
└────────┘           └───┬──┘ ┌────────┐
                         └───▶│  C     │
                              └────────┘

4. Request-Reply через очередь:
┌────────┐  Запрос   ┌────────┐  Запрос  ┌────────┐
│  A     │──────────▶│RequestQ│─────────▶│  B     │
│        │           └────────┘          │        │
│        │◀────────────────────────────  │        │
│        │  Ответ (в отдельной очереди)  │        │
└────────┘                               └────────┘
```

### Saga Pattern для распределённых транзакций

```
Проблема: Нельзя использовать ACID транзакции между сервисами

Решение: Saga — последовательность локальных транзакций
         с компенсирующими действиями при ошибке

Создание заказа (Saga):

┌────────────────────────────────────────────────────────────────────┐
│                                                                    │
│  1. CreateOrder     2. ReserveStock    3. ProcessPayment          │
│  ┌────────────┐    ┌────────────┐    ┌────────────┐               │
│  │   Order    │───▶│  Inventory │───▶│  Payment   │               │
│  │  Service   │    │  Service   │    │  Service   │               │
│  └────────────┘    └────────────┘    └────────────┘               │
│        │                │                  │                       │
│        ▼                ▼                  ▼                       │
│   [Заказ создан]  [Товар забронирован] [Оплата прошла]            │
│                                            │                       │
│                                            ▼ Успех!                │
│                                       [Заказ завершён]            │
│                                                                    │
│  При ошибке на шаге 3:                                            │
│                                                                    │
│  1. CancelPayment   2. ReleaseStock    3. CancelOrder             │
│  ┌────────────┐    ┌────────────┐    ┌────────────┐               │
│  │  Payment   │◀───│  Inventory │◀───│   Order    │               │
│  │  Service   │    │  Service   │    │  Service   │               │
│  │ (compensate)    │ (compensate)    │ (compensate)               │
│  └────────────┘    └────────────┘    └────────────┘               │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

```python
# Оркестратор Saga
class CreateOrderSaga:
    def __init__(self, order_service, inventory_service, payment_service):
        self.order_service = order_service
        self.inventory_service = inventory_service
        self.payment_service = payment_service
        self.completed_steps = []

    def execute(self, order_data: dict) -> Order:
        try:
            # Шаг 1: Создать заказ
            order = self.order_service.create(order_data)
            self.completed_steps.append(('order', order.id))

            # Шаг 2: Зарезервировать товар
            self.inventory_service.reserve(order.items)
            self.completed_steps.append(('inventory', order.items))

            # Шаг 3: Обработать платёж
            self.payment_service.process(order.id, order.total)
            self.completed_steps.append(('payment', order.id))

            return order

        except Exception as e:
            self._compensate()
            raise SagaFailedError(str(e))

    def _compensate(self):
        """Откат выполненных шагов в обратном порядке"""
        for step_type, step_data in reversed(self.completed_steps):
            if step_type == 'payment':
                self.payment_service.refund(step_data)
            elif step_type == 'inventory':
                self.inventory_service.release(step_data)
            elif step_type == 'order':
                self.order_service.cancel(step_data)
```

---

## API Gateway

### Что такое API Gateway

**API Gateway** — это единая точка входа для всех клиентов, которая маршрутизирует запросы к нужным микросервисам и реализует сквозную функциональность.

```
Без API Gateway:                    С API Gateway:

┌────────────┐                      ┌────────────┐
│   Client   │                      │   Client   │
└─────┬──────┘                      └─────┬──────┘
      │                                   │
      ├────────────────┐                  ▼
      │                │            ┌───────────────┐
      │                │            │  API Gateway  │
      ▼                ▼            │               │
┌──────────┐    ┌──────────┐       │ • Auth        │
│  Users   │    │  Orders  │       │ • Rate Limit  │
│ Service  │    │ Service  │       │ • Logging     │
└──────────┘    └──────────┘       │ • Routing     │
                                   └───────┬───────┘
  Клиент должен:                           │
  • Знать адреса всех сервисов      ┌──────┴──────┐
  • Реализовать auth везде          ▼             ▼
  • Обрабатывать разные протоколы  ┌──────────┐ ┌──────────┐
                                   │  Users   │ │  Orders  │
                                   │ Service  │ │ Service  │
                                   └──────────┘ └──────────┘
```

### Функции API Gateway

```
┌─────────────────────────────────────────────────────────────────────┐
│                          API GATEWAY                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐   │
│  │   Routing   │ │    Auth     │ │ Rate Limit  │ │   Caching   │   │
│  │             │ │             │ │             │ │             │   │
│  │ /users →    │ │ JWT verify  │ │ 100 req/min │ │ GET → Cache │   │
│  │ user-svc    │ │ API Keys    │ │ per client  │ │             │   │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘   │
│                                                                     │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐   │
│  │   Logging   │ │  Metrics    │ │  Transform  │ │ Load Balance│   │
│  │             │ │             │ │             │ │             │   │
│  │ Request ID  │ │ Prometheus  │ │ GraphQL →   │ │ Round Robin │   │
│  │ Tracing     │ │ Latency     │ │ REST        │ │ Weighted    │   │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘   │
│                                                                     │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐                   │
│  │Circuit Break│ │   Retry     │ │  CORS       │                   │
│  │             │ │             │ │             │                   │
│  │ Fail fast   │ │ Exp backoff │ │ Origin check│                   │
│  └─────────────┘ └─────────────┘ └─────────────┘                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Пример конфигурации (Kong)

```yaml
# Kong Gateway configuration
services:
  - name: user-service
    url: http://user-service:8080
    routes:
      - name: users-route
        paths:
          - /api/v1/users
        methods:
          - GET
          - POST
    plugins:
      - name: rate-limiting
        config:
          minute: 100
          policy: local
      - name: jwt
        config:
          secret_is_base64: false

  - name: order-service
    url: http://order-service:8080
    routes:
      - name: orders-route
        paths:
          - /api/v1/orders
    plugins:
      - name: request-transformer
        config:
          add:
            headers:
              - "X-Request-ID:$(uuid)"
```

### Популярные API Gateway решения

| Gateway | Язык | Особенности |
|---------|------|-------------|
| **Kong** | Lua/Go | Плагины, Kubernetes-native |
| **AWS API Gateway** | Managed | Serverless, Lambda интеграция |
| **Nginx** | C | Высокая производительность |
| **Envoy** | C++ | Service Mesh ready, gRPC |
| **Traefik** | Go | Автоконфигурация, Docker |
| **APISIX** | Lua | Динамические плагины |

### Backend for Frontend (BFF)

```
Проблема: разным клиентам нужны разные данные

                    ┌─────────────────┐
    Mobile App ────▶│                 │
                    │   API Gateway   │
    Web App ───────▶│  (Один для всех)│
                    │                 │
    TV App ────────▶│                 │
                    └─────────────────┘

    • Mobile нужны минимальные данные (трафик)
    • Web нужны полные данные
    • TV нужен специфичный формат


Решение: Backend for Frontend (BFF)

┌──────────┐     ┌─────────────┐
│  Mobile  │────▶│ Mobile BFF  │────┐
│   App    │     │ (минимум)   │    │
└──────────┘     └─────────────┘    │
                                    ▼
┌──────────┐     ┌─────────────┐  ┌────────────┐
│   Web    │────▶│   Web BFF   │─▶│  Backend   │
│   App    │     │ (полные)    │  │  Services  │
└──────────┘     └─────────────┘  └────────────┘
                                    ▲
┌──────────┐     ┌─────────────┐    │
│    TV    │────▶│   TV BFF    │────┘
│   App    │     │(специфичный)│
└──────────┘     └─────────────┘
```

---

## Sidecar Pattern и Service Mesh

### Sidecar Pattern

**Sidecar** — это вспомогательный контейнер, который деплоится рядом с основным приложением и берёт на себя инфраструктурные задачи.

```
Без Sidecar:                        С Sidecar:
┌─────────────────────────────┐     ┌──────────────────────────────────┐
│       Application           │     │              Pod                  │
│  ┌───────────────────────┐  │     │  ┌─────────────┐ ┌─────────────┐ │
│  │    Business Logic     │  │     │  │ Application │ │   Sidecar   │ │
│  ├───────────────────────┤  │     │  │             │ │   (Envoy)   │ │
│  │  Logging              │  │     │  │  Business   │ │             │ │
│  │  Metrics              │  │     │  │   Logic     │ │  • Logging  │ │
│  │  Service Discovery    │  │     │  │   ONLY      │ │  • Metrics  │ │
│  │  Circuit Breaker      │  │     │  │             │ │  • mTLS     │ │
│  │  Retry Logic          │  │     │  │             │ │  • Retry    │ │
│  │  TLS Termination      │  │     │  │             │ │  • Circuit  │ │
│  │  Tracing              │  │     │  │             │ │    Breaker  │ │
│  └───────────────────────┘  │     │  └─────────────┘ └─────────────┘ │
│                             │     │        │               ▲          │
│  Код перегружен             │     │        └───localhost───┘          │
│  инфраструктурой            │     │                                   │
└─────────────────────────────┘     └──────────────────────────────────┘
```

### Преимущества Sidecar

1. **Разделение ответственности** — приложение содержит только бизнес-логику
2. **Единообразие** — один sidecar для всех сервисов
3. **Независимое обновление** — можно обновить sidecar без изменения приложения
4. **Языконезависимость** — работает с любым стеком технологий

### Service Mesh

**Service Mesh** — это инфраструктурный слой, состоящий из sidecar-прокси, которые управляют всей сетевой коммуникацией между сервисами.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           SERVICE MESH                                  │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                     Control Plane (Istiod)                       │   │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐             │   │
│  │  │   Config     │ │   Service    │ │  Certificate │             │   │
│  │  │   (Galley)   │ │  Discovery   │ │   Authority  │             │   │
│  │  └──────────────┘ └──────────────┘ └──────────────┘             │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                    │            │            │                          │
│                    ▼            ▼            ▼                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                       Data Plane                                 │   │
│  │                                                                  │   │
│  │  ┌─────────────────┐         ┌─────────────────┐                │   │
│  │  │      Pod A      │         │      Pod B      │                │   │
│  │  │ ┌─────┐ ┌─────┐ │ mTLS    │ ┌─────┐ ┌─────┐ │                │   │
│  │  │ │ App │ │Envoy│ │◀───────▶│ │Envoy│ │ App │ │                │   │
│  │  │ └─────┘ └─────┘ │         │ └─────┘ └─────┘ │                │   │
│  │  └─────────────────┘         └─────────────────┘                │   │
│  │                                                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Функции Service Mesh

```
┌─────────────────────────────────────────────────────────────────┐
│                     SERVICE MESH FEATURES                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TRAFFIC MANAGEMENT          SECURITY                          │
│  ┌─────────────────────┐    ┌─────────────────────┐            │
│  │ • Load Balancing    │    │ • mTLS (mutual TLS) │            │
│  │ • Circuit Breaking  │    │ • RBAC policies     │            │
│  │ • Retries/Timeouts  │    │ • Cert rotation     │            │
│  │ • Rate Limiting     │    │ • Identity          │            │
│  │ • Canary/A-B tests  │    └─────────────────────┘            │
│  │ • Traffic mirroring │                                        │
│  └─────────────────────┘    OBSERVABILITY                       │
│                              ┌─────────────────────┐            │
│  RESILIENCE                  │ • Distributed Trace │            │
│  ┌─────────────────────┐    │ • Metrics (Prometheus)           │
│  │ • Fault Injection   │    │ • Access Logs       │            │
│  │ • Health Checks     │    │ • Topology Graph    │            │
│  │ • Outlier Detection │    └─────────────────────┘            │
│  └─────────────────────┘                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Istio пример

```yaml
# VirtualService: маршрутизация трафика
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: order-service
spec:
  hosts:
    - order-service
  http:
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: order-service
            subset: v2
    - route:
        - destination:
            host: order-service
            subset: v1
          weight: 90
        - destination:
            host: order-service
            subset: v2
          weight: 10  # Canary deployment

---
# DestinationRule: политики для трафика
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: order-service
spec:
  host: order-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: UPGRADE
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 10s
      baseEjectionTime: 30s
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```

### Сравнение Service Mesh решений

| Feature | Istio | Linkerd | Consul Connect |
|---------|-------|---------|----------------|
| **Proxy** | Envoy | linkerd2-proxy | Envoy |
| **Сложность** | Высокая | Низкая | Средняя |
| **Ресурсы** | Больше | Меньше | Средне |
| **mTLS** | Да | Да | Да |
| **Multi-cluster** | Да | Да | Да |
| **Traffic split** | Да | Да | Да |
| **Язык proxy** | C++ | Rust | C++ |

---

## Примеры архитектур

### E-commerce платформа

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           КЛИЕНТЫ                                       │
│    ┌────────┐    ┌────────┐    ┌────────┐    ┌────────┐                │
│    │  Web   │    │ Mobile │    │  API   │    │  Admin │                │
│    │  App   │    │  App   │    │Partners│    │ Panel  │                │
│    └───┬────┘    └───┬────┘    └───┬────┘    └───┬────┘                │
└────────┼─────────────┼─────────────┼─────────────┼──────────────────────┘
         │             │             │             │
         └─────────────┴──────┬──────┴─────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        EDGE LAYER                                       │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                         CDN (CloudFlare)                          │ │
│  │              Static assets, DDoS protection, Caching              │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                              │                                          │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                      Load Balancer (AWS ALB)                       │ │
│  │                    SSL Termination, Health Checks                  │ │
│  └───────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      API GATEWAY (Kong)                                 │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐           │
│  │    Auth    │ │ Rate Limit │ │   Routing  │ │  Logging   │           │
│  └────────────┘ └────────────┘ └────────────┘ └────────────┘           │
└─────────────────────────────────────────────────────────────────────────┘
                              │
         ┌────────────────────┼────────────────────┐
         ▼                    ▼                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                       APPLICATION LAYER                                 │
│                                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐   │
│  │    User     │  │   Catalog   │  │    Cart     │  │    Order    │   │
│  │   Service   │  │   Service   │  │   Service   │  │   Service   │   │
│  │   (Go)      │  │   (Python)  │  │   (Node)    │  │   (Java)    │   │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘   │
│         │                │                │                │           │
│         ▼                ▼                ▼                ▼           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐   │
│  │ PostgreSQL  │  │  MongoDB    │  │    Redis    │  │ PostgreSQL  │   │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘   │
│                                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐   │
│  │   Payment   │  │  Shipping   │  │Notification │  │  Analytics  │   │
│  │   Service   │  │   Service   │  │   Service   │  │   Service   │   │
│  │   (Go)      │  │   (Go)      │  │  (Python)   │  │   (Python)  │   │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     MESSAGE BROKER (Kafka)                              │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │   order-events   │   payment-events   │   notification-events    │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     OBSERVABILITY                                       │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐        │
│  │ Prometheus │  │   Grafana  │  │   Jaeger   │  │     ELK    │        │
│  │  Metrics   │  │ Dashboards │  │  Tracing   │  │   Logging  │        │
│  └────────────┘  └────────────┘  └────────────┘  └────────────┘        │
└─────────────────────────────────────────────────────────────────────────┘
```

### Финтех платформа с высокими требованиями к надёжности

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         MULTI-REGION SETUP                              │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    Global Load Balancer                          │   │
│  │                  (GeoDNS + Anycast)                              │   │
│  └────────────────────────────┬────────────────────────────────────┘   │
│                               │                                         │
│          ┌────────────────────┼────────────────────┐                   │
│          │                    │                    │                   │
│          ▼                    ▼                    ▼                   │
│  ┌───────────────┐   ┌───────────────┐   ┌───────────────┐            │
│  │   Region A    │   │   Region B    │   │   Region C    │            │
│  │   (Primary)   │   │  (Secondary)  │   │   (DR site)   │            │
│  └───────┬───────┘   └───────┬───────┘   └───────┬───────┘            │
│          │                   │                   │                     │
└──────────┼───────────────────┼───────────────────┼─────────────────────┘
           │                   │                   │
           ▼                   ▼                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      REGION A (PRIMARY)                                 │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    Service Mesh (Istio)                          │   │
│  │                                                                  │   │
│  │  ┌──────────────────────────────────────────────────────────┐   │   │
│  │  │                    API Gateway                            │   │   │
│  │  │         (Rate Limiting, Auth, Circuit Breaker)            │   │   │
│  │  └──────────────────────────────────────────────────────────┘   │   │
│  │                              │                                   │   │
│  │      ┌───────────────────────┼───────────────────────┐          │   │
│  │      ▼                       ▼                       ▼          │   │
│  │  ┌─────────┐            ┌─────────┐            ┌─────────┐      │   │
│  │  │ Account │◀──────────▶│ Payment │◀──────────▶│  Fraud  │      │   │
│  │  │ Service │   gRPC     │ Service │   gRPC     │ Detection│     │   │
│  │  │ (3 pods)│            │ (5 pods)│            │ (3 pods)│      │   │
│  │  └────┬────┘            └────┬────┘            └────┬────┘      │   │
│  │       │                      │                      │           │   │
│  │       ▼                      ▼                      ▼           │   │
│  │  ┌─────────┐            ┌─────────┐            ┌─────────┐      │   │
│  │  │Postgres │            │Postgres │            │  Redis  │      │   │
│  │  │(Primary)│            │(Primary)│            │ Cluster │      │   │
│  │  │+ Replica│            │+ Replica│            │         │      │   │
│  │  └─────────┘            └─────────┘            └─────────┘      │   │
│  │                                                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    Event Streaming (Kafka)                       │   │
│  │      transaction-events  │  fraud-alerts  │  audit-logs         │   │
│  │                          │                │                      │   │
│  │   ┌──────────────────────┼────────────────┼────────────┐        │   │
│  │   ▼                      ▼                ▼            ▼        │   │
│  │ ┌────────┐          ┌────────┐       ┌────────┐  ┌────────┐     │   │
│  │ │Notific.│          │Complian│       │  Audit │  │Analytics│    │   │
│  │ │Service │          │Service │       │ Trail  │  │Pipeline │    │   │
│  │ └────────┘          └────────┘       └────────┘  └────────┘     │   │
│  │                                                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
                              │
                    Cross-region replication
                              │
                              ▼
        ┌─────────────────────────────────────────────────┐
        │              Region B (Hot Standby)             │
        │  • Read replicas для read-heavy операций        │
        │  • Async replication из Region A                │
        │  • Готов к failover за < 30 секунд             │
        └─────────────────────────────────────────────────┘
```

### SaaS платформа (Multi-tenant)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         MULTI-TENANT SaaS                               │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    Tenant Router (API Gateway)                   │   │
│  │                                                                  │   │
│  │  • Извлечение tenant_id из JWT/subdomain/header                 │   │
│  │  • Маршрутизация к нужному pool'у ресурсов                      │   │
│  │  • Rate limiting per tenant                                      │   │
│  │  • Tenant-specific feature flags                                 │   │
│  │                                                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│              ┌───────────────┼───────────────┐                         │
│              │               │               │                         │
│              ▼               ▼               ▼                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    APPLICATION LAYER                              │ │
│  │                                                                   │ │
│  │  Shared Services (все тенанты):                                  │ │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐            │ │
│  │  │   Auth   │ │  Billing │ │  Admin   │ │ Analytics│            │ │
│  │  │ Service  │ │ Service  │ │ Service  │ │ Service  │            │ │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘            │ │
│  │                                                                   │ │
│  │  Tenant-isolated Services:                                        │ │
│  │  ┌─────────────────────────────────────────────────────────────┐ │ │
│  │  │              Tenant Context Propagation                      │ │ │
│  │  │                                                              │ │ │
│  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │ │ │
│  │  │  │  Core    │  │ Workflow │  │ Reports  │  │  Import  │    │ │ │
│  │  │  │ Service  │  │ Engine   │  │ Service  │  │ Service  │    │ │ │
│  │  │  │          │  │          │  │          │  │          │    │ │ │
│  │  │  │tenant_id │  │tenant_id │  │tenant_id │  │tenant_id │    │ │ │
│  │  │  │ context  │  │ context  │  │ context  │  │ context  │    │ │ │
│  │  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘    │ │ │
│  │  │                                                              │ │ │
│  │  └─────────────────────────────────────────────────────────────┘ │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                              │                                          │
│                              ▼                                          │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                      DATA LAYER                                   │ │
│  │                                                                   │ │
│  │  Strategy 1: Shared DB + Row-level security                       │ │
│  │  ┌──────────────────────────────────────────────────────────┐    │ │
│  │  │  PostgreSQL (shared)                                      │    │ │
│  │  │  ┌────────────────────────────────────────────────────┐  │    │ │
│  │  │  │  SELECT * FROM orders WHERE tenant_id = :tenant    │  │    │ │
│  │  │  │  + Row Level Security policies                      │  │    │ │
│  │  │  └────────────────────────────────────────────────────┘  │    │ │
│  │  └──────────────────────────────────────────────────────────┘    │ │
│  │                                                                   │ │
│  │  Strategy 2: Schema per tenant                                    │ │
│  │  ┌──────────────────────────────────────────────────────────┐    │ │
│  │  │  PostgreSQL                                               │    │ │
│  │  │  ├── tenant_acme.orders                                   │    │ │
│  │  │  ├── tenant_globex.orders                                 │    │ │
│  │  │  └── tenant_initech.orders                                │    │ │
│  │  └──────────────────────────────────────────────────────────┘    │ │
│  │                                                                   │ │
│  │  Strategy 3: Database per tenant (Enterprise)                     │ │
│  │  ┌────────┐  ┌────────┐  ┌────────┐                              │ │
│  │  │ DB     │  │ DB     │  │ DB     │                              │ │
│  │  │ ACME   │  │GLOBEX  │  │INITECH │                              │ │
│  │  └────────┘  └────────┘  └────────┘                              │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Ключевые выводы

### Чек-лист при проектировании Application Layer

| Вопрос | Рекомендация |
|--------|--------------|
| **Монолит или микросервисы?** | Начинайте с модульного монолита, выделяйте сервисы по мере роста |
| **Service Discovery?** | Kubernetes: встроенный; вне K8s: Consul или etcd |
| **Синхронная или асинхронная связь?** | Синхронная для query, асинхронная для commands и events |
| **Нужен ли API Gateway?** | Да, если > 3 сервисов или нужна централизованная безопасность |
| **Нужен ли Service Mesh?** | Да, если > 10 сервисов и нужны mTLS, traffic management |

### Анти-паттерны

1. **Distributed Monolith** — микросервисы с сильной связностью
2. **Chatty Communication** — слишком много синхронных вызовов
3. **Shared Database** — несколько сервисов работают с одной таблицей
4. **Premature Decomposition** — микросервисы до понимания домена
5. **Missing Observability** — нет логирования, трейсинга, метрик

### Полезные ресурсы

- [microservices.io](https://microservices.io) — паттерны микросервисов
- [12factor.net](https://12factor.net) — принципы cloud-native приложений
- [Martin Fowler's blog](https://martinfowler.com) — архитектурные статьи
- [CNCF Landscape](https://landscape.cncf.io) — карта cloud-native технологий
