# Аутентификация в RabbitMQ

[prev: 05-quorum-queues-ha](../09-clustering/05-quorum-queues-ha.md) | [next: 02-authorization](./02-authorization.md)

---

## Введение

Аутентификация — это процесс проверки подлинности клиента, подключающегося к RabbitMQ. Брокер должен убедиться, что клиент является тем, за кого себя выдаёт, прежде чем разрешить доступ к ресурсам.

## Механизмы аутентификации SASL

RabbitMQ использует SASL (Simple Authentication and Security Layer) для аутентификации. Поддерживаются следующие механизмы:

### PLAIN

Самый распространённый механизм. Учётные данные передаются в открытом виде (username и password).

```erlang
% Конфигурация в rabbitmq.conf
auth_mechanisms.1 = PLAIN
```

**Важно**: При использовании PLAIN обязательно настройте TLS для шифрования соединения!

### AMQPLAIN

Устаревший механизм, специфичный для AMQP 0-9-1. Сохранён для обратной совместимости.

```erlang
% Включение AMQPLAIN
auth_mechanisms.1 = PLAIN
auth_mechanisms.2 = AMQPLAIN
```

### EXTERNAL

Делегирует аутентификацию внешнему механизму, например, клиентским сертификатам TLS.

```erlang
% Конфигурация для использования сертификатов
auth_mechanisms.1 = EXTERNAL
auth_mechanisms.2 = PLAIN

% Извлечение имени пользователя из сертификата
ssl_cert_login_from = common_name
```

Пример подключения с EXTERNAL:

```python
import ssl
import pika

context = ssl.SSLContext(ssl.PROTOCOL_TLS_CLIENT)
context.load_cert_chain(
    certfile='/path/to/client_cert.pem',
    keyfile='/path/to/client_key.pem'
)
context.load_verify_locations('/path/to/ca_bundle.pem')

credentials = pika.credentials.ExternalCredentials()
parameters = pika.ConnectionParameters(
    host='rabbitmq.example.com',
    port=5671,
    credentials=credentials,
    ssl_options=pika.SSLOptions(context)
)

connection = pika.BlockingConnection(parameters)
```

## Backend Plugins для аутентификации

RabbitMQ поддерживает подключаемые backend-плагины для различных источников аутентификации.

### Internal Backend (по умолчанию)

Хранит учётные данные во внутренней базе данных Mnesia.

```erlang
% rabbitmq.conf
auth_backends.1 = internal
```

Управление пользователями:

```bash
# Создание пользователя
rabbitmqctl add_user myuser mypassword

# Изменение пароля
rabbitmqctl change_password myuser newpassword

# Удаление пользователя
rabbitmqctl delete_user myuser

# Список пользователей
rabbitmqctl list_users
```

### LDAP Backend

Интеграция с LDAP/Active Directory.

```erlang
% Включение плагина
rabbitmq-plugins enable rabbitmq_auth_backend_ldap

% Конфигурация в rabbitmq.conf
auth_backends.1 = ldap

auth_ldap.servers.1 = ldap.example.com
auth_ldap.port = 389
auth_ldap.user_dn_pattern = cn=${username},ou=users,dc=example,dc=com
auth_ldap.use_ssl = false
auth_ldap.use_starttls = true

% Группы для определения тегов
auth_ldap.tag_queries.administrator.dn_group = cn=admin,ou=groups,dc=example,dc=com
auth_ldap.tag_queries.management.dn_group = cn=management,ou=groups,dc=example,dc=com
```

### HTTP Backend

Делегирование аутентификации внешнему HTTP-сервису.

```erlang
% Включение плагина
rabbitmq-plugins enable rabbitmq_auth_backend_http

% Конфигурация
auth_backends.1 = http

auth_http.http_method = post
auth_http.user_path = http://auth-service:8080/auth/user
auth_http.vhost_path = http://auth-service:8080/auth/vhost
auth_http.resource_path = http://auth-service:8080/auth/resource
auth_http.topic_path = http://auth-service:8080/auth/topic
```

Пример реализации auth-сервиса на Python:

```python
from flask import Flask, request

app = Flask(__name__)

@app.route('/auth/user', methods=['POST'])
def authenticate_user():
    username = request.form.get('username')
    password = request.form.get('password')

    # Проверка учётных данных
    if validate_credentials(username, password):
        return 'allow administrator'  # или 'allow' без тегов
    return 'deny'

@app.route('/auth/vhost', methods=['POST'])
def check_vhost():
    username = request.form.get('username')
    vhost = request.form.get('vhost')

    if user_can_access_vhost(username, vhost):
        return 'allow'
    return 'deny'

@app.route('/auth/resource', methods=['POST'])
def check_resource():
    username = request.form.get('username')
    vhost = request.form.get('vhost')
    resource = request.form.get('resource')  # exchange или queue
    name = request.form.get('name')
    permission = request.form.get('permission')  # configure, write, read

    if user_has_permission(username, vhost, resource, name, permission):
        return 'allow'
    return 'deny'
```

### OAuth 2.0 Backend

Интеграция с OAuth 2.0 провайдерами (Keycloak, Auth0 и др.).

```erlang
% Включение плагина
rabbitmq-plugins enable rabbitmq_auth_backend_oauth2

% Конфигурация
auth_backends.1 = oauth2

auth_oauth2.resource_server_id = rabbitmq
auth_oauth2.issuer = https://keycloak.example.com/realms/myrealm
auth_oauth2.https.cacertfile = /path/to/ca_bundle.pem
```

## Комбинирование Backend'ов

Можно настроить цепочку проверки через несколько backend'ов:

```erlang
% Сначала проверяем в LDAP, потом во внутренней базе
auth_backends.1 = ldap
auth_backends.2 = internal

% Или использовать LDAP для аутентификации, internal для авторизации
auth_backends.1.authn = ldap
auth_backends.1.authz = internal
```

## Хэширование паролей

RabbitMQ поддерживает несколько алгоритмов хэширования:

```erlang
% Рекомендуемый алгоритм (по умолчанию)
password_hashing_module = rabbit_password_hashing_sha256

% Более старый алгоритм
password_hashing_module = rabbit_password_hashing_sha512

% Устаревший (не рекомендуется)
password_hashing_module = rabbit_password_hashing_md5
```

## Best Practices

### 1. Отключите гостевого пользователя

```bash
# Удалите или измените пароль guest
rabbitmqctl delete_user guest

# Или ограничьте подключение только с localhost (по умолчанию)
# В rabbitmq.conf
loopback_users.guest = true
```

### 2. Используйте сложные пароли

```bash
# Генерация надёжного пароля
openssl rand -base64 32
```

### 3. Регулярно ротируйте учётные данные

```bash
# Скрипт ротации паролей
#!/bin/bash
NEW_PASSWORD=$(openssl rand -base64 32)
rabbitmqctl change_password myuser "$NEW_PASSWORD"
# Обновите конфигурацию клиентов
```

### 4. Используйте отдельных пользователей для приложений

```bash
# Для каждого сервиса — свой пользователь
rabbitmqctl add_user order-service $(openssl rand -base64 32)
rabbitmqctl add_user payment-service $(openssl rand -base64 32)
rabbitmqctl add_user notification-service $(openssl rand -base64 32)
```

## Распространённые уязвимости

### 1. Использование стандартных учётных данных

**Проблема**: guest/guest остаётся активным.
**Решение**: Удалить или заблокировать гостевого пользователя.

### 2. Передача паролей в открытом виде

**Проблема**: PLAIN без TLS.
**Решение**: Всегда использовать TLS или механизм EXTERNAL.

### 3. Хранение паролей в коде

**Проблема**: Учётные данные в репозитории.
**Решение**: Использовать переменные окружения или секретные хранилища (Vault, AWS Secrets Manager).

```python
import os
import pika

credentials = pika.PlainCredentials(
    username=os.environ['RABBITMQ_USER'],
    password=os.environ['RABBITMQ_PASSWORD']
)
```

## Мониторинг аутентификации

Логирование неудачных попыток:

```erlang
% Включение расширенного логирования
log.connection.level = info
```

Просмотр логов:

```bash
# Поиск неудачных попыток
grep "authentication failure" /var/log/rabbitmq/rabbit@hostname.log
```

## Резюме

- Используйте механизм EXTERNAL с клиентскими сертификатами для максимальной безопасности
- Комбинируйте backend'ы для гибкой интеграции с корпоративными системами
- Всегда удаляйте или блокируйте пользователя guest в production
- Храните пароли в защищённых хранилищах, не в коде
- Мониторьте неудачные попытки аутентификации

---

[prev: 05-quorum-queues-ha](../09-clustering/05-quorum-queues-ha.md) | [next: 02-authorization](./02-authorization.md)
