# Virtual Hosts

## Что такое Virtual Host?

**Virtual Host (vhost)** — это логическое разделение ресурсов внутри одного RabbitMQ сервера. Каждый vhost имеет собственный набор очередей, exchange'ей, bindings и прав доступа.

## Назначение Virtual Hosts

```
┌─────────────────────────────────────────────────────────────────┐
│                    RabbitMQ Server                               │
├─────────────────────────────────────────────────────────────────┤
│  ┌───────────────────┐  ┌───────────────────┐  ┌──────────────┐│
│  │  vhost: /         │  │  vhost: /prod     │  │ vhost: /dev  ││
│  ├───────────────────┤  ├───────────────────┤  ├──────────────┤│
│  │ - exchanges       │  │ - exchanges       │  │ - exchanges  ││
│  │ - queues          │  │ - queues          │  │ - queues     ││
│  │ - bindings        │  │ - bindings        │  │ - bindings   ││
│  │ - permissions     │  │ - permissions     │  │ - permissions││
│  └───────────────────┘  └───────────────────┘  └──────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

Virtual Hosts позволяют:
- **Изоляцию** — разделение ресурсов между приложениями
- **Multi-tenancy** — обслуживание нескольких клиентов на одном сервере
- **Безопасность** — разные права доступа для разных пользователей
- **Организацию** — логическое группирование по средам (dev/prod)

## Создание Virtual Hosts

### Через CLI (rabbitmqctl)

```bash
# Список всех vhosts
rabbitmqctl list_vhosts

# Создание vhost
rabbitmqctl add_vhost /production
rabbitmqctl add_vhost /staging
rabbitmqctl add_vhost /development

# Создание vhost с описанием и тегами
rabbitmqctl add_vhost /production \
  --description "Production environment" \
  --tags "production,critical"

# Удаление vhost (ОСТОРОЖНО: удаляет все данные!)
rabbitmqctl delete_vhost /old_vhost
```

### Через Management HTTP API

```python
import requests
from requests.auth import HTTPBasicAuth

BASE_URL = "http://localhost:15672/api"
AUTH = HTTPBasicAuth('guest', 'guest')


def create_vhost(name: str, description: str = None):
    """Создание virtual host"""
    url = f"{BASE_URL}/vhosts/{name}"
    data = {}
    if description:
        data['description'] = description

    response = requests.put(url, json=data, auth=AUTH)
    return response.status_code == 201


def delete_vhost(name: str):
    """Удаление virtual host"""
    url = f"{BASE_URL}/vhosts/{name}"
    response = requests.delete(url, auth=AUTH)
    return response.status_code == 204


def list_vhosts():
    """Список всех virtual hosts"""
    url = f"{BASE_URL}/vhosts"
    response = requests.get(url, auth=AUTH)
    return response.json()


# Использование
create_vhost('production', 'Production environment')
create_vhost('staging', 'Staging environment')

for vhost in list_vhosts():
    print(f"VHost: {vhost['name']}")
```

### Через rabbitmqadmin

```bash
# Создание vhost
rabbitmqadmin declare vhost name=/production

# Список vhosts
rabbitmqadmin list vhosts

# Удаление vhost
rabbitmqadmin delete vhost name=/old_vhost
```

## Подключение к Virtual Host

### С помощью pika

```python
import pika

# Подключение к default vhost (/)
connection = pika.BlockingConnection(
    pika.ConnectionParameters(
        host='localhost',
        virtual_host='/'  # Default
    )
)

# Подключение к конкретному vhost
connection = pika.BlockingConnection(
    pika.ConnectionParameters(
        host='localhost',
        virtual_host='/production',
        credentials=pika.PlainCredentials('prod_user', 'password')
    )
)

# Через URL (/ кодируется как %2F)
connection = pika.BlockingConnection(
    pika.URLParameters('amqp://user:pass@localhost:5672/%2Fproduction')
)

channel = connection.channel()
print(f"Connected to vhost")
connection.close()
```

### URL-кодирование

```python
from urllib.parse import quote

# Правильное кодирование имени vhost в URL
vhost = '/production'
encoded_vhost = quote(vhost, safe='')  # %2Fproduction

url = f'amqp://user:pass@localhost:5672/{encoded_vhost}'
```

## Права доступа (Permissions)

### Структура прав

Права в RabbitMQ состоят из трёх компонентов:
- **configure** — создание/удаление ресурсов (очередей, exchange'ей)
- **write** — публикация сообщений
- **read** — потребление сообщений, получение из очередей

Права задаются как регулярные выражения.

### Установка прав через CLI

```bash
# Полные права для пользователя на vhost
rabbitmqctl set_permissions -p /production user_name ".*" ".*" ".*"

# Только чтение (потребление)
rabbitmqctl set_permissions -p /production reader "" "" ".*"

# Только запись (публикация)
rabbitmqctl set_permissions -p /production writer "" ".*" ""

# Права на определённые очереди (по паттерну)
rabbitmqctl set_permissions -p /production order_service \
  "^order\." "^order\." "^order\."

# Просмотр прав
rabbitmqctl list_permissions -p /production

# Просмотр прав пользователя
rabbitmqctl list_user_permissions user_name

# Удаление прав
rabbitmqctl clear_permissions -p /production user_name
```

### Установка прав через API

```python
import requests
from requests.auth import HTTPBasicAuth

BASE_URL = "http://localhost:15672/api"
AUTH = HTTPBasicAuth('guest', 'guest')


def set_permissions(
    vhost: str,
    user: str,
    configure: str = ".*",
    write: str = ".*",
    read: str = ".*"
):
    """Установка прав доступа"""
    url = f"{BASE_URL}/permissions/{vhost}/{user}"
    data = {
        'configure': configure,
        'write': write,
        'read': read
    }
    response = requests.put(url, json=data, auth=AUTH)
    return response.status_code == 201


def get_permissions(vhost: str):
    """Получение всех прав для vhost"""
    url = f"{BASE_URL}/vhosts/{vhost}/permissions"
    response = requests.get(url, auth=AUTH)
    return response.json()


# Примеры
# Полный доступ
set_permissions('/production', 'admin')

# Только публикация в очереди events.*
set_permissions('/production', 'event_publisher',
    configure='',
    write='^events\\..*',
    read=''
)

# Только потребление из очередей tasks.*
set_permissions('/production', 'task_worker',
    configure='',
    write='',
    read='^tasks\\..*'
)
```

## Multi-tenancy

### Архитектура для множества клиентов

```
┌─────────────────────────────────────────────────────────────────┐
│                    RabbitMQ Server                               │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────────┐│
│  │ vhost: /client_a│ │ vhost: /client_b│ │ vhost: /client_c    ││
│  │ user: client_a  │ │ user: client_b  │ │ user: client_c      ││
│  │ - orders queue  │ │ - orders queue  │ │ - orders queue      ││
│  │ - events exch   │ │ - events exch   │ │ - events exch       ││
│  └─────────────────┘ └─────────────────┘ └─────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

### Реализация multi-tenancy

```python
import pika
import requests
from requests.auth import HTTPBasicAuth
from typing import Dict, Any


class TenantManager:
    """Менеджер для управления tenant'ами"""

    def __init__(
        self,
        host: str = 'localhost',
        admin_user: str = 'guest',
        admin_pass: str = 'guest'
    ):
        self.host = host
        self.api_url = f"http://{host}:15672/api"
        self.auth = HTTPBasicAuth(admin_user, admin_pass)

    def create_tenant(
        self,
        tenant_id: str,
        password: str,
        queues: list = None,
        exchanges: list = None
    ) -> Dict[str, Any]:
        """Создание нового tenant'а с изолированным vhost"""

        vhost = f"/tenant_{tenant_id}"
        user = f"tenant_{tenant_id}"

        # 1. Создание vhost
        requests.put(
            f"{self.api_url}/vhosts/{vhost.replace('/', '%2F')}",
            json={'description': f'Tenant {tenant_id}'},
            auth=self.auth
        )

        # 2. Создание пользователя
        requests.put(
            f"{self.api_url}/users/{user}",
            json={
                'password': password,
                'tags': 'tenant'
            },
            auth=self.auth
        )

        # 3. Установка прав
        requests.put(
            f"{self.api_url}/permissions/{vhost.replace('/', '%2F')}/{user}",
            json={
                'configure': '.*',
                'write': '.*',
                'read': '.*'
            },
            auth=self.auth
        )

        # 4. Создание базовых ресурсов
        self._setup_tenant_resources(vhost, user, password, queues, exchanges)

        return {
            'vhost': vhost,
            'user': user,
            'amqp_url': f'amqp://{user}:{password}@{self.host}:5672/{vhost.replace("/", "%2F")}'
        }

    def _setup_tenant_resources(
        self,
        vhost: str,
        user: str,
        password: str,
        queues: list = None,
        exchanges: list = None
    ):
        """Создание ресурсов для tenant'а"""

        connection = pika.BlockingConnection(
            pika.ConnectionParameters(
                host=self.host,
                virtual_host=vhost,
                credentials=pika.PlainCredentials(user, password)
            )
        )
        channel = connection.channel()

        # Создание очередей
        default_queues = ['events', 'tasks', 'notifications']
        for queue in (queues or default_queues):
            channel.queue_declare(queue=queue, durable=True)

        # Создание exchange'ей
        default_exchanges = [
            ('events', 'topic'),
            ('notifications', 'fanout')
        ]
        for exchange, ex_type in (exchanges or default_exchanges):
            channel.exchange_declare(
                exchange=exchange,
                exchange_type=ex_type,
                durable=True
            )

        connection.close()

    def delete_tenant(self, tenant_id: str):
        """Удаление tenant'а"""
        vhost = f"/tenant_{tenant_id}"
        user = f"tenant_{tenant_id}"

        # Удаление vhost (удалит все ресурсы)
        requests.delete(
            f"{self.api_url}/vhosts/{vhost.replace('/', '%2F')}",
            auth=self.auth
        )

        # Удаление пользователя
        requests.delete(
            f"{self.api_url}/users/{user}",
            auth=self.auth
        )

    def list_tenants(self) -> list:
        """Список всех tenant'ов"""
        response = requests.get(f"{self.api_url}/vhosts", auth=self.auth)
        vhosts = response.json()

        return [
            v['name'] for v in vhosts
            if v['name'].startswith('/tenant_')
        ]


# Использование
manager = TenantManager()

# Создание tenant'а
tenant_info = manager.create_tenant(
    tenant_id='company_abc',
    password='secure_password_123',
    queues=['orders', 'payments', 'notifications']
)

print(f"Tenant created:")
print(f"  VHost: {tenant_info['vhost']}")
print(f"  AMQP URL: {tenant_info['amqp_url']}")

# Подключение от имени tenant'а
connection = pika.BlockingConnection(
    pika.URLParameters(tenant_info['amqp_url'])
)
channel = connection.channel()

# Публикация в изолированном пространстве
channel.basic_publish(
    exchange='',
    routing_key='orders',
    body='{"order_id": 123}'
)

connection.close()
```

## Разделение окружений

### Development / Staging / Production

```python
import pika
from enum import Enum


class Environment(Enum):
    DEVELOPMENT = '/development'
    STAGING = '/staging'
    PRODUCTION = '/production'


class EnvironmentConfig:
    """Конфигурация для разных окружений"""

    CONFIGS = {
        Environment.DEVELOPMENT: {
            'host': 'localhost',
            'user': 'dev_user',
            'password': 'dev_pass',
            'queue_prefix': 'dev_'
        },
        Environment.STAGING: {
            'host': 'staging.rabbitmq.local',
            'user': 'staging_user',
            'password': 'staging_pass',
            'queue_prefix': 'stg_'
        },
        Environment.PRODUCTION: {
            'host': 'prod.rabbitmq.local',
            'user': 'prod_user',
            'password': 'prod_pass',
            'queue_prefix': 'prod_'
        }
    }

    @classmethod
    def get_connection(cls, env: Environment):
        """Получение соединения для окружения"""
        config = cls.CONFIGS[env]

        return pika.BlockingConnection(
            pika.ConnectionParameters(
                host=config['host'],
                virtual_host=env.value,
                credentials=pika.PlainCredentials(
                    config['user'],
                    config['password']
                )
            )
        )


# Использование
import os

# Определение окружения
env_name = os.getenv('ENVIRONMENT', 'DEVELOPMENT')
env = Environment[env_name]

# Подключение к нужному vhost
connection = EnvironmentConfig.get_connection(env)
channel = connection.channel()

print(f"Connected to {env.value}")
```

## Мониторинг Virtual Hosts

### Статистика через API

```python
import requests
from requests.auth import HTTPBasicAuth


def get_vhost_stats(vhost: str):
    """Получение статистики vhost"""

    encoded_vhost = vhost.replace('/', '%2F')
    url = f"http://localhost:15672/api/vhosts/{encoded_vhost}"
    auth = HTTPBasicAuth('guest', 'guest')

    response = requests.get(url, auth=auth)
    data = response.json()

    stats = {
        'name': data['name'],
        'messages': data.get('messages', 0),
        'messages_ready': data.get('messages_ready', 0),
        'messages_unacknowledged': data.get('messages_unacknowledged', 0),
        'recv_oct': data.get('recv_oct', 0),  # Bytes received
        'send_oct': data.get('send_oct', 0),  # Bytes sent
    }

    return stats


def get_vhost_queues(vhost: str):
    """Получение списка очередей в vhost"""

    encoded_vhost = vhost.replace('/', '%2F')
    url = f"http://localhost:15672/api/queues/{encoded_vhost}"
    auth = HTTPBasicAuth('guest', 'guest')

    response = requests.get(url, auth=auth)
    return response.json()


# Мониторинг всех vhosts
def monitor_all_vhosts():
    url = "http://localhost:15672/api/vhosts"
    auth = HTTPBasicAuth('guest', 'guest')

    response = requests.get(url, auth=auth)
    vhosts = response.json()

    for vhost in vhosts:
        print(f"\nVHost: {vhost['name']}")
        print(f"  Messages: {vhost.get('messages', 0)}")
        print(f"  Ready: {vhost.get('messages_ready', 0)}")
        print(f"  Unacked: {vhost.get('messages_unacknowledged', 0)}")


monitor_all_vhosts()
```

## Лимиты и квоты

### Установка лимитов на vhost

```bash
# Максимальное количество очередей
rabbitmqctl set_vhost_limits -p /production '{"max-queues": 100}'

# Максимальное количество соединений
rabbitmqctl set_vhost_limits -p /production '{"max-connections": 50}'

# Просмотр лимитов
rabbitmqctl list_vhost_limits -p /production

# Сброс лимитов
rabbitmqctl clear_vhost_limits -p /production
```

### Через API

```python
import requests
from requests.auth import HTTPBasicAuth

BASE_URL = "http://localhost:15672/api"
AUTH = HTTPBasicAuth('guest', 'guest')


def set_vhost_limits(vhost: str, max_queues: int = None, max_connections: int = None):
    """Установка лимитов для vhost"""

    encoded_vhost = vhost.replace('/', '%2F')
    url = f"{BASE_URL}/vhost-limits/{encoded_vhost}/max-queues"

    limits = {}
    if max_queues:
        limits['value'] = max_queues
        requests.put(
            f"{BASE_URL}/vhost-limits/{encoded_vhost}/max-queues",
            json=limits,
            auth=AUTH
        )

    if max_connections:
        limits['value'] = max_connections
        requests.put(
            f"{BASE_URL}/vhost-limits/{encoded_vhost}/max-connections",
            json=limits,
            auth=AUTH
        )


# Установка лимитов
set_vhost_limits('/production', max_queues=100, max_connections=50)
```

## Best Practices

### 1. Используйте отдельные vhost для окружений

```bash
rabbitmqctl add_vhost /development
rabbitmqctl add_vhost /staging
rabbitmqctl add_vhost /production
```

### 2. Создавайте отдельных пользователей для каждого vhost

```bash
rabbitmqctl add_user prod_user secure_password
rabbitmqctl set_permissions -p /production prod_user ".*" ".*" ".*"
```

### 3. Применяйте принцип минимальных прав

```bash
# Только чтение
rabbitmqctl set_permissions -p /production reader "" "" "^tasks\\..*"

# Только запись
rabbitmqctl set_permissions -p /production writer "" "^events\\..*" ""
```

### 4. Устанавливайте лимиты для защиты от перегрузки

```bash
rabbitmqctl set_vhost_limits -p /production '{"max-queues": 1000}'
```

### 5. Мониторьте использование ресурсов

```python
# Регулярная проверка статистики
stats = get_vhost_stats('/production')
if stats['messages_unacknowledged'] > 10000:
    alert("Too many unacknowledged messages!")
```

### 6. Документируйте структуру vhost

```yaml
# vhosts.yaml
vhosts:
  - name: /production
    description: Production environment
    users:
      - name: prod_app
        permissions:
          configure: ".*"
          write: ".*"
          read: ".*"
    limits:
      max-queues: 1000
      max-connections: 500
```

## Типичные ошибки

1. **Использование одного vhost для всех окружений** — нет изоляции
2. **Слишком широкие права** — риски безопасности
3. **Отсутствие лимитов** — возможность DoS
4. **Неправильное кодирование vhost в URL** — ошибки подключения
5. **Удаление vhost без бэкапа** — потеря всех данных

## Заключение

Virtual Hosts — мощный инструмент для:
- **Изоляции** приложений и окружений
- **Multi-tenancy** — обслуживания множества клиентов
- **Безопасности** — разграничения прав доступа
- **Организации** — логической группировки ресурсов

Правильное использование vhosts обеспечивает безопасную и масштабируемую архитектуру вашей системы обмена сообщениями.
