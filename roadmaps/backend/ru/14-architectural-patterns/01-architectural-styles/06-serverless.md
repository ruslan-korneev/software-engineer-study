# Бессерверная архитектура (Serverless Architecture)

## Определение

**Serverless Architecture** (бессерверная архитектура) — это модель выполнения облачных вычислений, при которой провайдер динамически управляет распределением серверных ресурсов. Разработчик пишет код в виде функций, а инфраструктура полностью абстрагирована. Оплата производится только за фактически использованные ресурсы (pay-per-execution).

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       SERVERLESS ARCHITECTURE                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   ┌─────────────┐      ┌─────────────┐      ┌─────────────┐             │
│   │   Events    │      │   Events    │      │   Events    │             │
│   │  (HTTP/S3/  │      │ (SQS/SNS/   │      │  (Schedule  │             │
│   │   API GW)   │      │   Kinesis)  │      │   /Cron)    │             │
│   └──────┬──────┘      └──────┬──────┘      └──────┬──────┘             │
│          │                    │                    │                     │
│          ▼                    ▼                    ▼                     │
│   ┌─────────────────────────────────────────────────────────┐           │
│   │                    FaaS Platform                         │           │
│   │         (AWS Lambda / Azure Functions / GCP)             │           │
│   │  ┌──────────┐  ┌──────────┐  ┌──────────┐               │           │
│   │  │Function 1│  │Function 2│  │Function 3│   ...         │           │
│   │  └──────────┘  └──────────┘  └──────────┘               │           │
│   └─────────────────────────────────────────────────────────┘           │
│          │                    │                    │                     │
│          ▼                    ▼                    ▼                     │
│   ┌─────────────────────────────────────────────────────────┐           │
│   │                  Managed Services (BaaS)                 │           │
│   │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐            │           │
│   │  │DynamoDB│ │   S3   │ │  SQS   │ │Cognito │  ...       │           │
│   │  └────────┘ └────────┘ └────────┘ └────────┘            │           │
│   └─────────────────────────────────────────────────────────┘           │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**Важно**: "Serverless" не означает отсутствие серверов — серверы есть, но ими управляет провайдер.

## Ключевые характеристики

### 1. Компоненты Serverless

| Компонент | Описание | Примеры |
|-----------|----------|---------|
| **FaaS** (Function as a Service) | Выполнение функций по событиям | AWS Lambda, Azure Functions, GCP Cloud Functions |
| **BaaS** (Backend as a Service) | Управляемые backend-сервисы | Firebase, AWS Amplify, Supabase |
| **Managed Services** | Полностью управляемые сервисы | DynamoDB, S3, API Gateway, Cognito |

### 2. Модель выполнения

```
┌─────────────────────────────────────────────────────────────────┐
│                     LAMBDA EXECUTION MODEL                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Event ──▶ Cold Start ──▶ Init ──▶ Handler ──▶ Response         │
│                │                      │                          │
│                │                      │                          │
│                ▼                      ▼                          │
│           Container              Function                        │
│           Creation               Execution                       │
│           (~100-500ms)           (~10-100ms)                    │
│                                                                  │
│  ─────────────────────────────────────────────────────────      │
│                                                                  │
│  Warm Start (контейнер переиспользуется):                       │
│                                                                  │
│  Event ──────────────────▶ Handler ──▶ Response                 │
│                              │                                   │
│                              ▼                                   │
│                          (~10-100ms)                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3. Event-Driven Nature

```
┌─────────────────────────────────────────────────────────────────┐
│                       EVENT SOURCES                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  HTTP Requests ─────┐                                           │
│  (API Gateway)      │                                           │
│                     │                                           │
│  File Uploads ──────┤                                           │
│  (S3 Events)        │      ┌──────────────┐                     │
│                     ├─────▶│   Lambda     │                     │
│  Queue Messages ────┤      │   Function   │                     │
│  (SQS/SNS)          │      └──────────────┘                     │
│                     │                                           │
│  Database Changes ──┤                                           │
│  (DynamoDB Streams) │                                           │
│                     │                                           │
│  Scheduled Events ──┘                                           │
│  (CloudWatch)                                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4. Характеристики FaaS

| Характеристика | Описание |
|----------------|----------|
| **Stateless** | Функции не хранят состояние между вызовами |
| **Short-lived** | Ограничение по времени выполнения (15 мин для Lambda) |
| **Event-triggered** | Запуск только по событию |
| **Auto-scaling** | Автоматическое масштабирование |
| **Pay-per-use** | Оплата за время выполнения |
| **Ephemeral** | Контейнеры создаются и уничтожаются динамически |

## Когда использовать

### Идеальные сценарии

| Сценарий | Почему Serverless подходит |
|----------|---------------------------|
| **Нерегулярная нагрузка** | Оплата только при использовании |
| **Event processing** | Обработка событий (файлы, сообщения) |
| **REST APIs** | Простые CRUD API с переменной нагрузкой |
| **Webhooks** | Обработка внешних событий |
| **Scheduled tasks** | Cron jobs без управления серверами |
| **Data pipelines** | ETL процессы |
| **MVP / Прототипы** | Быстрый запуск без ops |
| **Backends for mobile/SPA** | Простые backends |

### Примеры use cases

```
┌─────────────────────────────────────────────────────────────────┐
│                    SERVERLESS USE CASES                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Image Processing Pipeline                                   │
│     ┌─────┐     ┌──────────┐     ┌──────────┐     ┌─────┐      │
│     │ S3  │────▶│ Lambda   │────▶│ Lambda   │────▶│ S3  │      │
│     │Upload│    │ Resize   │     │ Thumbnail│     │Dest │      │
│     └─────┘     └──────────┘     └──────────┘     └─────┘      │
│                                                                  │
│  2. REST API                                                    │
│     ┌────────┐     ┌──────────┐     ┌──────────┐               │
│     │API     │────▶│ Lambda   │────▶│ DynamoDB │               │
│     │Gateway │     │ Handler  │     │          │               │
│     └────────┘     └──────────┘     └──────────┘               │
│                                                                  │
│  3. Real-time Data Processing                                   │
│     ┌────────┐     ┌──────────┐     ┌──────────┐               │
│     │Kinesis │────▶│ Lambda   │────▶│ S3/      │               │
│     │Stream  │     │ Process  │     │ Redshift │               │
│     └────────┘     └──────────┘     └──────────┘               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Не подходит для

- **Long-running processes** — ограничение времени выполнения
- **Stateful applications** — функции stateless
- **Высокая постоянная нагрузка** — может быть дороже традиционных серверов
- **Low-latency requirements** — cold start проблема
- **Complex orchestration** — сложные бизнес-процессы
- **Vendor lock-in concerns** — сильная привязка к провайдеру

## Преимущества и недостатки

### Преимущества

| Преимущество | Описание |
|--------------|----------|
| **No server management** | Нет администрирования серверов |
| **Auto-scaling** | Автоматическое масштабирование от 0 до тысяч |
| **Pay-per-use** | Оплата только за использованные ресурсы |
| **High availability** | Встроенная отказоустойчивость |
| **Faster time to market** | Фокус на бизнес-логике |
| **Built-in integrations** | Интеграции с другими облачными сервисами |

### Недостатки

| Недостаток | Описание |
|------------|----------|
| **Cold start latency** | Задержка при первом вызове |
| **Vendor lock-in** | Привязка к провайдеру |
| **Debugging complexity** | Сложная отладка распределённой системы |
| **Limited execution time** | Ограничение времени выполнения |
| **Stateless constraints** | Нельзя хранить состояние |
| **Cost unpredictability** | Сложно предсказать costs при scale |
| **Local development** | Сложность локального тестирования |

## Примеры реализации

### AWS Lambda с Python

```python
# handler.py
import json
import boto3
from datetime import datetime

# Инициализация клиентов (вне handler для reuse)
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('users')

def create_user(event, context):
    """
    Lambda handler для создания пользователя
    Триггер: API Gateway POST /users
    """
    try:
        # Парсинг тела запроса
        body = json.loads(event['body'])

        user = {
            'user_id': body['email'],  # partition key
            'name': body['name'],
            'email': body['email'],
            'created_at': datetime.utcnow().isoformat()
        }

        # Сохранение в DynamoDB
        table.put_item(Item=user)

        return {
            'statusCode': 201,
            'headers': {
                'Content-Type': 'application/json',
                'Access-Control-Allow-Origin': '*'
            },
            'body': json.dumps({
                'message': 'User created',
                'user': user
            })
        }

    except KeyError as e:
        return {
            'statusCode': 400,
            'body': json.dumps({'error': f'Missing field: {e}'})
        }
    except Exception as e:
        print(f"Error: {e}")
        return {
            'statusCode': 500,
            'body': json.dumps({'error': 'Internal server error'})
        }


def get_user(event, context):
    """
    Lambda handler для получения пользователя
    Триггер: API Gateway GET /users/{user_id}
    """
    user_id = event['pathParameters']['user_id']

    response = table.get_item(Key={'user_id': user_id})

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


def list_users(event, context):
    """
    Lambda handler для списка пользователей
    Триггер: API Gateway GET /users
    """
    response = table.scan()

    return {
        'statusCode': 200,
        'headers': {'Content-Type': 'application/json'},
        'body': json.dumps({
            'users': response['Items'],
            'count': response['Count']
        })
    }
```

### Serverless Framework Configuration

```yaml
# serverless.yml
service: user-service

provider:
  name: aws
  runtime: python3.11
  stage: ${opt:stage, 'dev'}
  region: eu-west-1

  # IAM permissions
  iam:
    role:
      statements:
        - Effect: Allow
          Action:
            - dynamodb:GetItem
            - dynamodb:PutItem
            - dynamodb:Scan
            - dynamodb:DeleteItem
          Resource: !GetAtt UsersTable.Arn

  # Environment variables
  environment:
    USERS_TABLE: !Ref UsersTable
    STAGE: ${self:provider.stage}

functions:
  createUser:
    handler: handler.create_user
    events:
      - http:
          path: users
          method: post
          cors: true

  getUser:
    handler: handler.get_user
    events:
      - http:
          path: users/{user_id}
          method: get
          cors: true

  listUsers:
    handler: handler.list_users
    events:
      - http:
          path: users
          method: get
          cors: true

  # Scheduled function
  cleanupInactiveUsers:
    handler: handler.cleanup_inactive_users
    events:
      - schedule: rate(1 day)

resources:
  Resources:
    UsersTable:
      Type: AWS::DynamoDB::Table
      Properties:
        TableName: users-${self:provider.stage}
        BillingMode: PAY_PER_REQUEST
        AttributeDefinitions:
          - AttributeName: user_id
            AttributeType: S
        KeySchema:
          - AttributeName: user_id
            KeyType: HASH

plugins:
  - serverless-python-requirements

custom:
  pythonRequirements:
    dockerizePip: true
```

### S3 Event Trigger

```python
# image_processor.py
import boto3
from PIL import Image
import io
import os

s3 = boto3.client('s3')

def process_image(event, context):
    """
    Lambda триггер: S3 upload event
    Создаёт thumbnail при загрузке изображения
    """
    for record in event['Records']:
        bucket = record['s3']['bucket']['name']
        key = record['s3']['object']['key']

        # Пропускаем уже обработанные (thumbnails)
        if key.startswith('thumbnails/'):
            continue

        print(f"Processing: s3://{bucket}/{key}")

        # Загрузка изображения
        response = s3.get_object(Bucket=bucket, Key=key)
        image_content = response['Body'].read()

        # Создание thumbnail
        img = Image.open(io.BytesIO(image_content))
        img.thumbnail((200, 200))

        # Сохранение в buffer
        buffer = io.BytesIO()
        img.save(buffer, 'JPEG', quality=85)
        buffer.seek(0)

        # Загрузка thumbnail в S3
        thumbnail_key = f"thumbnails/{os.path.basename(key)}"
        s3.put_object(
            Bucket=bucket,
            Key=thumbnail_key,
            Body=buffer,
            ContentType='image/jpeg'
        )

        print(f"Created thumbnail: s3://{bucket}/{thumbnail_key}")

    return {
        'statusCode': 200,
        'body': 'Images processed'
    }
```

### SQS Queue Consumer

```python
# queue_consumer.py
import json
import boto3
from botocore.exceptions import ClientError

dynamodb = boto3.resource('dynamodb')
orders_table = dynamodb.Table('orders')
ses = boto3.client('ses')

def process_order_queue(event, context):
    """
    Lambda триггер: SQS Queue
    Обрабатывает сообщения о новых заказах
    """
    failed_messages = []

    for record in event['Records']:
        try:
            # Парсинг сообщения
            body = json.loads(record['body'])
            order_id = body['order_id']
            user_email = body['user_email']

            print(f"Processing order: {order_id}")

            # Обновление статуса заказа
            orders_table.update_item(
                Key={'order_id': order_id},
                UpdateExpression='SET #status = :status',
                ExpressionAttributeNames={'#status': 'status'},
                ExpressionAttributeValues={':status': 'PROCESSING'}
            )

            # Отправка email
            send_order_confirmation(user_email, order_id)

            print(f"Order {order_id} processed successfully")

        except Exception as e:
            print(f"Error processing message: {e}")
            failed_messages.append({
                'itemIdentifier': record['messageId']
            })

    # Partial batch failure response
    if failed_messages:
        return {
            'batchItemFailures': failed_messages
        }

    return {'statusCode': 200}


def send_order_confirmation(email: str, order_id: str):
    """Отправка email через SES"""
    ses.send_email(
        Source='orders@example.com',
        Destination={'ToAddresses': [email]},
        Message={
            'Subject': {'Data': f'Order {order_id} Confirmed'},
            'Body': {
                'Text': {'Data': f'Your order {order_id} has been confirmed.'}
            }
        }
    )
```

### Step Functions (Orchestration)

```json
{
  "Comment": "Order Processing Workflow",
  "StartAt": "ValidateOrder",
  "States": {
    "ValidateOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:xxx:function:validate-order",
      "Next": "ProcessPayment",
      "Catch": [{
        "ErrorEquals": ["ValidationError"],
        "Next": "OrderFailed"
      }]
    },
    "ProcessPayment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:xxx:function:process-payment",
      "Next": "ReserveInventory",
      "Catch": [{
        "ErrorEquals": ["PaymentError"],
        "Next": "RefundPayment"
      }]
    },
    "ReserveInventory": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:xxx:function:reserve-inventory",
      "Next": "SendConfirmation",
      "Catch": [{
        "ErrorEquals": ["InventoryError"],
        "Next": "RefundPayment"
      }]
    },
    "SendConfirmation": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:xxx:function:send-confirmation",
      "End": true
    },
    "RefundPayment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:xxx:function:refund-payment",
      "Next": "OrderFailed"
    },
    "OrderFailed": {
      "Type": "Fail",
      "Error": "OrderProcessingFailed",
      "Cause": "Order could not be processed"
    }
  }
}
```

### Local Development с LocalStack

```yaml
# docker-compose.yml
version: '3.8'

services:
  localstack:
    image: localstack/localstack:latest
    ports:
      - "4566:4566"
    environment:
      - SERVICES=lambda,dynamodb,s3,sqs,apigateway
      - DEBUG=1
      - LAMBDA_EXECUTOR=docker
      - DOCKER_HOST=unix:///var/run/docker.sock
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
      - "./init-aws.sh:/etc/localstack/init/ready.d/init-aws.sh"
```

```bash
# init-aws.sh
#!/bin/bash

# Create DynamoDB table
awslocal dynamodb create-table \
    --table-name users \
    --attribute-definitions AttributeName=user_id,AttributeType=S \
    --key-schema AttributeName=user_id,KeyType=HASH \
    --billing-mode PAY_PER_REQUEST

# Create S3 bucket
awslocal s3 mb s3://images-bucket

# Create SQS queue
awslocal sqs create-queue --queue-name orders-queue
```

## Best practices и антипаттерны

### Best Practices

#### 1. Минимизация Cold Start
```python
# Инициализация ВНЕ handler (reuse)
import boto3

# Эти объекты переиспользуются между вызовами
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('users')

def handler(event, context):
    # Только бизнес-логика
    return table.get_item(Key={'id': event['id']})
```

#### 2. Идемпотентность
```python
import hashlib

def process_payment(event, context):
    # Генерация idempotency key
    idempotency_key = hashlib.md5(
        json.dumps(event, sort_keys=True).encode()
    ).hexdigest()

    # Проверка, обработано ли уже
    existing = table.get_item(Key={'idempotency_key': idempotency_key})
    if 'Item' in existing:
        return existing['Item']['result']

    # Обработка платежа
    result = do_payment(event)

    # Сохранение результата
    table.put_item(Item={
        'idempotency_key': idempotency_key,
        'result': result,
        'ttl': int(time.time()) + 86400  # 24 hours
    })

    return result
```

#### 3. Structured Logging
```python
import json
import logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def handler(event, context):
    # Structured logging для CloudWatch Insights
    logger.info(json.dumps({
        'event': 'order_created',
        'order_id': event['order_id'],
        'user_id': event['user_id'],
        'amount': event['amount'],
        'request_id': context.aws_request_id
    }))
```

#### 4. Error Handling
```python
class ValidationError(Exception):
    pass

class PaymentError(Exception):
    pass

def handler(event, context):
    try:
        result = process_order(event)
        return {
            'statusCode': 200,
            'body': json.dumps(result)
        }
    except ValidationError as e:
        return {'statusCode': 400, 'body': json.dumps({'error': str(e)})}
    except PaymentError as e:
        # Retry-able error
        raise  # Lambda retry механизм
    except Exception as e:
        logger.error(f"Unexpected error: {e}")
        return {'statusCode': 500, 'body': json.dumps({'error': 'Internal error'})}
```

### Антипаттерны

#### 1. Lambda Monolith
```python
# ПЛОХО: огромная Lambda со всей логикой
def handler(event, context):
    path = event['path']
    method = event['httpMethod']

    if path == '/users' and method == 'POST':
        return create_user(event)
    elif path == '/users' and method == 'GET':
        return list_users(event)
    elif path.startswith('/orders'):
        return handle_orders(event)
    # ... 1000 строк кода

# ХОРОШО: отдельные функции
# create_user.py
# list_users.py
# create_order.py
```

#### 2. Synchronous Chains
```python
# ПЛОХО: цепочка синхронных вызовов
def create_order(event, context):
    # Каждый invoke добавляет latency
    lambda_client.invoke(FunctionName='validate-user')
    lambda_client.invoke(FunctionName='check-inventory')
    lambda_client.invoke(FunctionName='process-payment')
    lambda_client.invoke(FunctionName='send-notification')

# ХОРОШО: асинхронная обработка через очереди
def create_order(event, context):
    order = save_order(event)
    sqs.send_message(QueueUrl=QUEUE_URL, MessageBody=json.dumps(order))
    return order
```

#### 3. Хранение состояния в Lambda
```python
# ПЛОХО: состояние в глобальных переменных
cache = {}

def handler(event, context):
    key = event['key']
    if key in cache:
        return cache[key]  # Может не работать при новом контейнере!

    result = expensive_operation()
    cache[key] = result  # Потеряется при cold start
    return result

# ХОРОШО: внешнее хранилище состояния
def handler(event, context):
    key = event['key']
    cached = redis.get(key)  # Или DynamoDB
    if cached:
        return json.loads(cached)

    result = expensive_operation()
    redis.setex(key, 3600, json.dumps(result))
    return result
```

## Связанные паттерны

| Паттерн | Описание |
|---------|----------|
| **Event-Driven Architecture** | Serverless основан на событиях |
| **CQRS** | Разделение read/write в serverless |
| **Saga Pattern** | Распределённые транзакции через Step Functions |
| **Fan-Out/Fan-In** | Параллельная обработка через SNS/SQS |
| **API Gateway Pattern** | Единая точка входа для Lambda |
| **Backend for Frontend** | Отдельные Lambda для разных клиентов |

## Сравнение провайдеров

| Аспект | AWS Lambda | Azure Functions | GCP Cloud Functions |
|--------|------------|-----------------|---------------------|
| **Max execution time** | 15 min | 10 min (Premium: unlimited) | 9 min |
| **Languages** | Python, Node, Java, Go, .NET, Ruby | .NET, Node, Python, Java, PowerShell | Node, Python, Go, Java, .NET |
| **Triggers** | 200+ AWS services | Azure services, HTTP | GCP services, HTTP |
| **Cold start** | 100-500ms | 500ms-2s | 100-500ms |
| **Pricing model** | Per 1ms, GB-seconds | Per execution, GB-seconds | Per 100ms, GB-seconds |

## Ресурсы для изучения

### Книги
- **"Serverless Architectures on AWS"** — Peter Sbarski
- **"AWS Lambda in Action"** — Danilo Poccia
- **"Programming AWS Lambda"** — John Chapin

### Документация
- [AWS Lambda Developer Guide](https://docs.aws.amazon.com/lambda/latest/dg/)
- [Serverless Framework Docs](https://www.serverless.com/framework/docs/)
- [AWS SAM Documentation](https://docs.aws.amazon.com/serverless-application-model/)

### Курсы
- "AWS Lambda & Serverless Architecture" — Udemy
- "Serverless Framework Bootcamp" — A Cloud Guru

### Инструменты
- **Serverless Framework** — Multi-cloud serverless deployment
- **AWS SAM** — AWS Serverless Application Model
- **LocalStack** — Local AWS cloud stack
- **SST (Serverless Stack)** — Fullstack serverless framework

---

## Резюме

Serverless — это paradigm shift в разработке backend:

- **Ключевые концепции**: FaaS, BaaS, event-driven, pay-per-use
- **Преимущества**: no ops, auto-scaling, cost efficiency
- **Недостатки**: cold start, vendor lock-in, debugging complexity
- **Подходит для**: event processing, APIs, scheduled tasks, MVPs

**Правило**: Serverless отлично подходит для event-driven workloads с переменной нагрузкой, но не для всех случаев. Оценивайте trade-offs для каждого проекта.
