# Management UI

## Обзор

RabbitMQ Management UI — это веб-интерфейс для управления и мониторинга RabbitMQ сервера. Он предоставляет удобный графический интерфейс для выполнения административных задач, просмотра статистики и диагностики проблем.

## Установка и включение

Management UI поставляется как плагин, который нужно включить:

```bash
# Включение плагина Management
rabbitmq-plugins enable rabbitmq_management

# Проверка включенных плагинов
rabbitmq-plugins list --enabled

# Для кластера — включить на всех нодах
rabbitmq-plugins enable rabbitmq_management --all
```

После включения плагина интерфейс доступен по адресу:
- **URL**: `http://localhost:15672/`
- **Логин по умолчанию**: `guest`
- **Пароль по умолчанию**: `guest`

> **Важно**: Пользователь `guest` по умолчанию может подключаться только с localhost. Для удаленного доступа создайте нового пользователя.

## Структура интерфейса

### Главная страница (Overview)

Главная страница отображает общую информацию о состоянии брокера:

```
┌─────────────────────────────────────────────────────────────┐
│  Overview  │  Connections  │  Channels  │  Exchanges  │  Queues  │  Admin  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Totals:                      Message rates:                │
│  ├── Queued messages: 1,234   ├── Publish: 150/s            │
│  ├── Connections: 45          ├── Deliver: 145/s            │
│  ├── Channels: 120            └── Acknowledge: 145/s        │
│  └── Consumers: 30                                          │
│                                                             │
│  Nodes:                                                     │
│  ├── rabbit@node1 (running)   Memory: 512MB / 2GB           │
│  └── rabbit@node2 (running)   Disk: 50GB free               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Вкладка Connections

Отображает все активные соединения с брокером:

| Поле | Описание |
|------|----------|
| Name | Имя соединения |
| User name | Пользователь |
| State | Состояние (running, blocking, blocked) |
| SSL/TLS | Использование шифрования |
| Channels | Количество каналов |
| From client | Трафик от клиента |
| To client | Трафик к клиенту |

### Вкладка Channels

Показывает все открытые каналы:

```
Channel         | Virtual host | User   | Mode    | Prefetch | Unacked
----------------+--------------+--------+---------+----------+---------
172.17.0.1:5672 | /            | admin  | confirm | 10       | 5
192.168.1.5:443 | /prod        | worker | normal  | 50       | 0
```

### Вкладка Exchanges

Управление обменниками:

- **Создание обменника**:
  - Name: имя обменника
  - Type: direct, topic, fanout, headers
  - Durability: durable/transient
  - Auto delete: автоматическое удаление
  - Internal: только для внутреннего использования

- **Просмотр bindings**: какие очереди привязаны к обменнику

### Вкладка Queues

Полное управление очередями:

```
Queue      | Type    | Features | State   | Ready | Unacked | Total
-----------+---------+----------+---------+-------+---------+-------
orders     | classic | D        | running | 1500  | 50      | 1550
emails     | quorum  | D        | running | 0     | 0       | 0
analytics  | classic | D AD     | idle    | 50000 | 0       | 50000

D = Durable, AD = Auto-delete
```

**Операции с очередями через UI**:
- Publish message — отправить тестовое сообщение
- Get messages — получить сообщения (с ack или без)
- Purge — очистить очередь
- Delete — удалить очередь

### Вкладка Admin

Управление пользователями и правами:

```
Users:
├── admin     [administrator]
├── producer  [management, policymaker]
├── consumer  [monitoring]
└── guest     [administrator] (localhost only)

Virtual Hosts:
├── /
├── /production
└── /development
```

## Создание пользователей через UI

1. Перейти в Admin → Users
2. Нажать "Add a user"
3. Заполнить форму:

```
Username: app_user
Password: ********
Tags: monitoring, management

# Tags определяют уровень доступа:
# - none: только AMQP доступ
# - management: доступ к UI (только свои vhost)
# - policymaker: управление политиками
# - monitoring: просмотр всех данных
# - administrator: полный доступ
```

## Настройка прав доступа

Права задаются для каждого virtual host:

```
Virtual Host: /production
User: app_user

Configure regexp: ^app\..*     # Может создавать объекты с префиксом "app."
Write regexp:     ^app\..*     # Может публиковать в объекты с префиксом "app."
Read regexp:      .*           # Может читать из всех объектов
```

## Конфигурация Management UI

### Изменение порта

В файле `rabbitmq.conf`:

```ini
# Порт для Management UI
management.tcp.port = 15672

# Привязка к определенному IP
management.tcp.ip = 0.0.0.0

# SSL для Management UI
management.ssl.port = 15671
management.ssl.cacertfile = /path/to/ca_certificate.pem
management.ssl.certfile = /path/to/server_certificate.pem
management.ssl.keyfile = /path/to/server_key.pem
```

### Настройка аутентификации

```ini
# Время жизни сессии (в минутах)
management.login_session_timeout = 60

# Разрешить доступ guest с любого хоста (не рекомендуется для production)
loopback_users = none

# Отключить guest полностью
loopback_users.guest = false
```

### Ограничение отображаемых данных

```ini
# Максимальное количество выборок для графиков
management.sample_retention_policies.global.minute = 5
management.sample_retention_policies.global.hour = 60
management.sample_retention_policies.global.day = 1200

# Лимит отображаемых объектов
management.max_queues = 10000
management.max_connections = 10000
```

## HTTP API

Management UI предоставляет REST API для автоматизации:

```bash
# Получить информацию о кластере
curl -u admin:password http://localhost:15672/api/overview

# Список очередей
curl -u admin:password http://localhost:15672/api/queues

# Создать очередь
curl -u admin:password -X PUT \
  -H "Content-Type: application/json" \
  -d '{"durable":true,"auto_delete":false}' \
  http://localhost:15672/api/queues/%2F/my-queue

# Опубликовать сообщение
curl -u admin:password -X POST \
  -H "Content-Type: application/json" \
  -d '{"properties":{},"routing_key":"my-queue","payload":"Hello","payload_encoding":"string"}' \
  http://localhost:15672/api/exchanges/%2F/amq.default/publish

# Получить сообщения из очереди
curl -u admin:password -X POST \
  -H "Content-Type: application/json" \
  -d '{"count":5,"ackmode":"ack_requeue_false","encoding":"auto"}' \
  http://localhost:15672/api/queues/%2F/my-queue/get
```

## Policies (Политики)

Политики позволяют централизованно управлять настройками очередей и обменников:

```
Policy:
  Name: ha-all
  Pattern: ^ha\..*
  Apply to: queues
  Priority: 0

  Definition:
    ha-mode: all
    ha-sync-mode: automatic
    message-ttl: 3600000
    max-length: 100000
```

## Best Practices

### Безопасность

1. **Никогда не используйте guest в production**:
```bash
rabbitmqctl delete_user guest
rabbitmqctl add_user admin secure_password
rabbitmqctl set_user_tags admin administrator
```

2. **Используйте SSL/TLS**:
```ini
management.ssl.port = 15671
management.ssl.cacertfile = /etc/rabbitmq/ssl/ca.pem
management.ssl.certfile = /etc/rabbitmq/ssl/server.pem
management.ssl.keyfile = /etc/rabbitmq/ssl/server-key.pem
```

3. **Ограничьте доступ по IP**:
```ini
# Использовать reverse proxy (nginx) для ограничения доступа
management.tcp.ip = 127.0.0.1
```

### Производительность

1. **Отключите статистику для высоконагруженных систем**:
```ini
management.rates_mode = none  # Отключает rate calculations
```

2. **Уменьшите частоту обновления**:
```ini
collect_statistics_interval = 10000  # 10 секунд вместо 5
```

### Мониторинг через UI

- Регулярно проверяйте вкладку Overview на наличие проблем
- Следите за Memory и Disk alarms
- Мониторьте количество unacked сообщений
- Отслеживайте рост очередей

## Troubleshooting

### UI не загружается

```bash
# Проверить статус плагина
rabbitmq-plugins list | grep management

# Проверить порт
netstat -tlnp | grep 15672

# Проверить логи
tail -f /var/log/rabbitmq/rabbit@hostname.log
```

### Ошибка аутентификации

```bash
# Проверить пользователя
rabbitmqctl list_users

# Сбросить пароль
rabbitmqctl change_password admin new_password

# Проверить права
rabbitmqctl list_user_permissions admin
```

## Заключение

Management UI — незаменимый инструмент для администрирования RabbitMQ. Он предоставляет:

- Визуальный мониторинг состояния кластера
- Управление пользователями и правами
- Создание и настройку очередей и обменников
- REST API для автоматизации
- Диагностику проблем

Для production-окружений рекомендуется использовать SSL, создавать отдельных пользователей с минимальными правами и настраивать мониторинг через API.
