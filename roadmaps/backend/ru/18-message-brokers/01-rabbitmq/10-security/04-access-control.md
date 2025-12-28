# Контроль доступа в RabbitMQ

[prev: 03-tls-ssl](./03-tls-ssl.md) | [next: 01-management-ui](../11-management/01-management-ui.md)

---

## Введение

Контроль доступа в RabbitMQ — это комплексная система управления пользователями, виртуальными хостами (vhosts) и ограничениями ресурсов. Правильная настройка контроля доступа обеспечивает изоляцию приложений и защиту от несанкционированного использования ресурсов.

## Управление пользователями

### Создание и настройка пользователей

```bash
# Создание пользователя
rabbitmqctl add_user username password

# Изменение пароля
rabbitmqctl change_password username new_password

# Удаление пользователя
rabbitmqctl delete_user username

# Список всех пользователей
rabbitmqctl list_users
```

### Стратегии именования пользователей

```bash
# По сервисам
rabbitmqctl add_user order-service $(openssl rand -base64 24)
rabbitmqctl add_user payment-service $(openssl rand -base64 24)
rabbitmqctl add_user notification-service $(openssl rand -base64 24)

# По средам
rabbitmqctl add_user app-dev $(openssl rand -base64 24)
rabbitmqctl add_user app-staging $(openssl rand -base64 24)
rabbitmqctl add_user app-prod $(openssl rand -base64 24)

# Комбинированный подход
rabbitmqctl add_user order-service-prod $(openssl rand -base64 24)
rabbitmqctl add_user order-service-dev $(openssl rand -base64 24)
```

### Пользователь guest

По умолчанию RabbitMQ создаёт пользователя `guest` с паролем `guest`:

```erlang
% rabbitmq.conf
% Ограничение guest только локальными подключениями (по умолчанию)
loopback_users.guest = true

% Разрешить guest подключаться удалённо (НЕ РЕКОМЕНДУЕТСЯ!)
% loopback_users.guest = false
```

**Рекомендация для production**:

```bash
# Удалите пользователя guest
rabbitmqctl delete_user guest

# Или измените пароль на надёжный
rabbitmqctl change_password guest $(openssl rand -base64 32)
```

## Роли и теги

### Системные теги

| Тег | Возможности |
|-----|-------------|
| `administrator` | Полный контроль над всей системой |
| `monitoring` | Доступ к мониторингу и метрикам |
| `policymaker` | Управление политиками и параметрами |
| `management` | Базовый доступ к Management UI |
| (без тегов) | Только AMQP операции без доступа к UI |

### Назначение тегов

```bash
# Администратор системы
rabbitmqctl set_user_tags admin administrator

# Оператор мониторинга
rabbitmqctl set_user_tags monitoring-user monitoring

# Менеджер политик (не администратор)
rabbitmqctl set_user_tags policy-manager policymaker management

# Пользователь только для AMQP (сервисы)
rabbitmqctl set_user_tags order-service
# Пустой список тегов — нет доступа к Management UI
```

### Создание ролей через скрипты

```bash
#!/bin/bash
# create_service_user.sh

SERVICE_NAME=$1
VHOST=$2
PASSWORD=$(openssl rand -base64 24)

# Создание пользователя без тегов
rabbitmqctl add_user "$SERVICE_NAME" "$PASSWORD"

# Минимальные права
rabbitmqctl set_permissions -p "$VHOST" "$SERVICE_NAME" \
    "^${SERVICE_NAME}\\..*" \
    "^${SERVICE_NAME}\\..*|^amq\\..*" \
    "^${SERVICE_NAME}\\..*"

echo "User: $SERVICE_NAME"
echo "Password: $PASSWORD"
echo "Vhost: $VHOST"
```

## Virtual Hosts (vhosts)

### Концепция vhosts

Vhosts обеспечивают логическую изоляцию ресурсов RabbitMQ:
- Каждый vhost имеет свой набор очередей, обменников и привязок
- Пользователи могут иметь разные права в разных vhosts
- Сообщения не пересекают границы vhosts

### Управление vhosts

```bash
# Создание vhost
rabbitmqctl add_vhost /production
rabbitmqctl add_vhost /staging
rabbitmqctl add_vhost /development

# С описанием и тегами
rabbitmqctl add_vhost /production \
    --description "Production environment" \
    --default-queue-type quorum \
    --tags production,critical

# Список vhosts
rabbitmqctl list_vhosts name description tags

# Удаление vhost (УДАЛЯЕТ ВСЕ ДАННЫЕ!)
rabbitmqctl delete_vhost /old-environment
```

### Стратегии организации vhosts

#### По средам

```bash
rabbitmqctl add_vhost /development
rabbitmqctl add_vhost /staging
rabbitmqctl add_vhost /production
```

#### По приложениям/командам

```bash
rabbitmqctl add_vhost /team-orders
rabbitmqctl add_vhost /team-payments
rabbitmqctl add_vhost /team-notifications
```

#### По клиентам (multi-tenant)

```bash
rabbitmqctl add_vhost /tenant-company-a
rabbitmqctl add_vhost /tenant-company-b
rabbitmqctl add_vhost /tenant-company-c
```

### Настройка прав на vhost

```bash
# Полные права в production для admin
rabbitmqctl set_permissions -p /production admin ".*" ".*" ".*"

# Только чтение для мониторинга
rabbitmqctl set_permissions -p /production monitoring "" "" ".*"

# Конкретные права для сервиса
rabbitmqctl set_permissions -p /production order-service \
    "^order\\.(temp\\..*|dlq)$" \
    "^order\\..*|^amq\\..*" \
    "^order\\..*"
```

## Ограничения ресурсов

### Лимиты на vhost

```bash
# Максимальное количество очередей
rabbitmqctl set_vhost_limits -p /development \
    '{"max-queues": 100}'

# Максимальное количество соединений
rabbitmqctl set_vhost_limits -p /development \
    '{"max-connections": 50}'

# Комбинированные лимиты
rabbitmqctl set_vhost_limits -p /tenant-small \
    '{"max-queues": 50, "max-connections": 20}'

# Просмотр лимитов
rabbitmqctl list_vhost_limits

# Удаление лимитов
rabbitmqctl clear_vhost_limits -p /development
```

### Лимиты на пользователя

```bash
# Максимальное количество соединений для пользователя
rabbitmqctl set_user_limits order-service \
    '{"max-connections": 10}'

# Максимальное количество каналов
rabbitmqctl set_user_limits order-service \
    '{"max-channels": 50}'

# Комбинированные лимиты
rabbitmqctl set_user_limits batch-processor \
    '{"max-connections": 5, "max-channels": 100}'

# Просмотр лимитов
rabbitmqctl list_user_limits

# Удаление лимитов
rabbitmqctl clear_user_limits order-service
```

### Глобальные лимиты

```erlang
% rabbitmq.conf

% Максимальное количество каналов на соединение
channel_max = 128

% Размер фрейма
frame_max = 131072

% Heartbeat
heartbeat = 60

% Максимальный размер сообщения
max_message_size = 134217728  # 128 MB
```

## Memory и Disk Alarms

### Настройка порогов памяти

```erlang
% rabbitmq.conf

% Абсолютное значение
vm_memory_high_watermark.absolute = 2GB

% Относительное значение (от общей памяти системы)
vm_memory_high_watermark.relative = 0.4

% Paging порог (когда начинается сброс на диск)
vm_memory_high_watermark_paging_ratio = 0.5
```

### Настройка порогов диска

```erlang
% rabbitmq.conf

% Абсолютное значение
disk_free_limit.absolute = 5GB

% Относительное значение (от общего объёма RAM)
disk_free_limit.relative = 1.5
```

### Мониторинг алармов

```bash
# Проверка статуса алармов
rabbitmqctl status | grep -A 5 "Alarms"

# Через API
curl -u admin:password http://localhost:15672/api/nodes | \
    jq '.[].mem_alarm, .[].disk_free_alarm'
```

## Политики доступа

### Политики для очередей

```bash
# Ограничение размера очереди
rabbitmqctl set_policy -p /production queue-limits \
    "^app\\..*" \
    '{"max-length": 100000, "overflow": "reject-publish"}' \
    --priority 10 \
    --apply-to queues

# TTL для сообщений
rabbitmqctl set_policy -p /production message-ttl \
    "^temp\\..*" \
    '{"message-ttl": 3600000}' \
    --apply-to queues

# Автоудаление очередей
rabbitmqctl set_policy -p /production auto-delete \
    "^ephemeral\\..*" \
    '{"expires": 1800000}' \
    --apply-to queues
```

### Политики для HA

```bash
# Quorum очереди для критических данных
rabbitmqctl set_policy -p /production ha-critical \
    "^critical\\..*" \
    '{"queue-mode": "lazy", "ha-mode": "all"}' \
    --priority 100 \
    --apply-to queues
```

## Operator Policies

Operator policies имеют приоритет над обычными политиками и используются для установки ограничений уровня оператора:

```bash
# Максимальная длина очереди (operator policy)
rabbitmqctl set_operator_policy -p /production max-queue-length \
    ".*" \
    '{"max-length": 500000}' \
    --apply-to queues

# Принудительная репликация
rabbitmqctl set_operator_policy -p /production force-ha \
    ".*" \
    '{"ha-mode": "exactly", "ha-params": 2}' \
    --apply-to queues
```

## Аудит и мониторинг

### Логирование действий

```erlang
% rabbitmq.conf

% Уровень логирования
log.console.level = info
log.file.level = info

% Логирование соединений
log.connection.level = info

% Логирование каналов
log.channel.level = warning
```

### Скрипт аудита

```bash
#!/bin/bash
# audit_rabbitmq.sh

OUTPUT_DIR="/var/log/rabbitmq/audit"
DATE=$(date +%Y-%m-%d_%H-%M-%S)
REPORT="$OUTPUT_DIR/audit_$DATE.txt"

mkdir -p "$OUTPUT_DIR"

echo "=== RabbitMQ Security Audit ===" > "$REPORT"
echo "Date: $(date)" >> "$REPORT"
echo "" >> "$REPORT"

echo "=== Users ===" >> "$REPORT"
rabbitmqctl list_users >> "$REPORT"
echo "" >> "$REPORT"

echo "=== Vhosts ===" >> "$REPORT"
rabbitmqctl list_vhosts name description >> "$REPORT"
echo "" >> "$REPORT"

echo "=== Permissions ===" >> "$REPORT"
for vhost in $(rabbitmqctl list_vhosts --quiet); do
    echo "--- Vhost: $vhost ---" >> "$REPORT"
    rabbitmqctl list_permissions -p "$vhost" >> "$REPORT"
done
echo "" >> "$REPORT"

echo "=== Active Connections ===" >> "$REPORT"
rabbitmqctl list_connections user vhost ssl ssl_protocol >> "$REPORT"
echo "" >> "$REPORT"

echo "=== Resource Limits ===" >> "$REPORT"
echo "Vhost limits:" >> "$REPORT"
rabbitmqctl list_vhost_limits >> "$REPORT"
echo "User limits:" >> "$REPORT"
rabbitmqctl list_user_limits >> "$REPORT"

echo "Audit complete: $REPORT"
```

### Мониторинг через API

```python
import requests
from requests.auth import HTTPBasicAuth

class RabbitMQAuditor:
    def __init__(self, host, username, password):
        self.base_url = f"http://{host}:15672/api"
        self.auth = HTTPBasicAuth(username, password)

    def get_users(self):
        response = requests.get(f"{self.base_url}/users", auth=self.auth)
        return response.json()

    def get_permissions(self):
        response = requests.get(f"{self.base_url}/permissions", auth=self.auth)
        return response.json()

    def check_guest_user(self):
        users = self.get_users()
        for user in users:
            if user['name'] == 'guest':
                print("WARNING: guest user exists!")
                return True
        return False

    def check_wildcard_permissions(self):
        permissions = self.get_permissions()
        wildcards = []
        for perm in permissions:
            if perm['configure'] == '.*' and perm['write'] == '.*' and perm['read'] == '.*':
                wildcards.append({
                    'user': perm['user'],
                    'vhost': perm['vhost']
                })
        if wildcards:
            print(f"WARNING: Users with full permissions: {wildcards}")
        return wildcards

    def audit(self):
        print("=== RabbitMQ Security Audit ===")
        self.check_guest_user()
        self.check_wildcard_permissions()
        # Дополнительные проверки...

# Использование
auditor = RabbitMQAuditor('localhost', 'admin', 'password')
auditor.audit()
```

## Best Practices

### 1. Принцип наименьших привилегий

```bash
# Плохо: полные права
rabbitmqctl set_permissions -p /production service ".*" ".*" ".*"

# Хорошо: минимальные необходимые права
rabbitmqctl set_permissions -p /production order-service \
    "^order\\.temp\\..*" \
    "^order\\.events$" \
    "^order\\.tasks$"
```

### 2. Изоляция сред

```bash
# Отдельные vhosts для разных сред
rabbitmqctl add_vhost /prod
rabbitmqctl add_vhost /staging
rabbitmqctl add_vhost /dev

# Разработчики не имеют доступа к prod
rabbitmqctl set_permissions -p /dev developer ".*" ".*" ".*"
rabbitmqctl set_permissions -p /staging developer "" "" ".*"
# Нет прав на /prod
```

### 3. Регулярный аудит

```bash
# Cron задача для аудита
0 6 * * * /opt/scripts/audit_rabbitmq.sh
```

### 4. Автоматизация через IaC

```yaml
# Terraform пример
resource "rabbitmq_user" "order_service" {
  name     = "order-service"
  password = var.order_service_password
  tags     = []
}

resource "rabbitmq_permissions" "order_service_prod" {
  user  = rabbitmq_user.order_service.name
  vhost = rabbitmq_vhost.production.name

  permissions {
    configure = "^order\\.temp\\..*"
    write     = "^order\\..*"
    read      = "^order\\..*"
  }
}
```

## Распространённые уязвимости

### 1. Открытый Management UI

**Проблема**: Management UI доступен из интернета.

**Решение**: Используйте firewall или VPN:

```bash
# iptables правило
iptables -A INPUT -p tcp --dport 15672 -s 10.0.0.0/8 -j ACCEPT
iptables -A INPUT -p tcp --dport 15672 -j DROP
```

### 2. Отсутствие лимитов ресурсов

**Проблема**: Один клиент может исчерпать все ресурсы.

**Решение**: Установите лимиты на vhost и пользователей.

### 3. Слишком широкие права

**Проблема**: Все сервисы имеют полные права.

**Решение**: Внедрите принцип минимальных привилегий.

## Резюме

- Удаляйте или блокируйте пользователя guest в production
- Создавайте отдельного пользователя для каждого сервиса
- Используйте vhosts для изоляции сред и приложений
- Устанавливайте лимиты на соединения, каналы и очереди
- Применяйте политики для контроля размера очередей и TTL
- Регулярно проводите аудит прав доступа
- Автоматизируйте управление доступом через IaC

---

[prev: 03-tls-ssl](./03-tls-ssl.md) | [next: 01-management-ui](../11-management/01-management-ui.md)
