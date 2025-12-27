# Что такое RabbitMQ?

## Введение

**RabbitMQ** — это один из самых популярных брокеров сообщений с открытым исходным кодом. Он выступает посредником между приложениями, позволяя им обмениваться сообщениями асинхронно, что делает системы более масштабируемыми, отказоустойчивыми и слабосвязанными.

RabbitMQ изначально реализует протокол **AMQP (Advanced Message Queuing Protocol)**, но также поддерживает другие протоколы: STOMP, MQTT, HTTP и WebSockets.

---

## История создания

RabbitMQ был создан компанией **Rabbit Technologies Ltd** (позже приобретённой VMware, а затем Pivotal Software) в 2007 году. Основные вехи:

| Год | Событие |
|-----|---------|
| 2007 | Первый релиз RabbitMQ |
| 2010 | Приобретение SpringSource (VMware) |
| 2013 | Переход под управление Pivotal Software |
| 2019 | Возврат под управление VMware |
| 2023 | Переход под управление Broadcom после приобретения VMware |

RabbitMQ написан на **Erlang** — языке, изначально разработанном для телекоммуникационных систем Ericsson. Это обеспечивает:
- Высокую отказоустойчивость
- Поддержку миллионов одновременных соединений
- Горячее обновление кода без остановки системы

---

## Архитектура RabbitMQ

### Основные компоненты

```
┌─────────────┐     ┌──────────────────────────────────────────┐     ┌─────────────┐
│  Producer   │────▶│              RabbitMQ Broker             │────▶│  Consumer   │
│ (Отправ.)   │     │  ┌──────────┐    ┌───────┐    ┌───────┐  │     │ (Получ.)    │
└─────────────┘     │  │ Exchange │───▶│Binding│───▶│ Queue │  │     └─────────────┘
                    │  └──────────┘    └───────┘    └───────┘  │
                    └──────────────────────────────────────────┘
```

### Компоненты системы

1. **Producer (Издатель)** — приложение, отправляющее сообщения
2. **Consumer (Потребитель)** — приложение, получающее сообщения
3. **Exchange (Обменник)** — принимает сообщения и маршрутизирует их в очереди
4. **Queue (Очередь)** — буфер для хранения сообщений
5. **Binding (Привязка)** — правило связи между Exchange и Queue
6. **Virtual Host (vhost)** — логическая изоляция ресурсов
7. **Connection** — TCP-соединение с брокером
8. **Channel** — виртуальное соединение внутри Connection

### Типы Exchange

```python
# Пример создания различных типов Exchange с использованием pika

import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Direct Exchange — точное совпадение routing key
channel.exchange_declare(exchange='direct_logs', exchange_type='direct')

# Fanout Exchange — отправка во все привязанные очереди
channel.exchange_declare(exchange='broadcast', exchange_type='fanout')

# Topic Exchange — паттерны routing key (*.error, logs.#)
channel.exchange_declare(exchange='topic_logs', exchange_type='topic')

# Headers Exchange — маршрутизация по заголовкам
channel.exchange_declare(exchange='headers_logs', exchange_type='headers')
```

---

## Основные преимущества RabbitMQ

### 1. Надёжность доставки сообщений

```python
# Подтверждение доставки (Publisher Confirms)
channel.confirm_delivery()

# Отправка с подтверждением
try:
    channel.basic_publish(
        exchange='',
        routing_key='task_queue',
        body='Important message',
        properties=pika.BasicProperties(
            delivery_mode=2,  # Персистентное сообщение
        ),
        mandatory=True
    )
    print("Сообщение доставлено")
except pika.exceptions.UnroutableError:
    print("Сообщение не доставлено")
```

### 2. Гибкая маршрутизация

- Поддержка сложных паттернов маршрутизации
- Возможность создания цепочек обменников
- Dead Letter Exchange для обработки недоставленных сообщений

### 3. Кластеризация и высокая доступность

```bash
# Создание кластера RabbitMQ
rabbitmqctl stop_app
rabbitmqctl join_cluster rabbit@node1
rabbitmqctl start_app

# Проверка статуса кластера
rabbitmqctl cluster_status
```

### 4. Богатый набор плагинов

- Management UI (веб-интерфейс управления)
- Shovel (репликация между брокерами)
- Federation (распределённые системы)
- STOMP, MQTT плагины

### 5. Поддержка множества языков

Официальные и community клиенты для:
- Python (pika, aio-pika)
- Java (RabbitMQ Java Client)
- .NET (RabbitMQ .NET Client)
- JavaScript (amqplib)
- Go, Ruby, PHP и многих других

---

## Когда использовать RabbitMQ

### Идеальные сценарии

| Сценарий | Описание |
|----------|----------|
| **Асинхронные задачи** | Обработка email, генерация отчётов, ресайз изображений |
| **Микросервисная архитектура** | Слабосвязанная коммуникация между сервисами |
| **Распределение нагрузки** | Work queues для балансировки задач |
| **Событийная архитектура** | Pub/Sub для уведомлений и событий |
| **RPC вызовы** | Синхронные запросы через очереди |

### Когда НЕ стоит использовать

- **Простые приложения** — избыточная сложность
- **Гигантские объёмы данных** — для стриминга лучше Kafka
- **Требуется долгосрочное хранение** — RabbitMQ не база данных
- **Строгий порядок сообщений** — гарантируется только в одной очереди

---

## Быстрый старт

### Установка через Docker

```bash
# Запуск RabbitMQ с management UI
docker run -d --hostname rabbitmq-host \
    --name rabbitmq \
    -p 5672:5672 \
    -p 15672:15672 \
    rabbitmq:3-management
```

### Простой пример Producer/Consumer

```python
# producer.py
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Создаём очередь
channel.queue_declare(queue='hello', durable=True)

# Отправляем сообщение
channel.basic_publish(
    exchange='',
    routing_key='hello',
    body='Hello, RabbitMQ!',
    properties=pika.BasicProperties(
        delivery_mode=2,  # Персистентное сообщение
    )
)

print(" [x] Сообщение отправлено")
connection.close()
```

```python
# consumer.py
import pika

def callback(ch, method, properties, body):
    print(f" [x] Получено: {body.decode()}")
    ch.basic_ack(delivery_tag=method.delivery_tag)

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

channel.queue_declare(queue='hello', durable=True)
channel.basic_qos(prefetch_count=1)  # Fair dispatch
channel.basic_consume(queue='hello', on_message_callback=callback)

print(' [*] Ожидание сообщений. Для выхода нажмите CTRL+C')
channel.start_consuming()
```

---

## Best Practices

1. **Всегда используйте персистентные очереди и сообщения** для важных данных
2. **Настраивайте prefetch_count** для контроля нагрузки на консьюмеров
3. **Используйте подтверждения (ack)** для гарантии обработки
4. **Мониторьте метрики** через Management API
5. **Настраивайте Dead Letter Exchange** для обработки ошибок
6. **Используйте Connection pooling** в production
7. **Разделяйте окружения через Virtual Hosts**

---

## Заключение

RabbitMQ — мощный и зрелый брокер сообщений, который отлично подходит для:
- Традиционных очередей задач
- Микросервисной архитектуры
- Сложной маршрутизации сообщений
- Интеграции разнородных систем

Благодаря богатой экосистеме, отличной документации и активному сообществу, RabbitMQ остаётся одним из лучших выборов для асинхронной коммуникации в современных приложениях.
