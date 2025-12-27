# Serverless (Бессерверная архитектура)

## Что такое Serverless?

Serverless (бессерверная архитектура) — это модель облачных вычислений, при которой облачный провайдер динамически управляет выделением и масштабированием серверных ресурсов. Разработчики пишут и развёртывают код, не беспокоясь об инфраструктуре.

Несмотря на название, серверы всё ещё существуют — просто ими управляет провайдер.

```
┌─────────────────────────────────────────────────────────────────┐
│                     Serverless Architecture                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│    Событие          ┌─────────────────┐        Результат        │
│   (HTTP, Queue,     │                 │       (Response,        │
│    Schedule,   ────►│    Function     │────►   Database,        │
│    Database)        │    (Lambda)     │        Queue)           │
│                     └─────────────────┘                         │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │                   Облачный провайдер                     │  │
│   │  • Автоматическое масштабирование                       │  │
│   │  • Оплата за использование                              │  │
│   │  • Управление инфраструктурой                           │  │
│   │  • Высокая доступность                                  │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Ключевые концепции

### Function as a Service (FaaS)

FaaS — основа serverless. Код выполняется в ответ на события.

```python
# AWS Lambda пример
import json
import boto3
from datetime import datetime


def lambda_handler(event, context):
    """
    AWS Lambda функция для обработки HTTP запроса.

    event: данные события (HTTP запрос, сообщение из очереди и т.д.)
    context: информация о среде выполнения
    """

    # Извлекаем данные из запроса
    http_method = event.get('httpMethod', 'GET')
    path = event.get('path', '/')
    body = json.loads(event.get('body', '{}')) if event.get('body') else {}

    # Бизнес-логика
    if http_method == 'POST' and path == '/users':
        return create_user(body)
    elif http_method == 'GET' and path.startswith('/users/'):
        user_id = path.split('/')[-1]
        return get_user(user_id)
    else:
        return {
            'statusCode': 404,
            'body': json.dumps({'error': 'Not found'})
        }


def create_user(data: dict) -> dict:
    """Создание пользователя"""
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table('users')

    user = {
        'id': data.get('id'),
        'email': data.get('email'),
        'name': data.get('name'),
        'created_at': datetime.now().isoformat()
    }

    table.put_item(Item=user)

    return {
        'statusCode': 201,
        'headers': {'Content-Type': 'application/json'},
        'body': json.dumps(user)
    }


def get_user(user_id: str) -> dict:
    """Получение пользователя"""
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table('users')

    response = table.get_item(Key={'id': user_id})

    if 'Item' not in response:
        return {
            'statusCode': 404,
            'body': json.dumps({'error': 'User not found'})
        }

    return {
        'statusCode': 200,
        'headers': {'Content-Type': 'application/json'},
        'body': json.dumps(response['Item'])
    }
```

### Backend as a Service (BaaS)

BaaS — готовые сервисы для типичных backend-задач.

```python
# Пример использования Firebase (BaaS) с Python
import firebase_admin
from firebase_admin import credentials, firestore, auth


# Инициализация Firebase
cred = credentials.Certificate('service-account.json')
firebase_admin.initialize_app(cred)

db = firestore.client()


class UserService:
    """Сервис пользователей на Firebase"""

    def __init__(self):
        self.collection = db.collection('users')

    def create_user(self, email: str, password: str, name: str) -> dict:
        """Создание пользователя с аутентификацией"""
        # Firebase Auth создаёт пользователя
        user_record = auth.create_user(
            email=email,
            password=password,
            display_name=name
        )

        # Сохраняем дополнительные данные в Firestore
        user_data = {
            'uid': user_record.uid,
            'email': email,
            'name': name,
            'created_at': firestore.SERVER_TIMESTAMP
        }
        self.collection.document(user_record.uid).set(user_data)

        return user_data

    def get_user(self, uid: str) -> dict:
        """Получение пользователя"""
        doc = self.collection.document(uid).get()
        if doc.exists:
            return doc.to_dict()
        return None

    def update_user(self, uid: str, data: dict) -> dict:
        """Обновление пользователя"""
        self.collection.document(uid).update(data)
        return self.get_user(uid)

    def delete_user(self, uid: str) -> None:
        """Удаление пользователя"""
        auth.delete_user(uid)
        self.collection.document(uid).delete()

    def list_users(self, limit: int = 100) -> list:
        """Список пользователей"""
        docs = self.collection.limit(limit).stream()
        return [doc.to_dict() for doc in docs]
```

## Типы событий в Serverless

### HTTP события (API Gateway)

```python
# AWS Lambda с API Gateway
import json


def api_handler(event, context):
    """Обработчик HTTP запросов"""

    # Маршрутизация
    routes = {
        ('GET', '/products'): list_products,
        ('GET', '/products/{id}'): get_product,
        ('POST', '/products'): create_product,
        ('PUT', '/products/{id}'): update_product,
        ('DELETE', '/products/{id}'): delete_product,
    }

    method = event['httpMethod']
    path = event['resource']

    handler = routes.get((method, path))
    if handler:
        return handler(event)

    return {
        'statusCode': 404,
        'body': json.dumps({'error': 'Route not found'})
    }


def list_products(event):
    """GET /products"""
    # Параметры запроса
    query_params = event.get('queryStringParameters') or {}
    category = query_params.get('category')
    limit = int(query_params.get('limit', 10))

    # Получение из DynamoDB
    products = fetch_products(category=category, limit=limit)

    return {
        'statusCode': 200,
        'headers': {
            'Content-Type': 'application/json',
            'Access-Control-Allow-Origin': '*'
        },
        'body': json.dumps(products)
    }


def get_product(event):
    """GET /products/{id}"""
    product_id = event['pathParameters']['id']
    product = fetch_product_by_id(product_id)

    if not product:
        return {
            'statusCode': 404,
            'body': json.dumps({'error': 'Product not found'})
        }

    return {
        'statusCode': 200,
        'body': json.dumps(product)
    }


def create_product(event):
    """POST /products"""
    body = json.loads(event['body'])

    # Валидация
    if not body.get('name') or not body.get('price'):
        return {
            'statusCode': 400,
            'body': json.dumps({'error': 'Name and price are required'})
        }

    product = save_product(body)

    return {
        'statusCode': 201,
        'body': json.dumps(product)
    }
```

### События из очередей (SQS/SNS)

```python
# Обработка сообщений из SQS
import json
import boto3


def sqs_handler(event, context):
    """Обработчик сообщений SQS"""

    processed = 0
    failed = 0

    for record in event['Records']:
        try:
            # Парсим сообщение
            message = json.loads(record['body'])
            message_type = message.get('type')

            # Обработка в зависимости от типа
            if message_type == 'order.created':
                process_new_order(message['data'])
            elif message_type == 'order.cancelled':
                process_order_cancellation(message['data'])
            elif message_type == 'user.registered':
                send_welcome_email(message['data'])

            processed += 1

        except Exception as e:
            print(f"Error processing message: {e}")
            failed += 1
            # Сообщение вернётся в очередь для повторной обработки

    return {
        'processed': processed,
        'failed': failed
    }


def process_new_order(data: dict):
    """Обработка нового заказа"""
    order_id = data['order_id']
    items = data['items']

    # Резервируем товары на складе
    for item in items:
        reserve_inventory(item['product_id'], item['quantity'])

    # Отправляем уведомление
    send_notification(
        user_id=data['user_id'],
        message=f"Order {order_id} confirmed"
    )


# Публикация сообщений в SNS
def publish_event(topic_arn: str, event_type: str, data: dict):
    """Публикация события в SNS"""
    sns = boto3.client('sns')

    message = {
        'type': event_type,
        'data': data,
        'timestamp': datetime.now().isoformat()
    }

    sns.publish(
        TopicArn=topic_arn,
        Message=json.dumps(message),
        MessageAttributes={
            'event_type': {
                'DataType': 'String',
                'StringValue': event_type
            }
        }
    )
```

### События по расписанию (CloudWatch Events)

```python
# Запланированные задачи
import boto3
from datetime import datetime, timedelta


def scheduled_cleanup(event, context):
    """
    Ежедневная очистка устаревших данных.
    Триггер: CloudWatch Events (cron: 0 2 * * ? *)
    """
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table('sessions')

    # Удаляем сессии старше 30 дней
    cutoff_date = (datetime.now() - timedelta(days=30)).isoformat()

    # Сканируем и удаляем устаревшие записи
    response = table.scan(
        FilterExpression='created_at < :cutoff',
        ExpressionAttributeValues={':cutoff': cutoff_date}
    )

    deleted_count = 0
    for item in response['Items']:
        table.delete_item(Key={'session_id': item['session_id']})
        deleted_count += 1

    print(f"Deleted {deleted_count} expired sessions")

    return {'deleted': deleted_count}


def scheduled_report(event, context):
    """
    Еженедельный отчёт.
    Триггер: CloudWatch Events (cron: 0 9 ? * MON *)
    """
    # Сбор статистики
    stats = collect_weekly_stats()

    # Генерация отчёта
    report = generate_report(stats)

    # Отправка по email через SES
    ses = boto3.client('ses')
    ses.send_email(
        Source='reports@example.com',
        Destination={'ToAddresses': ['admin@example.com']},
        Message={
            'Subject': {'Data': f'Weekly Report - {datetime.now().strftime("%Y-%m-%d")}'},
            'Body': {'Html': {'Data': report}}
        }
    )

    return {'status': 'sent'}
```

### События из базы данных (DynamoDB Streams)

```python
# Реакция на изменения в DynamoDB
import json


def dynamodb_stream_handler(event, context):
    """Обработчик DynamoDB Streams"""

    for record in event['Records']:
        event_name = record['eventName']  # INSERT, MODIFY, REMOVE

        if event_name == 'INSERT':
            # Новая запись
            new_image = record['dynamodb']['NewImage']
            handle_new_record(deserialize_dynamodb(new_image))

        elif event_name == 'MODIFY':
            # Изменение записи
            old_image = record['dynamodb']['OldImage']
            new_image = record['dynamodb']['NewImage']
            handle_modified_record(
                old=deserialize_dynamodb(old_image),
                new=deserialize_dynamodb(new_image)
            )

        elif event_name == 'REMOVE':
            # Удаление записи
            old_image = record['dynamodb']['OldImage']
            handle_deleted_record(deserialize_dynamodb(old_image))


def handle_new_record(record: dict):
    """Обработка новой записи"""
    # Например, индексация в ElasticSearch
    if record.get('type') == 'product':
        index_product(record)
    # Или синхронизация с другим сервисом
    sync_to_external_system(record)


def handle_modified_record(old: dict, new: dict):
    """Обработка изменённой записи"""
    # Отслеживание изменений статуса заказа
    if old.get('status') != new.get('status'):
        notify_status_change(
            order_id=new['order_id'],
            old_status=old['status'],
            new_status=new['status']
        )
```

## Serverless Framework

### serverless.yml

```yaml
# serverless.yml - конфигурация Serverless Framework

service: my-api

frameworkVersion: '3'

provider:
  name: aws
  runtime: python3.11
  region: eu-west-1
  stage: ${opt:stage, 'dev'}

  # IAM роль для функций
  iam:
    role:
      statements:
        - Effect: Allow
          Action:
            - dynamodb:GetItem
            - dynamodb:PutItem
            - dynamodb:UpdateItem
            - dynamodb:DeleteItem
            - dynamodb:Scan
            - dynamodb:Query
          Resource:
            - !GetAtt UsersTable.Arn
            - !GetAtt OrdersTable.Arn

  # Переменные окружения
  environment:
    USERS_TABLE: ${self:service}-users-${self:provider.stage}
    ORDERS_TABLE: ${self:service}-orders-${self:provider.stage}

functions:
  # HTTP API функции
  createUser:
    handler: handlers/users.create
    events:
      - http:
          path: users
          method: post
          cors: true

  getUser:
    handler: handlers/users.get
    events:
      - http:
          path: users/{id}
          method: get
          cors: true

  listUsers:
    handler: handlers/users.list
    events:
      - http:
          path: users
          method: get
          cors: true

  # Обработчик очереди
  processOrder:
    handler: handlers/orders.process
    events:
      - sqs:
          arn: !GetAtt OrderQueue.Arn
          batchSize: 10

  # Запланированная задача
  dailyCleanup:
    handler: handlers/maintenance.cleanup
    events:
      - schedule: cron(0 2 * * ? *)

  # DynamoDB Stream
  syncToSearch:
    handler: handlers/sync.to_elasticsearch
    events:
      - stream:
          type: dynamodb
          arn: !GetAtt UsersTable.StreamArn
          batchSize: 100
          startingPosition: LATEST

# Ресурсы AWS
resources:
  Resources:
    UsersTable:
      Type: AWS::DynamoDB::Table
      Properties:
        TableName: ${self:provider.environment.USERS_TABLE}
        AttributeDefinitions:
          - AttributeName: id
            AttributeType: S
        KeySchema:
          - AttributeName: id
            KeyType: HASH
        BillingMode: PAY_PER_REQUEST
        StreamSpecification:
          StreamViewType: NEW_AND_OLD_IMAGES

    OrdersTable:
      Type: AWS::DynamoDB::Table
      Properties:
        TableName: ${self:provider.environment.ORDERS_TABLE}
        AttributeDefinitions:
          - AttributeName: id
            AttributeType: S
          - AttributeName: user_id
            AttributeType: S
        KeySchema:
          - AttributeName: id
            KeyType: HASH
        GlobalSecondaryIndexes:
          - IndexName: user-orders-index
            KeySchema:
              - AttributeName: user_id
                KeyType: HASH
            Projection:
              ProjectionType: ALL
        BillingMode: PAY_PER_REQUEST

    OrderQueue:
      Type: AWS::SQS::Queue
      Properties:
        QueueName: ${self:service}-orders-${self:provider.stage}
        VisibilityTimeout: 60
        RedrivePolicy:
          deadLetterTargetArn: !GetAtt OrderDLQ.Arn
          maxReceiveCount: 3

    OrderDLQ:
      Type: AWS::SQS::Queue
      Properties:
        QueueName: ${self:service}-orders-dlq-${self:provider.stage}

plugins:
  - serverless-python-requirements
  - serverless-offline

custom:
  pythonRequirements:
    dockerizePip: non-linux
```

### Структура проекта

```
my-serverless-api/
├── serverless.yml
├── requirements.txt
├── handlers/
│   ├── __init__.py
│   ├── users.py
│   ├── orders.py
│   ├── maintenance.py
│   └── sync.py
├── services/
│   ├── __init__.py
│   ├── user_service.py
│   └── order_service.py
├── models/
│   ├── __init__.py
│   ├── user.py
│   └── order.py
└── tests/
    ├── test_users.py
    └── test_orders.py
```

```python
# handlers/users.py
import json
import os
from services.user_service import UserService


user_service = UserService(os.environ['USERS_TABLE'])


def create(event, context):
    """POST /users"""
    try:
        body = json.loads(event['body'])
        user = user_service.create(body)

        return {
            'statusCode': 201,
            'headers': {'Content-Type': 'application/json'},
            'body': json.dumps(user)
        }
    except ValueError as e:
        return {
            'statusCode': 400,
            'body': json.dumps({'error': str(e)})
        }
    except Exception as e:
        return {
            'statusCode': 500,
            'body': json.dumps({'error': 'Internal server error'})
        }


def get(event, context):
    """GET /users/{id}"""
    user_id = event['pathParameters']['id']
    user = user_service.get(user_id)

    if not user:
        return {
            'statusCode': 404,
            'body': json.dumps({'error': 'User not found'})
        }

    return {
        'statusCode': 200,
        'headers': {'Content-Type': 'application/json'},
        'body': json.dumps(user)
    }


def list(event, context):
    """GET /users"""
    query_params = event.get('queryStringParameters') or {}
    limit = int(query_params.get('limit', 20))

    users = user_service.list(limit=limit)

    return {
        'statusCode': 200,
        'headers': {'Content-Type': 'application/json'},
        'body': json.dumps(users)
    }
```

```python
# services/user_service.py
import boto3
import uuid
from datetime import datetime
from typing import Optional, List


class UserService:
    def __init__(self, table_name: str):
        self.dynamodb = boto3.resource('dynamodb')
        self.table = self.dynamodb.Table(table_name)

    def create(self, data: dict) -> dict:
        """Создание пользователя"""
        if not data.get('email') or not data.get('name'):
            raise ValueError('Email and name are required')

        user = {
            'id': str(uuid.uuid4()),
            'email': data['email'],
            'name': data['name'],
            'created_at': datetime.now().isoformat(),
            'updated_at': datetime.now().isoformat()
        }

        self.table.put_item(Item=user)
        return user

    def get(self, user_id: str) -> Optional[dict]:
        """Получение пользователя"""
        response = self.table.get_item(Key={'id': user_id})
        return response.get('Item')

    def update(self, user_id: str, data: dict) -> Optional[dict]:
        """Обновление пользователя"""
        update_expr = 'SET updated_at = :updated_at'
        expr_values = {':updated_at': datetime.now().isoformat()}

        if 'name' in data:
            update_expr += ', #name = :name'
            expr_values[':name'] = data['name']

        if 'email' in data:
            update_expr += ', email = :email'
            expr_values[':email'] = data['email']

        response = self.table.update_item(
            Key={'id': user_id},
            UpdateExpression=update_expr,
            ExpressionAttributeValues=expr_values,
            ExpressionAttributeNames={'#name': 'name'},
            ReturnValues='ALL_NEW'
        )

        return response.get('Attributes')

    def delete(self, user_id: str) -> bool:
        """Удаление пользователя"""
        self.table.delete_item(Key={'id': user_id})
        return True

    def list(self, limit: int = 20) -> List[dict]:
        """Список пользователей"""
        response = self.table.scan(Limit=limit)
        return response.get('Items', [])
```

## Cold Start и оптимизация

### Проблема Cold Start

```python
# Оптимизация cold start

# 1. Инициализация вне обработчика
import boto3

# Это выполняется один раз при cold start
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('users')


def handler(event, context):
    # Это выполняется при каждом вызове
    # Но dynamodb и table уже инициализированы
    return table.get_item(Key={'id': event['id']})


# 2. Provisioned Concurrency (AWS Lambda)
# В serverless.yml:
# functions:
#   myFunction:
#     handler: handler.main
#     provisionedConcurrency: 5  # Всегда 5 тёплых инстансов


# 3. Lazy loading
class LazyLoader:
    """Отложенная загрузка тяжёлых модулей"""

    _heavy_module = None

    @classmethod
    def get_heavy_module(cls):
        if cls._heavy_module is None:
            import heavy_module  # Загружается только при первом использовании
            cls._heavy_module = heavy_module
        return cls._heavy_module


# 4. Уменьшение размера пакета
# requirements.txt - только необходимые зависимости
# Использование Lambda Layers для общих библиотек
```

### Мониторинг и логирование

```python
# Структурированное логирование
import json
import logging
import time
from functools import wraps

logger = logging.getLogger()
logger.setLevel(logging.INFO)


def log_execution(func):
    """Декоратор для логирования выполнения"""
    @wraps(func)
    def wrapper(event, context):
        start_time = time.time()

        # Логируем входящий запрос
        logger.info(json.dumps({
            'level': 'INFO',
            'message': 'Function invoked',
            'function_name': context.function_name,
            'request_id': context.aws_request_id,
            'event_type': event.get('httpMethod', 'unknown')
        }))

        try:
            result = func(event, context)

            # Логируем успешное выполнение
            duration = (time.time() - start_time) * 1000
            logger.info(json.dumps({
                'level': 'INFO',
                'message': 'Function completed',
                'request_id': context.aws_request_id,
                'duration_ms': duration,
                'status_code': result.get('statusCode', 200)
            }))

            return result

        except Exception as e:
            # Логируем ошибку
            duration = (time.time() - start_time) * 1000
            logger.error(json.dumps({
                'level': 'ERROR',
                'message': 'Function failed',
                'request_id': context.aws_request_id,
                'duration_ms': duration,
                'error': str(e),
                'error_type': type(e).__name__
            }))
            raise

    return wrapper


@log_execution
def handler(event, context):
    """Обработчик с логированием"""
    # Бизнес-логика
    pass
```

## Плюсы и минусы Serverless

### Плюсы

1. **Нет управления серверами** — провайдер занимается инфраструктурой
2. **Автоматическое масштабирование** — от 0 до миллионов запросов
3. **Оплата за использование** — платите только за выполнение кода
4. **Быстрое развёртывание** — деплой за секунды
5. **Высокая доступность** — встроенная отказоустойчивость
6. **Фокус на коде** — меньше DevOps работы

### Минусы

1. **Cold start** — задержка при первом вызове после простоя
2. **Vendor lock-in** — привязка к конкретному провайдеру
3. **Ограничения** — лимиты на время выполнения, память, размер пакета
4. **Сложность отладки** — труднее отлаживать распределённые функции
5. **Стоимость при высокой нагрузке** — может быть дороже VM
6. **Stateless** — нет состояния между вызовами

## Когда использовать Serverless

### Используйте Serverless когда:

- Непредсказуемая или переменная нагрузка
- Event-driven архитектура
- API с низким или средним трафиком
- Фоновые задачи и обработка событий
- Прототипы и MVP
- Микросервисы с чёткими границами

### Не используйте Serverless когда:

- Критичные требования к латентности (< 100ms)
- Долгие вычисления (> 15 минут)
- Постоянная высокая нагрузка (дешевле использовать VM)
- Сложное состояние между запросами
- Legacy приложения с монолитной архитектурой

## Заключение

Serverless архитектура позволяет сосредоточиться на бизнес-логике, оставив инфраструктурные заботы провайдеру. Она особенно хороша для event-driven систем и приложений с переменной нагрузкой. Однако важно понимать ограничения и выбирать этот подход осознанно.
