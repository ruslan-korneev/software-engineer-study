# CLI инструменты

[prev: 01-management-ui](./01-management-ui.md) | [next: 03-monitoring](./03-monitoring.md)

---

## Обзор

RabbitMQ предоставляет набор мощных CLI инструментов для управления, мониторинга и диагностики брокера сообщений. Основные инструменты:

- **rabbitmqctl** — основная утилита управления
- **rabbitmq-plugins** — управление плагинами
- **rabbitmq-diagnostics** — диагностика и health checks
- **rabbitmqadmin** — HTTP API клиент

## rabbitmqctl

Основная утилита для управления RabbitMQ сервером.

### Управление сервером

```bash
# Статус ноды
rabbitmqctl status

# Остановка приложения RabbitMQ (Erlang VM продолжает работать)
rabbitmqctl stop_app

# Запуск приложения RabbitMQ
rabbitmqctl start_app

# Полная остановка ноды
rabbitmqctl stop

# Сброс ноды (ВНИМАНИЕ: удаляет все данные!)
rabbitmqctl reset

# Принудительный сброс (для recovery)
rabbitmqctl force_reset

# Ротация логов
rabbitmqctl rotate_logs
```

### Управление кластером

```bash
# Присоединение к кластеру
rabbitmqctl stop_app
rabbitmqctl join_cluster rabbit@node1
rabbitmqctl start_app

# Проверка статуса кластера
rabbitmqctl cluster_status

# Удаление ноды из кластера
rabbitmqctl forget_cluster_node rabbit@node2

# Изменение типа ноды
rabbitmqctl change_cluster_node_type disc  # или ram

# Синхронизация очереди
rabbitmqctl sync_queue queue_name

# Отмена синхронизации
rabbitmqctl cancel_sync_queue queue_name

# Обновление metadata кластера
rabbitmqctl update_cluster_nodes rabbit@node1
```

### Управление пользователями

```bash
# Список пользователей
rabbitmqctl list_users

# Добавление пользователя
rabbitmqctl add_user username password

# Удаление пользователя
rabbitmqctl delete_user username

# Изменение пароля
rabbitmqctl change_password username new_password

# Очистка пароля (только для внешней аутентификации)
rabbitmqctl clear_password username

# Назначение тегов (ролей)
rabbitmqctl set_user_tags username administrator

# Доступные теги: management, policymaker, monitoring, administrator

# Аутентификация пользователя (проверка)
rabbitmqctl authenticate_user username password
```

### Управление правами доступа

```bash
# Список прав для пользователя
rabbitmqctl list_user_permissions username

# Список прав для vhost
rabbitmqctl list_permissions -p /production

# Установка прав
# Формат: set_permissions [-p vhost] user conf write read
rabbitmqctl set_permissions -p /production app_user "^app\\..*" "^app\\..*" ".*"

# Очистка прав
rabbitmqctl clear_permissions -p /production app_user

# Установка topic permissions (для topic exchange)
rabbitmqctl set_topic_permissions -p / user exchange "^logs\\." "^logs\\."

# Очистка topic permissions
rabbitmqctl clear_topic_permissions -p / user
```

### Управление Virtual Hosts

```bash
# Список vhosts
rabbitmqctl list_vhosts

# Создание vhost
rabbitmqctl add_vhost /production

# Удаление vhost (ВНИМАНИЕ: удаляет все объекты!)
rabbitmqctl delete_vhost /production

# Установка лимитов для vhost
rabbitmqctl set_vhost_limits -p /production '{"max-connections": 1000, "max-queues": 500}'

# Просмотр лимитов
rabbitmqctl list_vhost_limits -p /production

# Очистка лимитов
rabbitmqctl clear_vhost_limits -p /production
```

### Управление очередями

```bash
# Список очередей
rabbitmqctl list_queues

# Подробный список с параметрами
rabbitmqctl list_queues name messages consumers memory state durable auto_delete

# Очереди в определенном vhost
rabbitmqctl list_queues -p /production name messages

# Удаление очереди
rabbitmqctl delete_queue queue_name

# Очистка очереди (purge)
rabbitmqctl purge_queue queue_name

# Проверка состояния quorum queue
rabbitmqctl list_queues name type leader online
```

### Управление обменниками и bindings

```bash
# Список обменников
rabbitmqctl list_exchanges

# Подробный список
rabbitmqctl list_exchanges name type durable auto_delete internal

# Список bindings
rabbitmqctl list_bindings

# Bindings для определенного vhost
rabbitmqctl list_bindings -p /production source_name source_kind \
  destination_name destination_kind routing_key
```

### Управление соединениями и каналами

```bash
# Список соединений
rabbitmqctl list_connections

# Подробный список соединений
rabbitmqctl list_connections name user vhost state ssl ssl_protocol peer_host

# Закрытие соединения
rabbitmqctl close_connection "<connection_name>" "reason"

# Список каналов
rabbitmqctl list_channels

# Подробный список каналов
rabbitmqctl list_channels connection user vhost prefetch_count consumer_count confirm
```

### Управление политиками

```bash
# Список политик
rabbitmqctl list_policies

# Установка политики
# Формат: set_policy [-p vhost] [--priority N] [--apply-to queues|exchanges|all] name pattern definition
rabbitmqctl set_policy -p / --priority 1 --apply-to queues \
  ha-all "^ha\\." '{"ha-mode":"all","ha-sync-mode":"automatic"}'

# Примеры политик:
# TTL для сообщений
rabbitmqctl set_policy ttl-policy "^ttl\\." '{"message-ttl":60000}'

# Максимальная длина очереди
rabbitmqctl set_policy max-length "^limited\\." '{"max-length":10000}'

# Dead Letter Exchange
rabbitmqctl set_policy dlx "^order\\." '{"dead-letter-exchange":"dlx","dead-letter-routing-key":"failed"}'

# Удаление политики
rabbitmqctl clear_policy -p / ha-all
```

### Parameters и Runtime Parameters

```bash
# Список параметров
rabbitmqctl list_parameters

# Установка параметра
rabbitmqctl set_parameter component_name name value

# Пример: Federation upstream
rabbitmqctl set_parameter federation-upstream my-upstream \
  '{"uri":"amqp://server-name","prefetch-count":1000}'

# Global parameters
rabbitmqctl list_global_parameters
rabbitmqctl set_global_parameter name value
rabbitmqctl clear_global_parameter name
```

## rabbitmq-plugins

Управление плагинами RabbitMQ.

```bash
# Список всех плагинов
rabbitmq-plugins list

# Список включенных плагинов
rabbitmq-plugins list --enabled

# Включение плагина
rabbitmq-plugins enable rabbitmq_management

# Включение с зависимостями (явно)
rabbitmq-plugins enable rabbitmq_shovel rabbitmq_shovel_management

# Отключение плагина
rabbitmq-plugins disable rabbitmq_management

# Включение плагина на всех нодах кластера
rabbitmq-plugins enable rabbitmq_management --all

# Проверка состояния плагина
rabbitmq-plugins is_enabled rabbitmq_management

# Директории плагинов
rabbitmq-plugins directories
```

### Популярные плагины

```bash
# Management UI (веб-интерфейс)
rabbitmq-plugins enable rabbitmq_management

# Prometheus метрики
rabbitmq-plugins enable rabbitmq_prometheus

# Shovel (перемещение сообщений между брокерами)
rabbitmq-plugins enable rabbitmq_shovel rabbitmq_shovel_management

# Federation (федерация кластеров)
rabbitmq-plugins enable rabbitmq_federation rabbitmq_federation_management

# Consistent Hash Exchange
rabbitmq-plugins enable rabbitmq_consistent_hash_exchange

# Delayed Message Exchange
rabbitmq-plugins enable rabbitmq_delayed_message_exchange

# MQTT протокол
rabbitmq-plugins enable rabbitmq_mqtt

# STOMP протокол
rabbitmq-plugins enable rabbitmq_stomp

# Web STOMP (WebSocket)
rabbitmq-plugins enable rabbitmq_web_stomp
```

## rabbitmq-diagnostics

Инструмент диагностики и проверки здоровья.

### Health Checks

```bash
# Базовая проверка здоровья
rabbitmq-diagnostics check_running

# Проверка локального alarms
rabbitmq-diagnostics check_local_alarms

# Проверка доступности порта
rabbitmq-diagnostics check_port_connectivity

# Проверка virtual hosts
rabbitmq-diagnostics check_virtual_hosts

# Комплексная проверка
rabbitmq-diagnostics check_if_node_is_quorum_critical

# Проверка certificate expiration
rabbitmq-diagnostics check_certificate_expiration --within 1month

# Полная проверка ноды (рекомендуется для CI/CD)
rabbitmq-diagnostics -q check_running && \
rabbitmq-diagnostics -q check_local_alarms && \
rabbitmq-diagnostics -q check_port_connectivity && \
rabbitmq-diagnostics -q check_virtual_hosts
```

### Информация о системе

```bash
# Полный статус ноды
rabbitmq-diagnostics status

# Информация об окружении
rabbitmq-diagnostics environment

# Runtime информация (Erlang)
rabbitmq-diagnostics runtime_thread_stats

# Информация об ОС
rabbitmq-diagnostics os_info

# Версия Erlang
rabbitmq-diagnostics erlang_version

# Информация о памяти
rabbitmq-diagnostics memory_breakdown

# Использование памяти по категориям
rabbitmq-diagnostics memory_breakdown --unit mb

# File descriptors
rabbitmq-diagnostics file_handle_breakdown
```

### Диагностика сети

```bash
# Проверка listeners
rabbitmq-diagnostics listeners

# Проверка resolver
rabbitmq-diagnostics resolve_hostname hostname.example.com

# Проверка подключения к ноде
rabbitmq-diagnostics ping

# Проверка связи между нодами
rabbitmq-diagnostics cluster_info
```

### Диагностика очередей

```bash
# Статистика quorum queues
rabbitmq-diagnostics quorum_status queue_name

# Проверка синхронизации mirrors
rabbitmq-diagnostics check_if_node_is_mirror_sync_critical

# Observer CLI (интерактивный мониторинг)
rabbitmq-diagnostics observer
```

### Логирование

```bash
# Текущий уровень логирования
rabbitmq-diagnostics log_location

# Установка уровня логирования на лету
rabbitmq-diagnostics set_log_level debug

# Уровни: debug, info, warning, error, critical, none

# Логирование для конкретной категории
rabbitmq-diagnostics set_log_level connection debug

# Сброс к настройкам по умолчанию
rabbitmq-diagnostics set_log_level info
```

## rabbitmqadmin

CLI клиент для HTTP API (требует включенного management плагина).

### Установка

```bash
# Скачивание с Management UI
wget http://localhost:15672/cli/rabbitmqadmin
chmod +x rabbitmqadmin
sudo mv rabbitmqadmin /usr/local/bin/

# Или через pip
pip install rabbitmqadmin
```

### Конфигурация

Файл `~/.rabbitmqadmin.conf`:

```ini
[default]
hostname = localhost
port = 15672
username = admin
password = secret
vhost = /

[production]
hostname = rabbitmq.prod.example.com
port = 15672
username = prod_admin
password = prod_secret
ssl = true
vhost = /production
```

### Основные команды

```bash
# Использование конфигурации
rabbitmqadmin -N production list queues

# Список команд
rabbitmqadmin help subcommands

# Список очередей
rabbitmqadmin list queues

# Подробный список
rabbitmqadmin list queues name messages consumers

# Список обменников
rabbitmqadmin list exchanges

# Список bindings
rabbitmqadmin list bindings
```

### Создание объектов

```bash
# Создание exchange
rabbitmqadmin declare exchange name=my-exchange type=topic durable=true

# Создание очереди
rabbitmqadmin declare queue name=my-queue durable=true

# Создание binding
rabbitmqadmin declare binding source=my-exchange destination=my-queue routing_key="order.#"

# Создание пользователя
rabbitmqadmin declare user name=app_user password=secret tags=monitoring

# Создание vhost
rabbitmqadmin declare vhost name=/staging

# Создание политики
rabbitmqadmin declare policy name=ha-policy \
  pattern="^ha\." \
  definition='{"ha-mode":"all"}' \
  apply-to=queues
```

### Удаление объектов

```bash
# Удаление очереди
rabbitmqadmin delete queue name=my-queue

# Удаление exchange
rabbitmqadmin delete exchange name=my-exchange

# Удаление binding
rabbitmqadmin delete binding source=my-exchange destination=my-queue properties_key="order.#"
```

### Публикация и получение сообщений

```bash
# Публикация сообщения
rabbitmqadmin publish exchange=my-exchange routing_key=test payload="Hello World"

# Публикация из файла
rabbitmqadmin publish exchange=my-exchange routing_key=test payload="$(cat message.json)"

# С properties
rabbitmqadmin publish exchange=amq.default routing_key=my-queue \
  payload="Hello" \
  properties='{"content_type":"text/plain","delivery_mode":2}'

# Получение сообщений
rabbitmqadmin get queue=my-queue count=10

# Получение с requeue
rabbitmqadmin get queue=my-queue ackmode=ack_requeue_true

# Очистка очереди
rabbitmqadmin purge queue name=my-queue
```

### Экспорт и импорт

```bash
# Экспорт всех определений
rabbitmqadmin export definitions.json

# Импорт определений
rabbitmqadmin import definitions.json

# Экспорт только очередей
rabbitmqadmin list queues > queues.txt
```

## Скрипты и автоматизация

### Backup и Restore

```bash
#!/bin/bash
# backup.sh - Бэкап определений RabbitMQ

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/var/backups/rabbitmq"

mkdir -p $BACKUP_DIR

# Экспорт определений через API
curl -u admin:password \
  http://localhost:15672/api/definitions > \
  $BACKUP_DIR/definitions_$DATE.json

# Или через rabbitmqadmin
rabbitmqadmin export $BACKUP_DIR/definitions_$DATE.json

echo "Backup saved to $BACKUP_DIR/definitions_$DATE.json"
```

### Health Check скрипт

```bash
#!/bin/bash
# health_check.sh - Проверка здоровья RabbitMQ

check_health() {
    rabbitmq-diagnostics -q check_running && \
    rabbitmq-diagnostics -q check_local_alarms && \
    rabbitmq-diagnostics -q check_port_connectivity
}

if check_health; then
    echo "RabbitMQ is healthy"
    exit 0
else
    echo "RabbitMQ health check failed"
    exit 1
fi
```

### Мониторинг очередей

```bash
#!/bin/bash
# queue_monitor.sh - Мониторинг размера очередей

THRESHOLD=10000

rabbitmqctl list_queues name messages --no-table-headers | while read queue messages; do
    if [ "$messages" -gt "$THRESHOLD" ]; then
        echo "WARNING: Queue $queue has $messages messages (threshold: $THRESHOLD)"
        # Отправить alert
    fi
done
```

## Best Practices

1. **Используйте конфигурационные файлы** для rabbitmqadmin вместо передачи credentials в командной строке

2. **Создавайте скрипты для повторяющихся задач** и храните их в системе контроля версий

3. **Используйте флаг -q (quiet)** для скриптов, чтобы получать только результат

4. **Регулярно делайте бэкап определений** через `rabbitmqadmin export`

5. **Используйте health checks** в CI/CD пайплайнах и системах мониторинга

6. **Документируйте политики** и храните их как код:
```bash
# policies.sh
rabbitmqctl set_policy ha-all "^ha\\." '{"ha-mode":"all"}'
rabbitmqctl set_policy ttl "^temp\\." '{"message-ttl":3600000}'
```

## Заключение

CLI инструменты RabbitMQ предоставляют полный контроль над брокером:

- **rabbitmqctl** — полное управление сервером, пользователями, очередями
- **rabbitmq-plugins** — управление плагинами
- **rabbitmq-diagnostics** — диагностика и health checks
- **rabbitmqadmin** — удобный клиент для HTTP API

Комбинирование этих инструментов позволяет автоматизировать все аспекты управления RabbitMQ.

---

[prev: 01-management-ui](./01-management-ui.md) | [next: 03-monitoring](./03-monitoring.md)
