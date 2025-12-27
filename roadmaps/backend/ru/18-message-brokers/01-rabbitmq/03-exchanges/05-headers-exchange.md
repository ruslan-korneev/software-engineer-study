# Headers Exchange

## Введение

Headers Exchange - это тип обменника в RabbitMQ, который маршрутизирует сообщения на основе **заголовков (headers)** сообщения, а не routing key. Это позволяет создавать сложные правила маршрутизации на основе множества атрибутов.

## Принцип работы

```
┌──────────┐
│ Producer │
└────┬─────┘
     │
     │ Message с headers:
     │ {
     │   "format": "pdf",
     │   "type": "report",
     │   "priority": "high"
     │ }
     ▼
┌────────────────────┐
│  Headers Exchange  │
└─────────┬──────────┘
          │
          │ Сравнение headers сообщения
          │ с headers привязок
          │
    ┌─────┼─────┬─────────────┐
    │     │     │             │
    ▼     ▼     ▼             ▼
┌───────┐┌───────┐┌───────┐┌───────┐
│format:││format:││type:  ││priority│
│pdf    ││excel  ││report ││:high  │
└───────┘└───────┘└───────┘└───────┘
   ✓        ✗        ✓         ✓
```

## Ключевое отличие от других Exchange

| Exchange | Критерий маршрутизации |
|----------|------------------------|
| Direct | Точное совпадение routing_key |
| Topic | Паттерн routing_key |
| Fanout | Broadcast всем |
| **Headers** | Атрибуты в headers сообщения |

## Параметр x-match

При создании привязки с Headers Exchange указывается специальный аргумент `x-match`:

| Значение | Описание |
|----------|----------|
| `all` | Все указанные headers должны совпадать (AND) |
| `any` | Хотя бы один header должен совпадать (OR) |

```python
# x-match: all - ВСЕ заголовки должны совпасть
channel.queue_bind(
    exchange='headers_ex',
    queue='queue1',
    arguments={
        'x-match': 'all',
        'format': 'pdf',
        'type': 'report'
    }
)

# x-match: any - ЛЮБОЙ заголовок должен совпасть
channel.queue_bind(
    exchange='headers_ex',
    queue='queue2',
    arguments={
        'x-match': 'any',
        'format': 'pdf',
        'type': 'report'
    }
)
```

## Схема Headers Exchange

```
Message headers: { format: "pdf", type: "report" }

┌─────────────────────────────────────────────────────────────┐
│                     Headers Exchange                        │
└─────────────────────────────────────────────────────────────┘
                              │
         ┌────────────────────┼────────────────────┐
         │                    │                    │
         ▼                    ▼                    ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│ x-match: all    │  │ x-match: any    │  │ x-match: all    │
│ format: pdf     │  │ format: excel   │  │ format: pdf     │
│ type: report    │  │ type: report    │  │ priority: high  │
└────────┬────────┘  └────────┬────────┘  └────────┬────────┘
         │                    │                    │
      ✓ MATCH              ✓ MATCH              ✗ NO MATCH
   (all совпали)      (type совпал)       (priority нет)
         │                    │
         ▼                    ▼
    [Queue PDF]          [Queue Reports]
```

## Базовый пример

### Publisher

```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Объявление Headers Exchange
channel.exchange_declare(
    exchange='document_processor',
    exchange_type='headers',
    durable=True
)

# Отправка сообщения с headers
message = "Содержимое PDF отчёта"

channel.basic_publish(
    exchange='document_processor',
    routing_key='',  # routing_key игнорируется!
    body=message,
    properties=pika.BasicProperties(
        headers={
            'format': 'pdf',
            'type': 'report',
            'department': 'finance'
        },
        delivery_mode=2
    )
)

print(" [x] Отправлено сообщение с headers")
connection.close()
```

### Consumer

```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

channel.exchange_declare(
    exchange='document_processor',
    exchange_type='headers',
    durable=True
)

# Создаём очередь для PDF документов
channel.queue_declare(queue='pdf_processor', durable=True)

# Привязка с x-match: all
channel.queue_bind(
    exchange='document_processor',
    queue='pdf_processor',
    arguments={
        'x-match': 'all',      # ВСЕ headers должны совпасть
        'format': 'pdf'         # Только PDF файлы
    }
)

def callback(ch, method, properties, body):
    headers = properties.headers
    print(f" [x] Получен PDF документ")
    print(f"     Headers: {headers}")
    print(f"     Body: {body.decode()}")
    ch.basic_ack(delivery_tag=method.delivery_tag)

channel.basic_consume(
    queue='pdf_processor',
    on_message_callback=callback
)

print(' [*] Ожидание PDF документов...')
channel.start_consuming()
```

## Use Case 1: Обработка документов

### Архитектура

```
┌─────────────────┐
│  Document API   │
└────────┬────────┘
         │
         │ headers: {format, type, size, priority}
         ▼
┌─────────────────────┐
│  Headers Exchange   │
│  "documents"        │
└──────────┬──────────┘
           │
    ┌──────┼──────┬──────────────────┐
    │      │      │                  │
    ▼      ▼      ▼                  ▼
┌───────┐┌───────┐┌───────┐    ┌───────────┐
│ PDF   ││ Excel ││ Image │    │ Priority  │
│Processor│Processor│Processor│ │ Processor │
│format:││format:││format:│    │priority:  │
│pdf    ││xlsx   ││image  │    │high       │
└───────┘└───────┘└───────┘    └───────────┘
```

### Реализация

```python
import pika
import json
from enum import Enum

class DocumentFormat(str, Enum):
    PDF = 'pdf'
    XLSX = 'xlsx'
    DOCX = 'docx'
    IMAGE = 'image'

class Priority(str, Enum):
    LOW = 'low'
    NORMAL = 'normal'
    HIGH = 'high'

class DocumentPublisher:
    """Издатель документов с маршрутизацией по headers"""

    def __init__(self, host='localhost'):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters(host)
        )
        self.channel = self.connection.channel()

        self.channel.exchange_declare(
            exchange='documents',
            exchange_type='headers',
            durable=True
        )

    def publish(self, document: bytes, format: DocumentFormat,
                doc_type: str, priority: Priority = Priority.NORMAL,
                **extra_headers):
        """Публикация документа с метаданными в headers"""

        headers = {
            'format': format.value,
            'type': doc_type,
            'priority': priority.value,
            'size': len(document),
            **extra_headers
        }

        self.channel.basic_publish(
            exchange='documents',
            routing_key='',  # Игнорируется
            body=document,
            properties=pika.BasicProperties(
                headers=headers,
                delivery_mode=2,
                content_type=self._get_content_type(format)
            )
        )

        print(f"Документ опубликован: {format.value}, {doc_type}")

    def _get_content_type(self, format: DocumentFormat) -> str:
        mapping = {
            DocumentFormat.PDF: 'application/pdf',
            DocumentFormat.XLSX: 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
            DocumentFormat.DOCX: 'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
            DocumentFormat.IMAGE: 'image/png'
        }
        return mapping.get(format, 'application/octet-stream')

    def close(self):
        self.connection.close()


class DocumentProcessor:
    """Обработчик документов определённого типа"""

    def __init__(self, processor_name: str, binding_headers: dict,
                 x_match: str = 'all', host='localhost'):
        self.name = processor_name

        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters(host)
        )
        self.channel = self.connection.channel()

        self.channel.exchange_declare(
            exchange='documents',
            exchange_type='headers',
            durable=True
        )

        # Создаём очередь
        queue_name = f'{processor_name}_queue'
        self.channel.queue_declare(queue=queue_name, durable=True)

        # Привязка с headers
        arguments = {'x-match': x_match, **binding_headers}
        self.channel.queue_bind(
            exchange='documents',
            queue=queue_name,
            arguments=arguments
        )

        self.queue_name = queue_name
        self.channel.basic_qos(prefetch_count=1)

        print(f"[{processor_name}] Привязка: {arguments}")

    def process(self, handler):
        """Начать обработку документов"""

        def callback(ch, method, properties, body):
            headers = properties.headers
            print(f"[{self.name}] Получен документ: {headers}")

            try:
                handler(body, headers)
                ch.basic_ack(delivery_tag=method.delivery_tag)
            except Exception as e:
                print(f"[{self.name}] Ошибка: {e}")
                ch.basic_nack(delivery_tag=method.delivery_tag, requeue=True)

        self.channel.basic_consume(
            queue=self.queue_name,
            on_message_callback=callback
        )

        print(f"[{self.name}] Запущен")
        self.channel.start_consuming()


# Пример использования
if __name__ == "__main__":
    # Создание обработчиков

    # 1. PDF Processor - только PDF файлы
    # pdf_processor = DocumentProcessor(
    #     'pdf_processor',
    #     {'format': 'pdf'},
    #     x_match='all'
    # )

    # 2. High Priority Processor - только высокий приоритет
    # priority_processor = DocumentProcessor(
    #     'priority_processor',
    #     {'priority': 'high'},
    #     x_match='all'
    # )

    # 3. Finance Reports - PDF отчёты финансового отдела
    # finance_processor = DocumentProcessor(
    #     'finance_processor',
    #     {'format': 'pdf', 'type': 'report', 'department': 'finance'},
    #     x_match='all'
    # )

    # Публикация
    publisher = DocumentPublisher()
    publisher.publish(
        b'PDF content here',
        DocumentFormat.PDF,
        'report',
        Priority.HIGH,
        department='finance'
    )
    publisher.close()
```

## Use Case 2: Мультикритериальная маршрутизация

### x-match: any - маршрутизация по ЛЮБОМУ критерию

```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

channel.exchange_declare(
    exchange='alerts',
    exchange_type='headers',
    durable=True
)

# Очередь для критических алертов ИЛИ алертов безопасности
channel.queue_declare(queue='critical_or_security', durable=True)

channel.queue_bind(
    exchange='alerts',
    queue='critical_or_security',
    arguments={
        'x-match': 'any',           # ЛЮБОЙ из headers
        'severity': 'critical',      # Критические
        'category': 'security'       # ИЛИ безопасность
    }
)

# Этот алерт попадёт в очередь (severity=critical)
channel.basic_publish(
    exchange='alerts',
    routing_key='',
    body='Database connection failed',
    properties=pika.BasicProperties(
        headers={
            'severity': 'critical',
            'category': 'database',
            'source': 'app1'
        }
    )
)

# Этот тоже попадёт (category=security)
channel.basic_publish(
    exchange='alerts',
    routing_key='',
    body='Unauthorized access attempt',
    properties=pika.BasicProperties(
        headers={
            'severity': 'warning',
            'category': 'security',
            'source': 'firewall'
        }
    )
)

# Этот НЕ попадёт (нет совпадений)
channel.basic_publish(
    exchange='alerts',
    routing_key='',
    body='CPU usage high',
    properties=pika.BasicProperties(
        headers={
            'severity': 'warning',
            'category': 'performance',
            'source': 'monitoring'
        }
    )
)
```

## Use Case 3: Комплексная система уведомлений

```python
import pika
import json
from typing import List, Dict, Any

class NotificationRouter:
    """
    Роутер уведомлений с комплексной маршрутизацией.

    Уведомления маршрутизируются по комбинации:
    - channel: email, sms, push, slack
    - priority: low, normal, high, urgent
    - category: order, payment, user, system
    """

    def __init__(self, host='localhost'):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters(host)
        )
        self.channel = self.connection.channel()

        self.channel.exchange_declare(
            exchange='notifications',
            exchange_type='headers',
            durable=True
        )

        self._setup_queues()

    def _setup_queues(self):
        """Настройка очередей с разными правилами маршрутизации"""

        # 1. Email для всех обычных уведомлений
        self.channel.queue_declare(queue='email_normal', durable=True)
        self.channel.queue_bind(
            exchange='notifications',
            queue='email_normal',
            arguments={
                'x-match': 'all',
                'channel': 'email'
            }
        )

        # 2. SMS только для urgent
        self.channel.queue_declare(queue='sms_urgent', durable=True)
        self.channel.queue_bind(
            exchange='notifications',
            queue='sms_urgent',
            arguments={
                'x-match': 'all',
                'channel': 'sms',
                'priority': 'urgent'
            }
        )

        # 3. Slack для system ИЛИ urgent
        self.channel.queue_declare(queue='slack_alerts', durable=True)
        self.channel.queue_bind(
            exchange='notifications',
            queue='slack_alerts',
            arguments={
                'x-match': 'any',
                'category': 'system',
                'priority': 'urgent'
            }
        )

        # 4. Все платёжные уведомления (для аудита)
        self.channel.queue_declare(queue='payment_audit', durable=True)
        self.channel.queue_bind(
            exchange='notifications',
            queue='payment_audit',
            arguments={
                'x-match': 'all',
                'category': 'payment'
            }
        )

    def send(self, message: str, channel: str, priority: str,
             category: str, **extra):
        """Отправка уведомления"""

        notification = {
            'message': message,
            'metadata': extra
        }

        self.channel.basic_publish(
            exchange='notifications',
            routing_key='',
            body=json.dumps(notification),
            properties=pika.BasicProperties(
                headers={
                    'channel': channel,
                    'priority': priority,
                    'category': category,
                    **extra
                },
                delivery_mode=2
            )
        )

    def close(self):
        self.connection.close()


# Использование
router = NotificationRouter()

# Email уведомление -> email_normal
router.send(
    "Ваш заказ отправлен",
    channel='email',
    priority='normal',
    category='order',
    user_id='123'
)

# Urgent SMS -> sms_urgent
router.send(
    "Подтвердите платёж",
    channel='sms',
    priority='urgent',
    category='payment',
    user_id='123'
)

# System alert -> slack_alerts (category=system)
router.send(
    "Database backup completed",
    channel='internal',
    priority='low',
    category='system'
)

router.close()
```

## Специальные заголовки

Некоторые заголовки имеют специальное значение и **игнорируются** при матчинге:

| Header | Описание |
|--------|----------|
| `x-match` | Тип сравнения (all/any) |
| `x-*` | Все заголовки с префиксом x- игнорируются |

```python
# x-custom НЕ участвует в матчинге
arguments = {
    'x-match': 'all',
    'x-custom': 'ignored',  # Игнорируется!
    'format': 'pdf'          # Участвует в матчинге
}
```

## Сравнение с Topic Exchange

Когда использовать Headers вместо Topic:

| Критерий | Topic | Headers |
|----------|-------|---------|
| Один атрибут | Да | Избыточно |
| Несколько атрибутов | Сложно | Да |
| Числовые значения | Нет | Да |
| Логика AND | Сложно | x-match: all |
| Логика OR | Невозможно | x-match: any |
| Производительность | Лучше | Хуже |

```python
# Topic: один атрибут в routing_key
channel.queue_bind(routing_key='order.*.created')

# Headers: несколько атрибутов
channel.queue_bind(arguments={
    'x-match': 'all',
    'entity': 'order',
    'action': 'created',
    'region': 'eu',
    'priority': 'high'
})
```

## Производительность

Headers Exchange - **самый медленный** тип:

```
Сравнение производительности:
- Fanout:  ~100%  (базовая линия)
- Direct:  ~95%
- Topic:   ~70%
- Headers: ~40-50%
```

Причина: сравнение множества пар ключ-значение для каждого сообщения.

### Рекомендации по оптимизации:

1. Используйте минимальное количество headers в привязках
2. Избегайте большого количества очередей с разными привязками
3. Рассмотрите альтернативы (Topic с составным routing key)

## Best Practices

### 1. Стандартизируйте имена headers

```python
class HeaderNames:
    """Константы для имён headers"""
    FORMAT = 'format'
    PRIORITY = 'priority'
    CATEGORY = 'category'
    SOURCE = 'source'
    VERSION = 'version'

# Использование
headers = {
    HeaderNames.FORMAT: 'pdf',
    HeaderNames.PRIORITY: 'high'
}
```

### 2. Документируйте схему headers

```python
"""
Headers Schema:

Required headers:
- format: pdf | xlsx | docx | image
- type: report | invoice | notification

Optional headers:
- priority: low | normal | high | urgent (default: normal)
- department: string
- version: integer
"""
```

### 3. Валидация headers

```python
def validate_headers(headers: dict) -> bool:
    """Валидация обязательных headers"""
    required = ['format', 'type']

    for header in required:
        if header not in headers:
            raise ValueError(f"Missing required header: {header}")

    valid_formats = ['pdf', 'xlsx', 'docx', 'image']
    if headers.get('format') not in valid_formats:
        raise ValueError(f"Invalid format: {headers.get('format')}")

    return True
```

## Частые ошибки

### 1. Забыли x-match

```python
# Ошибка: без x-match используется 'all' по умолчанию
channel.queue_bind(
    exchange='headers_ex',
    queue='q',
    arguments={
        'format': 'pdf',
        'type': 'report'
    }
)
# Это эквивалентно x-match: all

# Лучше явно указывать
arguments = {
    'x-match': 'all',  # Явно!
    'format': 'pdf',
    'type': 'report'
}
```

### 2. Использование x-* headers для матчинга

```python
# Ошибка: x-custom игнорируется
arguments = {
    'x-match': 'all',
    'x-custom': 'value',  # НЕ участвует в матчинге!
    'format': 'pdf'
}
```

### 3. Путаница с типами значений

```python
# Headers сравниваются как строки!
# Это НЕ совпадёт:

# Привязка
arguments = {'x-match': 'all', 'count': 10}

# Сообщение
headers = {'count': '10'}  # Строка != число

# Решение: используйте строки везде
arguments = {'x-match': 'all', 'count': '10'}
headers = {'count': '10'}
```

## Заключение

Headers Exchange - специализированный инструмент для сложной маршрутизации:

**Используйте когда:**
- Нужна маршрутизация по нескольким атрибутам
- Требуется логика OR (x-match: any)
- Routing key неудобен для выражения критериев
- Маршрутизация документов/файлов по метаданным

**Не используйте когда:**
- Достаточно одного атрибута (Direct/Topic)
- Критична производительность
- Простая категоризация (Topic лучше)
