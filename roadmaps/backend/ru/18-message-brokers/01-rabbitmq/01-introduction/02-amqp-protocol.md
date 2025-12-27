# Протокол AMQP

## Введение

**AMQP (Advanced Message Queuing Protocol)** — это открытый стандартный протокол прикладного уровня для передачи сообщений между приложениями. Он был разработан для обеспечения надёжной, безопасной и интероперабельной передачи сообщений.

RabbitMQ изначально создавался как реализация AMQP 0-9-1, что делает понимание этого протокола ключевым для эффективной работы с брокером.

---

## История AMQP

| Год | Событие |
|-----|---------|
| 2003 | Начало разработки в JPMorgan Chase |
| 2006 | Создание AMQP Working Group |
| 2008 | Выпуск AMQP 0-9-1 (используется RabbitMQ) |
| 2011 | Выпуск AMQP 1.0 (существенно отличается) |
| 2012 | AMQP 1.0 становится стандартом OASIS |

> **Важно:** RabbitMQ использует AMQP 0-9-1, а не AMQP 1.0. Это два существенно разных протокола!

---

## Архитектура AMQP 0-9-1

### Модель AMQ (Advanced Message Queuing)

```
┌────────────────────────────────────────────────────────────────────┐
│                         AMQP Broker                                │
│                                                                    │
│  ┌──────────────┐      ┌─────────────┐      ┌──────────────────┐  │
│  │              │      │             │      │                  │  │
│  │   Exchange   │─────▶│   Binding   │─────▶│      Queue       │  │
│  │              │      │ (routing    │      │                  │  │
│  └──────────────┘      │    key)     │      └──────────────────┘  │
│         ▲              └─────────────┘              │              │
│         │                                           │              │
└─────────│───────────────────────────────────────────│──────────────┘
          │                                           │
          │ publish                          consume  │
          │                                           ▼
   ┌──────────────┐                          ┌──────────────┐
   │   Producer   │                          │   Consumer   │
   └──────────────┘                          └──────────────┘
```

### Ключевые сущности

1. **Message (Сообщение)** — единица данных с заголовками и телом
2. **Exchange (Обменник)** — точка приёма сообщений
3. **Queue (Очередь)** — хранилище сообщений
4. **Binding (Привязка)** — связь между Exchange и Queue
5. **Routing Key** — ключ маршрутизации
6. **Virtual Host** — логическое разделение

---

## Структура AMQP-сообщения

### Компоненты сообщения

```python
import pika

# Полная структура сообщения AMQP
properties = pika.BasicProperties(
    # Метаданные доставки
    delivery_mode=2,        # 1 = transient, 2 = persistent
    priority=5,             # Приоритет (0-9)

    # Идентификация
    message_id='msg-123',   # Уникальный ID сообщения
    correlation_id='req-1', # Для связи запрос-ответ

    # Маршрутизация ответа
    reply_to='response_queue',

    # Тип содержимого
    content_type='application/json',
    content_encoding='utf-8',

    # Пользовательские заголовки
    headers={
        'x-retry-count': 0,
        'x-source': 'order-service'
    },

    # Время жизни
    expiration='60000',     # TTL в миллисекундах

    # Дополнительно
    timestamp=1703673600,   # Unix timestamp
    type='order.created',   # Тип сообщения
    user_id='guest',        # ID пользователя AMQP
    app_id='my-app'         # ID приложения
)

channel.basic_publish(
    exchange='orders',
    routing_key='order.new',
    body=b'{"order_id": 123}',
    properties=properties
)
```

### Фрейм AMQP

Сообщение передаётся в виде фреймов:

```
┌─────────────────────────────────────────────────────────────┐
│                      AMQP Frame                             │
├─────────────┬───────────────┬─────────────┬────────────────┤
│ Frame Type  │ Channel ID    │ Payload     │ Frame End      │
│ (1 byte)    │ (2 bytes)     │ (variable)  │ (1 byte: 0xCE) │
└─────────────┴───────────────┴─────────────┴────────────────┘
```

Типы фреймов:
- **Method frame** — команды протокола
- **Content header frame** — метаданные сообщения
- **Body frame** — тело сообщения
- **Heartbeat frame** — проверка соединения

---

## Типы Exchange в AMQP

### 1. Direct Exchange

Точное совпадение routing key.

```python
# Создание Direct Exchange
channel.exchange_declare(exchange='direct_logs', exchange_type='direct')

# Привязка очереди с routing key
channel.queue_bind(exchange='direct_logs', queue='error_logs', routing_key='error')
channel.queue_bind(exchange='direct_logs', queue='info_logs', routing_key='info')

# Публикация — попадёт только в error_logs
channel.basic_publish(
    exchange='direct_logs',
    routing_key='error',
    body='Error occurred!'
)
```

```
                    routing_key='error'
Publisher ──────▶ Direct Exchange ──────────▶ error_logs queue
                        │
                        │ routing_key='info'
                        └────────────────────▶ info_logs queue
```

### 2. Fanout Exchange

Игнорирует routing key, отправляет во все привязанные очереди.

```python
# Broadcast pattern
channel.exchange_declare(exchange='notifications', exchange_type='fanout')

# Все привязанные очереди получат сообщение
channel.queue_bind(exchange='notifications', queue='email_queue')
channel.queue_bind(exchange='notifications', queue='sms_queue')
channel.queue_bind(exchange='notifications', queue='push_queue')

# Отправка всем
channel.basic_publish(
    exchange='notifications',
    routing_key='',  # Игнорируется
    body='New notification!'
)
```

### 3. Topic Exchange

Паттерны с wildcards (* и #).

```python
channel.exchange_declare(exchange='topic_logs', exchange_type='topic')

# Паттерны привязки:
# * - ровно одно слово
# # - ноль или более слов

channel.queue_bind(exchange='topic_logs', queue='all_errors',
                   routing_key='*.error')          # user.error, order.error

channel.queue_bind(exchange='topic_logs', queue='user_all',
                   routing_key='user.#')           # user.created, user.updated.email

channel.queue_bind(exchange='topic_logs', queue='critical',
                   routing_key='#.critical.#')     # any.critical.any
```

```
Routing Key Examples:
  "user.created"       → matches "user.*", "user.#", "#"
  "order.item.added"   → matches "order.#", "#.added", "#"
  "stock.error"        → matches "*.error", "stock.#", "#"
```

### 4. Headers Exchange

Маршрутизация по заголовкам сообщения.

```python
channel.exchange_declare(exchange='headers_logs', exchange_type='headers')

# x-match: all — все заголовки должны совпадать
channel.queue_bind(
    exchange='headers_logs',
    queue='pdf_reports',
    arguments={
        'x-match': 'all',
        'format': 'pdf',
        'type': 'report'
    }
)

# x-match: any — достаточно одного совпадения
channel.queue_bind(
    exchange='headers_logs',
    queue='urgent_tasks',
    arguments={
        'x-match': 'any',
        'priority': 'high',
        'urgent': 'true'
    }
)

# Публикация с заголовками
channel.basic_publish(
    exchange='headers_logs',
    routing_key='',  # Игнорируется
    body='Report data',
    properties=pika.BasicProperties(
        headers={'format': 'pdf', 'type': 'report'}
    )
)
```

---

## Методы AMQP

### Классы и методы протокола

```python
# Connection класс
connection = pika.BlockingConnection(parameters)  # Connection.Open
connection.close()                                 # Connection.Close

# Channel класс
channel = connection.channel()                     # Channel.Open
channel.close()                                    # Channel.Close

# Exchange класс
channel.exchange_declare(...)                      # Exchange.Declare
channel.exchange_delete(...)                       # Exchange.Delete
channel.exchange_bind(...)                         # Exchange.Bind

# Queue класс
channel.queue_declare(...)                         # Queue.Declare
channel.queue_bind(...)                            # Queue.Bind
channel.queue_purge(...)                           # Queue.Purge
channel.queue_delete(...)                          # Queue.Delete

# Basic класс (работа с сообщениями)
channel.basic_publish(...)                         # Basic.Publish
channel.basic_consume(...)                         # Basic.Consume
channel.basic_ack(...)                             # Basic.Ack
channel.basic_nack(...)                            # Basic.Nack
channel.basic_reject(...)                          # Basic.Reject
channel.basic_qos(...)                             # Basic.Qos
channel.basic_get(...)                             # Basic.Get
channel.basic_cancel(...)                          # Basic.Cancel

# Tx класс (транзакции)
channel.tx_select()                                # Tx.Select
channel.tx_commit()                                # Tx.Commit
channel.tx_rollback()                              # Tx.Rollback

# Confirm класс (подтверждения публикации)
channel.confirm_delivery()                         # Confirm.Select
```

---

## Подтверждения и надёжность

### Consumer Acknowledgements

```python
def callback(ch, method, properties, body):
    try:
        process_message(body)
        # Успешная обработка — подтверждаем
        ch.basic_ack(delivery_tag=method.delivery_tag)
    except Exception as e:
        # Ошибка — отклоняем с возможностью requeue
        ch.basic_nack(
            delivery_tag=method.delivery_tag,
            requeue=True  # Вернуть в очередь
        )

# Отключаем auto_ack для ручного подтверждения
channel.basic_consume(
    queue='tasks',
    on_message_callback=callback,
    auto_ack=False  # Важно!
)
```

### Publisher Confirms

```python
# Включаем режим подтверждений
channel.confirm_delivery()

# Синхронная отправка с ожиданием подтверждения
try:
    channel.basic_publish(
        exchange='',
        routing_key='important_queue',
        body='Critical data',
        mandatory=True  # Вернуть, если нет маршрута
    )
    print("Сообщение подтверждено брокером")
except pika.exceptions.UnroutableError:
    print("Сообщение не доставлено")
```

---

## AMQP 0-9-1 vs AMQP 1.0

| Аспект | AMQP 0-9-1 | AMQP 1.0 |
|--------|------------|----------|
| **Модель** | AMQ Model (Exchange, Queue, Binding) | Peer-to-peer |
| **Exchange** | Часть протокола | Не определено |
| **Binding** | Часть протокола | Не определено |
| **Routing** | Встроено | На уровне приложения |
| **Совместимость** | RabbitMQ native | Через плагин |
| **Сложность** | Богаче, но сложнее | Проще, но меньше возможностей |

---

## Heartbeat и контроль соединения

```python
# Настройка heartbeat при подключении
parameters = pika.ConnectionParameters(
    host='localhost',
    heartbeat=60,              # Интервал проверки (секунды)
    blocked_connection_timeout=300,  # Таймаут блокировки
    socket_timeout=10,         # Таймаут сокета
    connection_attempts=3,     # Попытки подключения
    retry_delay=5              # Задержка между попытками
)

connection = pika.BlockingConnection(parameters)
```

---

## Best Practices

1. **Всегда используйте ручные подтверждения (ack)** для критичных данных
2. **Настраивайте heartbeat** для обнаружения разорванных соединений
3. **Используйте QoS (prefetch)** для контроля нагрузки
4. **Делайте сообщения персистентными** (`delivery_mode=2`)
5. **Объявляйте очереди как durable** для сохранения при перезапуске
6. **Используйте mandatory флаг** когда важна доставка
7. **Обрабатывайте возвраты** через `add_on_return_callback`

---

## Заключение

AMQP 0-9-1 — это мощный протокол, предоставляющий:
- Гибкую модель маршрутизации через Exchange
- Надёжную доставку с подтверждениями
- Богатый набор возможностей для enterprise-решений

Понимание AMQP критически важно для эффективного использования RabbitMQ и проектирования надёжных систем обмена сообщениями.
