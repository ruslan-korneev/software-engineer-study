# Communication Patterns (Паттерны коммуникации)

Паттерны коммуникации определяют, как компоненты системы взаимодействуют друг с другом. Правильный выбор паттерна критически важен для масштабируемости, отказоустойчивости и поддерживаемости системы.

## Содержание раздела

- [x] [MVC (Model-View-Controller)](./01-mvc.md) — разделение приложения на слои данных, представления и управления
- [x] [Pub/Sub (Publish-Subscribe)](./02-pub-sub.md) — асинхронный обмен сообщениями через брокер
- [x] [Controller-Responder (Master-Slave)](./03-controller-responder.md) — централизованное управление и распределённое выполнение

## Обзор паттернов

### Сравнительная таблица

| Паттерн | Связанность | Синхронность | Масштабируемость | Сложность |
|---------|-------------|--------------|------------------|-----------|
| **MVC** | Средняя | Синхронный | Вертикальная | Низкая |
| **Pub/Sub** | Слабая | Асинхронный | Горизонтальная | Средняя |
| **Controller-Responder** | Средняя | Оба варианта | Горизонтальная (чтение) | Средняя |

### Когда использовать какой паттерн

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     ВЫБОР ПАТТЕРНА КОММУНИКАЦИИ                         │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │ Какой тип взаимодействия?     │
                    └───────────────────────────────┘
                                    │
            ┌───────────────────────┼───────────────────────┐
            ▼                       ▼                       ▼
    ┌───────────────┐      ┌───────────────┐      ┌───────────────┐
    │  UI + Логика  │      │ Событийное    │      │ Распределённые│
    │  + Данные     │      │ взаимодействие│      │ вычисления    │
    └───────┬───────┘      └───────┬───────┘      └───────┬───────┘
            │                      │                      │
            ▼                      ▼                      ▼
    ┌───────────────┐      ┌───────────────┐      ┌───────────────┐
    │     MVC       │      │   Pub/Sub     │      │  Controller-  │
    │               │      │               │      │  Responder    │
    │ Веб-приложения│      │ Микросервисы  │      │ Репликация БД │
    │ Desktop GUI   │      │ Уведомления   │      │ MapReduce     │
    │ REST APIs     │      │ Real-time     │      │ Load Balancing│
    └───────────────┘      └───────────────┘      └───────────────┘
```

## Основные концепции

### 1. MVC — Структурное разделение

Разделяет приложение на три слоя с чёткими обязанностями:

```
User → Controller → Model → View → User

• Model: бизнес-логика и данные
• View: отображение
• Controller: обработка ввода и координация
```

**Применение**: веб-фреймворки (Django, Rails, Spring MVC), desktop-приложения

### 2. Pub/Sub — Слабосвязанный обмен сообщениями

Издатели публикуют события, подписчики получают только интересующие их:

```
Publisher → [Message Broker] → Subscriber 1
                            → Subscriber 2
                            → Subscriber N
```

**Применение**: микросервисы, real-time системы, интеграция приложений

### 3. Controller-Responder — Централизованное управление

Один узел (Controller) управляет несколькими рабочими узлами (Responders):

```
            Controller
           /    |    \
          ▼     ▼     ▼
     Resp1   Resp2   Resp3
```

**Применение**: репликация БД, load balancing, распределённые вычисления

## Практические примеры выбора

### Сценарий 1: Интернет-магазин

```
Компоненты:
├── Веб-интерфейс → MVC (FastAPI + Jinja2)
├── Обработка заказов → Pub/Sub (RabbitMQ/Kafka)
│   ├── OrderService публикует "order.created"
│   ├── InventoryService подписан → резервирует товар
│   ├── PaymentService подписан → проводит оплату
│   └── NotificationService подписан → отправляет email
└── База данных → Controller-Responder
    ├── Primary: все записи
    └── Replicas: распределённое чтение
```

### Сценарий 2: Система аналитики

```
Компоненты:
├── Сбор данных → Pub/Sub (Kafka)
│   └── Collectors публикуют события в топики
├── Обработка → Controller-Responder (Spark)
│   ├── Driver (Controller): планирует задачи
│   └── Workers (Responders): выполняют MapReduce
└── Dashboard → MVC (React + FastAPI)
```

## Комбинирование паттернов

В реальных системах паттерны часто используются вместе:

```python
# MVC Controller использует Pub/Sub для интеграции
class OrderController:
    def __init__(self, order_service, event_bus):
        self.order_service = order_service
        self.event_bus = event_bus

    def create_order(self, request):
        # MVC: Controller обрабатывает запрос
        order = self.order_service.create(request.data)

        # Pub/Sub: публикуем событие для других сервисов
        self.event_bus.publish("order.created", {
            "order_id": order.id,
            "user_id": order.user_id
        })

        # MVC: возвращаем View
        return self.render("order_created.html", order=order)


# Controller-Responder для чтения
class OrderService:
    def __init__(self, db_cluster):
        self.db = db_cluster

    def create(self, data):
        # Запись через Primary
        return self.db.primary.insert("orders", data)

    def get_by_id(self, order_id):
        # Чтение с Replica
        return self.db.get_read_replica().find("orders", order_id)
```

## Дополнительные ресурсы

### Книги
- "Enterprise Integration Patterns" — Gregor Hohpe
- "Designing Data-Intensive Applications" — Martin Kleppmann
- "Patterns of Enterprise Application Architecture" — Martin Fowler

### Технологии для практики
- **MVC**: Django, FastAPI, Spring Boot
- **Pub/Sub**: Apache Kafka, RabbitMQ, Redis Pub/Sub
- **Controller-Responder**: PostgreSQL Replication, Redis Cluster, Celery

---

**Предыдущий раздел**: [Data Patterns](../02-data-patterns/readme.md)
**Следующий раздел**: [Resilience Patterns](../04-resilience-patterns/readme.md)
