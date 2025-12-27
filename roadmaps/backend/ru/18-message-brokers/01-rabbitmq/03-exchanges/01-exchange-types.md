# Типы обменников (Exchange Types)

## Введение

Exchange (обменник) - это компонент RabbitMQ, который принимает сообщения от издателей (producers) и направляет их в одну или несколько очередей на основе правил маршрутизации. Обменник никогда не хранит сообщения - он только маршрутизирует их.

## Схема работы Exchange

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│  Producer   │────▶│   Exchange   │────▶│   Queue 1   │
└─────────────┘     │              │     └─────────────┘
                    │  (routing)   │     ┌─────────────┐
                    │              │────▶│   Queue 2   │
                    └──────────────┘     └─────────────┘
                           │             ┌─────────────┐
                           └────────────▶│   Queue 3   │
                                         └─────────────┘
```

## Основные типы Exchange

RabbitMQ предоставляет четыре встроенных типа обменников:

| Тип | Описание | Routing Key |
|-----|----------|-------------|
| **Direct** | Точное совпадение routing key | Обязателен |
| **Fanout** | Broadcast во все привязанные очереди | Игнорируется |
| **Topic** | Паттерн-матчинг с wildcards | Паттерн с `.` |
| **Headers** | Маршрутизация по заголовкам | Не используется |

Также существует **Default Exchange** - безымянный direct exchange.

## Сравнение типов Exchange

### 1. Direct Exchange

```
Producer ──▶ [Direct Exchange] ──routing_key="error"──▶ [Error Queue]
                               ──routing_key="info"───▶ [Info Queue]
```

**Когда использовать:**
- Отправка задач конкретным воркерам
- Логирование по уровням (error, warning, info)
- Любая точная маршрутизация

### 2. Fanout Exchange

```
Producer ──▶ [Fanout Exchange] ──▶ [Queue 1]
                               ──▶ [Queue 2]
                               ──▶ [Queue 3]
```

**Когда использовать:**
- Рассылка уведомлений всем подписчикам
- Инвалидация кэша на всех серверах
- Pub/Sub паттерн

### 3. Topic Exchange

```
Producer ──▶ [Topic Exchange] ──"order.created"────▶ [Order Queue]   (*.created)
                              ──"user.created"────▶ [Audit Queue]    (#)
                              ──"order.updated"───▶ [Order Queue]    (order.*)
```

**Когда использовать:**
- Маршрутизация по категориям событий
- Гибкая подписка на группы сообщений
- Микросервисная архитектура

### 4. Headers Exchange

```
Producer ──▶ [Headers Exchange] ──headers={type:pdf}────▶ [PDF Queue]
                                ──headers={type:image}──▶ [Image Queue]
```

**Когда использовать:**
- Маршрутизация по нескольким атрибутам
- Когда routing key неудобен
- Сложная логика фильтрации

## Атрибуты Exchange

При объявлении exchange можно указать следующие атрибуты:

```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Объявление exchange с атрибутами
channel.exchange_declare(
    exchange='my_exchange',      # Имя обменника
    exchange_type='direct',      # Тип: direct, fanout, topic, headers
    durable=True,                # Переживает перезапуск брокера
    auto_delete=False,           # Не удаляется при отсутствии привязок
    internal=False,              # Можно публиковать напрямую
    arguments=None               # Дополнительные аргументы
)
```

### Параметры:

| Параметр | Описание |
|----------|----------|
| `durable` | Exchange сохраняется при перезапуске RabbitMQ |
| `auto_delete` | Exchange удаляется, когда последняя очередь отвязывается |
| `internal` | Exchange используется только для exchange-to-exchange binding |
| `arguments` | Специальные аргументы (например, alternate-exchange) |

## Привязки (Bindings)

Binding связывает exchange с очередью и определяет правила маршрутизации:

```python
# Привязка очереди к exchange
channel.queue_bind(
    queue='my_queue',
    exchange='my_exchange',
    routing_key='my_key',        # Для direct и topic
    arguments=None               # Для headers exchange
)
```

## Alternate Exchange

Если сообщение не может быть маршрутизировано, его можно отправить в альтернативный exchange:

```python
# Объявление exchange с alternate exchange
channel.exchange_declare(
    exchange='main_exchange',
    exchange_type='direct',
    arguments={'alternate-exchange': 'unrouted_exchange'}
)

# Объявление альтернативного exchange (обычно fanout)
channel.exchange_declare(
    exchange='unrouted_exchange',
    exchange_type='fanout'
)
```

## Таблица выбора типа Exchange

| Сценарий | Рекомендуемый тип |
|----------|-------------------|
| Один producer - один consumer | Default или Direct |
| Балансировка нагрузки между воркерами | Direct |
| Broadcast всем подписчикам | Fanout |
| Подписка по категориям | Topic |
| Сложная фильтрация по атрибутам | Headers |
| Routing по нескольким критериям | Headers или Topic |

## Производительность

Типы exchange отсортированы по производительности (от быстрого к медленному):

1. **Fanout** - самый быстрый (нет логики маршрутизации)
2. **Direct** - быстрый (простое сравнение строк)
3. **Topic** - средний (паттерн-матчинг)
4. **Headers** - самый медленный (сравнение множества заголовков)

## Пример: комбинирование типов

```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# 1. Fanout для broadcast уведомлений
channel.exchange_declare(exchange='notifications', exchange_type='fanout')

# 2. Topic для событий системы
channel.exchange_declare(exchange='events', exchange_type='topic')

# 3. Direct для задач
channel.exchange_declare(exchange='tasks', exchange_type='direct')

# Очереди
channel.queue_declare(queue='email_service')
channel.queue_declare(queue='sms_service')
channel.queue_declare(queue='order_processor')
channel.queue_declare(queue='audit_log')

# Привязки
channel.queue_bind(queue='email_service', exchange='notifications')
channel.queue_bind(queue='sms_service', exchange='notifications')
channel.queue_bind(queue='order_processor', exchange='events', routing_key='order.*')
channel.queue_bind(queue='audit_log', exchange='events', routing_key='#')

connection.close()
```

## Best Practices

1. **Используйте durable exchange** для продакшена
2. **Называйте exchange осмысленно** - по назначению или домену
3. **Не создавайте слишком много exchange** - группируйте по логике
4. **Используйте alternate exchange** для обработки unmapped сообщений
5. **Выбирайте тип по потребностям**, а не "на всякий случай"
6. **Документируйте схему маршрутизации** для команды

## Заключение

Выбор правильного типа exchange - ключевое архитектурное решение:

- **Direct** - для простой точной маршрутизации
- **Fanout** - для broadcast/pub-sub
- **Topic** - для гибкой категоризации
- **Headers** - для сложной фильтрации

В следующих разделах мы подробно рассмотрим каждый тип exchange с примерами и use cases.
