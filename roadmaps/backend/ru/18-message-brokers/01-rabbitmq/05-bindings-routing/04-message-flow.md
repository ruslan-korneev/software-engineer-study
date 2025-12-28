# Поток сообщений

[prev: 03-routing-patterns](./03-routing-patterns.md) | [next: 01-acknowledgments](../06-message-handling/01-acknowledgments.md)

---

## Обзор потока сообщений в RabbitMQ

Понимание полного пути сообщения от producer до consumer — ключ к эффективной работе с RabbitMQ. В этом разделе мы детально рассмотрим каждый этап прохождения сообщения через систему.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Полный путь сообщения                                 │
│                                                                              │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────────┐        │
│  │ Producer │────>│ Exchange │────>│  Queue   │────>│   Consumer   │        │
│  │          │     │          │     │          │     │              │        │
│  │ 1.Create │     │ 2.Route  │     │ 3.Store  │     │ 4.Deliver    │        │
│  │   Msg    │     │   Msg    │     │   Msg    │     │   & Process  │        │
│  └──────────┘     └──────────┘     └──────────┘     └──────────────┘        │
│       │                │                │                   │               │
│       ▼                ▼                ▼                   ▼               │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────────┐        │
│  │Serialize │     │ Match    │     │ Persist  │     │   Ack/Nack   │        │
│  │& Publish │     │ Bindings │     │ (если    │     │   & Remove   │        │
│  │          │     │          │     │ durable) │     │              │        │
│  └──────────┘     └──────────┘     └──────────┘     └──────────────┘        │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Этап 1: Создание и публикация сообщения

### Жизненный цикл сообщения у Producer

```python
import pika
import json
import uuid
from datetime import datetime

def create_and_publish_message():
    """Демонстрация полного процесса создания и публикации сообщения."""

    # 1. Установка соединения
    connection = pika.BlockingConnection(
        pika.ConnectionParameters(
            host='localhost',
            port=5672,
            credentials=pika.PlainCredentials('guest', 'guest'),
            heartbeat=60
        )
    )

    # 2. Создание канала
    channel = connection.channel()

    # 3. Подготовка сообщения
    message_body = {
        'order_id': 12345,
        'items': ['item1', 'item2'],
        'total': 99.99,
        'timestamp': datetime.utcnow().isoformat()
    }

    # 4. Сериализация в bytes
    body = json.dumps(message_body).encode('utf-8')

    # 5. Подготовка свойств сообщения
    properties = pika.BasicProperties(
        content_type='application/json',
        content_encoding='utf-8',
        delivery_mode=2,  # Persistent message
        priority=5,
        correlation_id=str(uuid.uuid4()),
        message_id=str(uuid.uuid4()),
        timestamp=int(datetime.utcnow().timestamp()),
        headers={
            'source': 'order-service',
            'version': '1.0'
        }
    )

    # 6. Публикация сообщения
    channel.basic_publish(
        exchange='orders',
        routing_key='order.created',
        body=body,
        properties=properties,
        mandatory=True  # Вернуть сообщение, если не удалось доставить
    )

    print(f"Message published with ID: {properties.message_id}")

    connection.close()
```

### Структура сообщения RabbitMQ

```
┌─────────────────────────────────────────────────────────┐
│                    AMQP Message                         │
├─────────────────────────────────────────────────────────┤
│  HEADERS (Заголовки)                                    │
│  ┌─────────────────────────────────────────────────┐    │
│  │ content_type: application/json                  │    │
│  │ content_encoding: utf-8                         │    │
│  │ delivery_mode: 2 (persistent)                   │    │
│  │ priority: 5                                     │    │
│  │ correlation_id: uuid                            │    │
│  │ message_id: uuid                                │    │
│  │ timestamp: 1703677200                           │    │
│  │ headers: {source: order-service, version: 1.0} │    │
│  └─────────────────────────────────────────────────┘    │
├─────────────────────────────────────────────────────────┤
│  BODY (Тело сообщения)                                  │
│  ┌─────────────────────────────────────────────────┐    │
│  │ {"order_id": 12345, "items": [...], ...}        │    │
│  └─────────────────────────────────────────────────┘    │
├─────────────────────────────────────────────────────────┤
│  ROUTING INFO (Информация маршрутизации)                │
│  ┌─────────────────────────────────────────────────┐    │
│  │ exchange: orders                                │    │
│  │ routing_key: order.created                      │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

## Этап 2: Маршрутизация через Exchange

### Процесс маршрутизации

```
┌─────────────────────────────────────────────────────────────────┐
│                    Exchange Processing                           │
│                                                                  │
│   Incoming Message                                               │
│   routing_key: "order.created"                                   │
│         │                                                        │
│         ▼                                                        │
│   ┌─────────────────────────────────────────────┐               │
│   │           Exchange: orders (topic)           │               │
│   │                                              │               │
│   │   Bindings:                                  │               │
│   │   ┌─────────────────────────────────────┐   │               │
│   │   │ 1. queue: new_orders                │   │               │
│   │   │    pattern: order.created      ✓    │───┼──> new_orders │
│   │   ├─────────────────────────────────────┤   │               │
│   │   │ 2. queue: all_orders                │   │               │
│   │   │    pattern: order.#            ✓    │───┼──> all_orders │
│   │   ├─────────────────────────────────────┤   │               │
│   │   │ 3. queue: updates                   │   │               │
│   │   │    pattern: order.updated      ✗    │   │               │
│   │   └─────────────────────────────────────┘   │               │
│   └─────────────────────────────────────────────┘               │
│                                                                  │
│   Result: Message routed to 2 queues                             │
└─────────────────────────────────────────────────────────────────┘
```

### Код для трассировки маршрутизации

```python
import pika

def trace_routing(exchange: str, routing_key: str):
    """Трассировка маршрутизации сообщения."""

    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Включаем publisher confirms
    channel.confirm_delivery()

    # Обработчик возврата (если сообщение не доставлено)
    def on_return(channel, method, properties, body):
        print(f"Message returned!")
        print(f"  Exchange: {method.exchange}")
        print(f"  Routing Key: {method.routing_key}")
        print(f"  Reply Code: {method.reply_code}")
        print(f"  Reply Text: {method.reply_text}")

    channel.add_on_return_callback(on_return)

    # Публикуем с mandatory=True
    try:
        channel.basic_publish(
            exchange=exchange,
            routing_key=routing_key,
            body=b'test message',
            properties=pika.BasicProperties(delivery_mode=2),
            mandatory=True
        )
        print(f"Message sent to {exchange} with key {routing_key}")
    except pika.exceptions.UnroutableError:
        print("Message was not routed to any queue!")

    connection.close()
```

## Этап 3: Хранение в очереди

### Жизнь сообщения в очереди

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Queue Storage                                │
│                                                                      │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │                    Queue: orders_new                         │   │
│   │                                                              │   │
│   │   Properties:                                                │   │
│   │   - durable: true                                            │   │
│   │   - x-max-length: 10000                                      │   │
│   │   - x-message-ttl: 86400000 (24 hours)                       │   │
│   │   - x-dead-letter-exchange: dlx                              │   │
│   │                                                              │   │
│   │   ┌─────────────────────────────────────────────────────┐   │   │
│   │   │ Message Queue (FIFO)                                │   │   │
│   │   │                                                     │   │   │
│   │   │  [Msg1] -> [Msg2] -> [Msg3] -> [Msg4] -> ...       │   │   │
│   │   │    │                                                │   │   │
│   │   │    └── Ready for delivery                           │   │   │
│   │   └─────────────────────────────────────────────────────┘   │   │
│   │                                                              │   │
│   │   Stats:                                                     │   │
│   │   - Messages ready: 4                                        │   │
│   │   - Messages unacked: 2                                      │   │
│   │   - Memory usage: 1.2 MB                                     │   │
│   └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

### Мониторинг очереди

```python
import requests
from typing import Dict, Any

def get_queue_stats(queue_name: str, vhost: str = '/') -> Dict[str, Any]:
    """Получение статистики очереди."""

    # Кодируем vhost для URL
    import urllib.parse
    vhost_encoded = urllib.parse.quote(vhost, safe='')

    response = requests.get(
        f'http://localhost:15672/api/queues/{vhost_encoded}/{queue_name}',
        auth=('guest', 'guest')
    )

    if response.status_code == 200:
        data = response.json()
        return {
            'name': data['name'],
            'messages_ready': data.get('messages_ready', 0),
            'messages_unacked': data.get('messages_unacknowledged', 0),
            'messages_total': data.get('messages', 0),
            'consumers': data.get('consumers', 0),
            'memory': data.get('memory', 0),
            'message_stats': data.get('message_stats', {})
        }
    else:
        return {'error': f'Queue not found: {response.status_code}'}

# Использование
stats = get_queue_stats('orders_new')
print(f"Queue: {stats['name']}")
print(f"  Ready: {stats['messages_ready']}")
print(f"  Unacked: {stats['messages_unacked']}")
print(f"  Consumers: {stats['consumers']}")
```

## Этап 4: Доставка Consumer

### Процесс доставки

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Message Delivery                                │
│                                                                      │
│   Queue                           Consumer                           │
│   ┌─────────┐                    ┌─────────────────┐                │
│   │ [Msg1]  │──── Deliver ──────>│    Callback     │                │
│   │ [Msg2]  │                    │                 │                │
│   │ [Msg3]  │                    │  1. Receive     │                │
│   │   ...   │                    │  2. Process     │                │
│   └─────────┘                    │  3. Ack/Nack    │                │
│        ▲                         └────────┬────────┘                │
│        │                                  │                         │
│        │         Acknowledgement          │                         │
│        └──────────────────────────────────┘                         │
│                                                                      │
│   Delivery States:                                                   │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │ Ready ──> Unacked ──> Acked ──> Removed                     │   │
│   │                  └──> Nacked ──> Requeued/Dead Letter       │   │
│   └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

### Полный код Consumer с обработкой

```python
import pika
import json
import logging
from typing import Callable, Optional
from dataclasses import dataclass

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

@dataclass
class Message:
    """Обёртка для сообщения RabbitMQ."""
    body: dict
    routing_key: str
    delivery_tag: int
    message_id: Optional[str]
    correlation_id: Optional[str]
    headers: dict
    redelivered: bool


class MessageConsumer:
    """Consumer с полной обработкой жизненного цикла сообщений."""

    def __init__(self, host: str = 'localhost'):
        self.host = host
        self.connection = None
        self.channel = None

    def connect(self):
        """Установка соединения."""

        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters(
                host=self.host,
                heartbeat=60,
                blocked_connection_timeout=300
            )
        )
        self.channel = self.connection.channel()

        # Устанавливаем prefetch для честного распределения
        self.channel.basic_qos(prefetch_count=10)

        logger.info("Connected to RabbitMQ")

    def consume(self, queue: str, handler: Callable[[Message], bool]):
        """
        Начало потребления сообщений.

        Args:
            queue: Имя очереди
            handler: Функция обработки, возвращает True для ack, False для nack
        """

        def callback(ch, method, properties, body):
            # Создаём объект Message
            try:
                parsed_body = json.loads(body)
            except json.JSONDecodeError:
                parsed_body = {'raw': body.decode('utf-8', errors='replace')}

            message = Message(
                body=parsed_body,
                routing_key=method.routing_key,
                delivery_tag=method.delivery_tag,
                message_id=properties.message_id,
                correlation_id=properties.correlation_id,
                headers=properties.headers or {},
                redelivered=method.redelivered
            )

            logger.info(
                f"Received message: {message.message_id}, "
                f"routing_key: {message.routing_key}, "
                f"redelivered: {message.redelivered}"
            )

            try:
                # Обрабатываем сообщение
                success = handler(message)

                if success:
                    # Подтверждаем успешную обработку
                    ch.basic_ack(delivery_tag=method.delivery_tag)
                    logger.info(f"Message {message.message_id} acknowledged")
                else:
                    # Отклоняем и возвращаем в очередь
                    ch.basic_nack(
                        delivery_tag=method.delivery_tag,
                        requeue=not message.redelivered  # Не requeue повторно
                    )
                    logger.warning(f"Message {message.message_id} nacked")

            except Exception as e:
                logger.error(f"Error processing message: {e}")
                # Отклоняем при ошибке
                ch.basic_nack(
                    delivery_tag=method.delivery_tag,
                    requeue=False  # Отправляем в DLX
                )

        self.channel.basic_consume(
            queue=queue,
            on_message_callback=callback,
            auto_ack=False  # Ручное подтверждение
        )

        logger.info(f"Starting to consume from '{queue}'")

        try:
            self.channel.start_consuming()
        except KeyboardInterrupt:
            self.channel.stop_consuming()
            logger.info("Consumer stopped")
        finally:
            self.connection.close()


# Пример использования
def order_handler(message: Message) -> bool:
    """Обработчик заказов."""

    order_id = message.body.get('order_id')
    logger.info(f"Processing order: {order_id}")

    # Бизнес-логика
    try:
        # Имитация обработки
        if order_id and order_id > 0:
            logger.info(f"Order {order_id} processed successfully")
            return True
        else:
            logger.warning(f"Invalid order ID")
            return False
    except Exception as e:
        logger.error(f"Failed to process order: {e}")
        return False


if __name__ == '__main__':
    consumer = MessageConsumer()
    consumer.connect()
    consumer.consume('new_orders', order_handler)
```

## Полная диаграмма потока

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     COMPLETE MESSAGE FLOW DIAGRAM                            │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                          PRODUCER                                    │    │
│  │  1. Create message body (JSON/bytes)                                 │    │
│  │  2. Set properties (delivery_mode, headers, etc.)                    │    │
│  │  3. Publish to exchange with routing_key                             │    │
│  └───────────────────────────────┬─────────────────────────────────────┘    │
│                                  │                                          │
│                                  ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                          EXCHANGE                                    │    │
│  │  4. Receive message                                                  │    │
│  │  5. Evaluate routing_key against all bindings                        │    │
│  │  6. Route to matching queues (0, 1, or many)                         │    │
│  │  7. If no match && alternate-exchange -> route there                 │    │
│  │  8. If no match && mandatory -> return to producer                   │    │
│  └───────────────────────────────┬─────────────────────────────────────┘    │
│                                  │                                          │
│                    ┌─────────────┼─────────────┐                            │
│                    ▼             ▼             ▼                            │
│  ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐             │
│  │     QUEUE 1      │ │     QUEUE 2      │ │     QUEUE N      │             │
│  │  9. Store msg    │ │  9. Store msg    │ │  9. Store msg    │             │
│  │  10. Persist     │ │  10. Persist     │ │  10. Persist     │             │
│  │      (if durable)│ │      (if durable)│ │      (if durable)│             │
│  │  11. Check TTL   │ │  11. Check TTL   │ │  11. Check TTL   │             │
│  │  12. Check limit │ │  12. Check limit │ │  12. Check limit │             │
│  └────────┬─────────┘ └────────┬─────────┘ └────────┬─────────┘             │
│           │                    │                    │                       │
│           ▼                    ▼                    ▼                       │
│  ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐             │
│  │   CONSUMER 1     │ │   CONSUMER 2     │ │   CONSUMER N     │             │
│  │  13. Deliver msg │ │  13. Deliver msg │ │  13. Deliver msg │             │
│  │  14. Process     │ │  14. Process     │ │  14. Process     │             │
│  │  15. Ack/Nack    │ │  15. Ack/Nack    │ │  15. Ack/Nack    │             │
│  └──────────────────┘ └──────────────────┘ └──────────────────┘             │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                      ACKNOWLEDGEMENT OUTCOMES                        │    │
│  │                                                                      │    │
│  │  ACK:  Message removed from queue permanently                        │    │
│  │  NACK + requeue: Message returned to queue head                      │    │
│  │  NACK + no requeue: Message sent to DLX (if configured)              │    │
│  │  REJECT: Same as NACK for single message                             │    │
│  │  TIMEOUT: Message requeued (consumer disconnect)                     │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Трассировка сообщений

### Включение Firehose Tracer

```bash
# Включить трассировку для vhost
rabbitmqctl trace_on -p /

# Выключить трассировку
rabbitmqctl trace_off -p /
```

### Чтение трассировки

```python
import pika
import json

def trace_messages():
    """Чтение сообщений трассировки."""

    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Очередь для трассировки
    result = channel.queue_declare(queue='', exclusive=True)
    trace_queue = result.method.queue

    # Привязываем к amq.rabbitmq.trace
    channel.queue_bind(
        queue=trace_queue,
        exchange='amq.rabbitmq.trace',
        routing_key='#'  # Все сообщения
    )

    def callback(ch, method, properties, body):
        routing_key = method.routing_key

        # routing_key формат: publish.{exchange} или deliver.{queue}
        if routing_key.startswith('publish.'):
            action = 'PUBLISHED'
            target = routing_key.replace('publish.', '')
        else:
            action = 'DELIVERED'
            target = routing_key.replace('deliver.', '')

        print(f"[{action}] -> {target}")
        print(f"  Properties: {properties}")
        print(f"  Body: {body[:100]}...")
        print()

    channel.basic_consume(
        queue=trace_queue,
        on_message_callback=callback,
        auto_ack=True
    )

    print("Waiting for trace messages...")
    channel.start_consuming()
```

## Обработка ошибок в потоке

```python
import pika
from pika.exceptions import (
    AMQPConnectionError,
    AMQPChannelError,
    UnroutableError
)

class RobustPublisher:
    """Publisher с обработкой всех этапов ошибок."""

    def __init__(self):
        self.connection = None
        self.channel = None

    def connect(self):
        try:
            self.connection = pika.BlockingConnection(
                pika.ConnectionParameters('localhost')
            )
            self.channel = self.connection.channel()
            self.channel.confirm_delivery()
            return True
        except AMQPConnectionError as e:
            print(f"Connection failed: {e}")
            return False

    def publish(self, exchange: str, routing_key: str, body: bytes) -> bool:
        try:
            self.channel.basic_publish(
                exchange=exchange,
                routing_key=routing_key,
                body=body,
                properties=pika.BasicProperties(delivery_mode=2),
                mandatory=True
            )
            return True

        except UnroutableError:
            print(f"Message unroutable: {routing_key}")
            return False

        except AMQPChannelError as e:
            print(f"Channel error: {e}")
            self.reconnect()
            return False

        except AMQPConnectionError as e:
            print(f"Connection lost: {e}")
            self.reconnect()
            return False

    def reconnect(self):
        try:
            if self.connection and not self.connection.is_closed:
                self.connection.close()
        except:
            pass
        self.connect()
```

## Best Practices

1. **Всегда используйте manual acknowledgements** для гарантии обработки
2. **Устанавливайте prefetch_count** для контроля нагрузки
3. **Используйте persistent messages** для важных данных
4. **Настройте Dead Letter Exchange** для необработанных сообщений
5. **Мониторьте длину очередей** и unacked сообщения
6. **Используйте correlation_id** для трассировки запросов
7. **Логируйте все этапы** обработки сообщений

## Резюме

- Сообщение проходит путь: Producer -> Exchange -> Queue -> Consumer
- На каждом этапе возможны ошибки, которые нужно обрабатывать
- Publisher Confirms гарантируют доставку до брокера
- Consumer Acknowledgements гарантируют обработку
- Трассировка помогает отлаживать маршрутизацию
- Правильная обработка ошибок критична для надёжности системы

---

[prev: 03-routing-patterns](./03-routing-patterns.md) | [next: 01-acknowledgments](../06-message-handling/01-acknowledgments.md)
