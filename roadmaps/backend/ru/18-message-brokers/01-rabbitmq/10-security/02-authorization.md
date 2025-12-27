# Авторизация в RabbitMQ

## Введение

Авторизация — это процесс определения, какие действия может выполнять аутентифицированный пользователь. В RabbitMQ авторизация охватывает доступ к vhosts, права на ресурсы (очереди, обменники) и тематическую авторизацию.

## Модель прав доступа

RabbitMQ использует модель прав доступа, основанную на трёх типах операций:

| Операция | Описание | Примеры |
|----------|----------|---------|
| **configure** | Создание и удаление ресурсов | Объявление очередей, обменников, привязок |
| **write** | Публикация сообщений | Publish в exchange |
| **read** | Получение сообщений | Consume из очереди, привязка к exchange |

## Permissions (Права доступа)

### Структура прав

Права задаются в виде регулярных выражений для каждого типа операции:

```bash
rabbitmqctl set_permissions [-p vhost] user configure_pattern write_pattern read_pattern
```

### Примеры настройки прав

```bash
# Полные права на все ресурсы в vhost
rabbitmqctl set_permissions -p /production admin ".*" ".*" ".*"

# Только чтение (consumer)
rabbitmqctl set_permissions -p /production reader "" "" ".*"

# Только запись (publisher)
rabbitmqctl set_permissions -p /production writer "" ".*" ""

# Права на конкретные очереди
rabbitmqctl set_permissions -p /production order-service \
    "^order\\..*" \
    "^order\\..*" \
    "^order\\..*"

# Права на создание временных очередей (amq.gen-*)
rabbitmqctl set_permissions -p /production rpc-client \
    "^amq\\.gen-.*" \
    "^rpc\\.requests$" \
    "^amq\\.gen-.*"
```

### Просмотр и удаление прав

```bash
# Список прав пользователя
rabbitmqctl list_user_permissions username

# Список всех прав в vhost
rabbitmqctl list_permissions -p /production

# Удаление прав
rabbitmqctl clear_permissions -p /production username
```

## Tags (Теги пользователей)

Теги определяют доступ к административным функциям RabbitMQ.

### Доступные теги

| Тег | Описание |
|-----|----------|
| `administrator` | Полный доступ ко всем функциям, включая управление пользователями и кластером |
| `monitoring` | Доступ к Management UI для мониторинга (только чтение) |
| `policymaker` | Создание и управление политиками и параметрами |
| `management` | Базовый доступ к Management UI (управление своими vhosts) |
| `impersonator` | Возможность использовать механизм AMQP impersonation |

### Назначение тегов

```bash
# Назначение одного тега
rabbitmqctl set_user_tags myuser administrator

# Назначение нескольких тегов
rabbitmqctl set_user_tags myuser monitoring policymaker

# Удаление всех тегов
rabbitmqctl set_user_tags myuser

# Просмотр тегов
rabbitmqctl list_users
```

### Иерархия тегов

```
administrator
    └── policymaker
    └── monitoring
        └── management
```

Пользователь с тегом `administrator` имеет все права `monitoring`, `policymaker` и `management`.

## Topic Authorization

Тематическая авторизация позволяет контролировать доступ к сообщениям на основе routing key при использовании topic exchanges.

### Включение topic authorization

```erlang
% rabbitmq.conf
auth_backends.1.authn = internal
auth_backends.1.authz = internal

% Для LDAP
auth_ldap.topic_access_query.query = {for, [{permission, {in_group, "cn=${permission},ou=acls,dc=example,dc=com"}}]}
```

### Настройка через HTTP backend

При использовании HTTP backend, topic authorization настраивается через отдельный endpoint:

```erlang
auth_http.topic_path = http://auth-service:8080/auth/topic
```

Запрос к сервису авторизации:

```json
{
    "username": "order-service",
    "vhost": "/production",
    "resource": "topic",
    "name": "events",
    "permission": "write",
    "routing_key": "orders.created.eu"
}
```

Пример реализации:

```python
from flask import Flask, request

app = Flask(__name__)

TOPIC_RULES = {
    'order-service': {
        'write': ['orders.*', 'orders.created.*'],
        'read': ['payments.confirmed.*', 'inventory.updated.*']
    },
    'payment-service': {
        'write': ['payments.*'],
        'read': ['orders.created.*']
    }
}

@app.route('/auth/topic', methods=['POST'])
def check_topic():
    username = request.form.get('username')
    permission = request.form.get('permission')
    routing_key = request.form.get('routing_key')

    rules = TOPIC_RULES.get(username, {})
    patterns = rules.get(permission, [])

    for pattern in patterns:
        if match_routing_key(routing_key, pattern):
            return 'allow'

    return 'deny'

def match_routing_key(key, pattern):
    """Сопоставление routing key с паттерном"""
    import re
    # Преобразуем AMQP паттерн в regex
    regex = pattern.replace('.', r'\.').replace('*', r'[^.]+').replace('#', r'.*')
    return re.match(f'^{regex}$', key) is not None
```

## Авторизация через политики

Политики позволяют устанавливать правила для групп ресурсов:

```bash
# Создание политики с ограничениями
rabbitmqctl set_policy -p /production max-length-policy \
    "^bounded\\..*" \
    '{"max-length": 10000, "overflow": "reject-publish"}' \
    --priority 10 \
    --apply-to queues
```

## Авторизация в кластере

### Federation и Shovel

Для cross-cluster связей требуются специальные права:

```bash
# Пользователь для federation
rabbitmqctl add_user federation_user strong_password
rabbitmqctl set_permissions -p /production federation_user \
    "^federation:.*" \
    "^federation:.*|^amq\\..*" \
    "^amq\\..*"
```

### Кластерные операции

Только пользователи с тегом `administrator` могут:
- Добавлять/удалять узлы кластера
- Управлять параметрами кластера
- Настраивать репликацию

## Авторизация в Management UI

### Ограничение доступа по vhost

```python
# HTTP backend для ограничения доступа к vhosts
@app.route('/auth/vhost', methods=['POST'])
def check_vhost():
    username = request.form.get('username')
    vhost = request.form.get('vhost')

    # Пользователи могут видеть только свои vhosts
    allowed_vhosts = get_user_vhosts(username)

    if vhost in allowed_vhosts:
        return 'allow'
    return 'deny'
```

### Ограничение API endpoints

```erlang
% Отключение опасных операций через Management API
management.disable_stats = false
management.enable_queue_totals = true

% Ограничение rate limiting
management.rates_mode = basic
```

## Best Practices

### 1. Принцип минимальных привилегий

```bash
# Только необходимые права
rabbitmqctl set_permissions -p /production order-consumer \
    "" \                           # Нет права создавать
    "" \                           # Нет права публиковать
    "^orders\\.processed$"         # Чтение только из одной очереди
```

### 2. Разделение прав по средам

```bash
# Разные vhosts для разных сред
rabbitmqctl add_vhost /development
rabbitmqctl add_vhost /staging
rabbitmqctl add_vhost /production

# Разработчик имеет полные права только в development
rabbitmqctl set_permissions -p /development developer ".*" ".*" ".*"
rabbitmqctl set_permissions -p /staging developer "" "" ".*"
# Нет прав в production
```

### 3. Аудит изменений прав

```bash
#!/bin/bash
# Скрипт для аудита прав
echo "=== Permission Audit Report ===" > /var/log/rabbitmq/audit.log
echo "Date: $(date)" >> /var/log/rabbitmq/audit.log

for vhost in $(rabbitmqctl list_vhosts --quiet); do
    echo "\n=== Vhost: $vhost ===" >> /var/log/rabbitmq/audit.log
    rabbitmqctl list_permissions -p "$vhost" >> /var/log/rabbitmq/audit.log
done
```

### 4. Использование групп пользователей (через LDAP)

```erlang
% Группы LDAP для управления правами
auth_ldap.tag_queries.administrator.dn_group = cn=rabbitmq-admins,ou=groups,dc=example,dc=com
auth_ldap.tag_queries.monitoring.dn_group = cn=rabbitmq-monitoring,ou=groups,dc=example,dc=com
auth_ldap.tag_queries.management.dn_group = cn=rabbitmq-users,ou=groups,dc=example,dc=com
```

## Распространённые уязвимости

### 1. Слишком широкие права

**Проблема**:
```bash
# Плохо: все права на все ресурсы
rabbitmqctl set_permissions -p / app-user ".*" ".*" ".*"
```

**Решение**:
```bash
# Хорошо: конкретные паттерны
rabbitmqctl set_permissions -p /production app-user \
    "^app\\.temp\\..+" \
    "^app\\.events$" \
    "^app\\.tasks$"
```

### 2. Отсутствие topic authorization

**Проблема**: Любой publisher может отправить сообщение с любым routing key.

**Решение**: Внедрить topic authorization через HTTP backend.

### 3. Общие учётные записи

**Проблема**: Несколько приложений используют одного пользователя.

**Решение**: Создать отдельного пользователя для каждого сервиса с минимальными правами.

## Мониторинг авторизации

### Логирование отказов

```erlang
% Включение детального логирования
log.channel.level = info
```

### Метрики авторизации

```bash
# Проверка через API
curl -u admin:password \
    http://localhost:15672/api/permissions | jq '.[] | {user, vhost, configure, write, read}'
```

## Резюме

- Используйте принцип минимальных привилегий при назначении прав
- Разделяйте теги: monitoring для мониторинга, policymaker для политик
- Внедряйте topic authorization для granular control над routing keys
- Создавайте отдельных пользователей для каждого сервиса
- Регулярно проводите аудит прав доступа
- Используйте LDAP для централизованного управления группами
