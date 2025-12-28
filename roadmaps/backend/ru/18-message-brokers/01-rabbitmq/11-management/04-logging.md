# Логирование

[prev: 03-monitoring](./03-monitoring.md) | [next: 05-metrics](./05-metrics.md)

---

## Обзор

Логирование в RabbitMQ — критически важный инструмент для диагностики проблем, аудита безопасности и мониторинга работы брокера. RabbitMQ поддерживает гибкую систему логирования с различными уровнями, форматами и destinations.

## Расположение логов

### По умолчанию

```bash
# Linux (package installation)
/var/log/rabbitmq/rabbit@hostname.log

# Linux (generic Unix)
$RABBITMQ_HOME/var/log/rabbitmq/

# macOS (Homebrew)
/usr/local/var/log/rabbitmq/

# Windows
%APPDATA%\RabbitMQ\log\

# Docker
# Логи выводятся в stdout/stderr
docker logs rabbitmq-container
```

### Проверка расположения

```bash
# Показать путь к логам
rabbitmq-diagnostics log_location

# Вывод:
# Log file(s) location(s): /var/log/rabbitmq/rabbit@hostname.log
```

## Конфигурация логирования

### Базовая конфигурация (rabbitmq.conf)

```ini
# Уровень логирования по умолчанию
log.default.level = info

# Логирование в файл
log.file = true
log.file.level = info
log.file.path = /var/log/rabbitmq/rabbit.log

# Ротация логов
log.file.rotation.date = $D0  # Ежедневно в полночь
log.file.rotation.size = 10485760  # 10 MB
log.file.rotation.count = 10  # Хранить 10 файлов

# Логирование в консоль (stdout)
log.console = true
log.console.level = info

# Логирование в syslog
log.syslog = true
log.syslog.level = warning
log.syslog.identity = rabbitmq
log.syslog.facility = daemon
```

### Расширенная конфигурация (advanced.config)

```erlang
[
  {rabbit, [
    {log, [
      {file, [{file, "/var/log/rabbitmq/rabbit.log"},
              {level, info},
              {date, "$D0"},
              {size, 10485760},
              {count, 10}]},
      {console, [{enabled, true},
                 {level, warning}]},
      {syslog, [{enabled, true},
                {level, warning},
                {facility, daemon},
                {identity, "rabbitmq"}]}
    ]}
  ]}
].
```

## Уровни логирования

### Доступные уровни

| Уровень | Описание | Использование |
|---------|----------|---------------|
| debug | Отладочная информация | Детальная диагностика |
| info | Информационные сообщения | Нормальная работа |
| warning | Предупреждения | Потенциальные проблемы |
| error | Ошибки | Проблемы, требующие внимания |
| critical | Критические ошибки | Серьезные сбои |
| none | Отключить логирование | - |

### Изменение уровня на лету

```bash
# Установить глобальный уровень
rabbitmqctl set_log_level debug

# Через diagnostics
rabbitmq-diagnostics set_log_level debug

# Установить уровень для категории
rabbitmqctl set_log_level connection debug

# Сброс к настройкам из конфигурации
rabbitmqctl set_log_level info
```

### Категории логирования

```ini
# Конфигурация уровней по категориям
log.default.level = info

# Соединения
log.connection.level = warning

# Каналы
log.channel.level = warning

# AMQP protocol
log.queue.level = info

# Mirroring (classic HA)
log.mirroring.level = warning

# Federation
log.federation.level = info

# Shovel
log.shovel.level = info

# LDAP authentication
log.ldap.level = warning

# Upgrade процессы
log.upgrade.level = info

# Raft (quorum queues)
log.ra.level = warning
```

## Structured Logging (JSON)

### Включение JSON формата

```ini
# rabbitmq.conf
log.file.formatter = json
log.console.formatter = json
```

### Пример JSON лога

```json
{
  "time": "2024-01-15T10:30:45.123456Z",
  "level": "info",
  "msg": "accepting AMQP connection",
  "connection": "172.17.0.1:52456 -> 172.17.0.2:5672",
  "vhost": "/",
  "user": "admin",
  "node": "rabbit@hostname"
}
```

```json
{
  "time": "2024-01-15T10:30:46.789012Z",
  "level": "warning",
  "msg": "message unroutable",
  "exchange": "orders",
  "routing_key": "unknown.key",
  "vhost": "/",
  "node": "rabbit@hostname"
}
```

### Преимущества JSON логов

1. **Легкость парсинга** — стандартный формат для log aggregation систем
2. **Structured data** — поля для фильтрации и поиска
3. **Интеграция** — работает с ELK, Loki, Splunk и другими

## Типы логируемых событий

### События соединений

```
# Новое соединение
accepting AMQP connection 172.17.0.1:52456 -> 172.17.0.2:5672

# Закрытие соединения
closing AMQP connection <0.1234.0> (172.17.0.1:52456 -> 172.17.0.2:5672, vhost: '/', user: 'admin')

# Ошибка аутентификации
Error on AMQP connection <0.1234.0>: authentication failure for user 'unknown'
```

### События очередей

```
# Создание очереди
Queue 'orders' in vhost '/' created

# Удаление очереди
Queue 'temp-queue' in vhost '/' deleted

# Синхронизация mirror
Mirrored queue 'ha-queue' in vhost '/': synchronising: 0 messages to go
```

### События кластера

```
# Присоединение к кластеру
Node rabbit@node2 joined cluster

# Отключение ноды
Node rabbit@node2 disconnected

# Network partition
Network partition detected: [rabbit@node1] and [rabbit@node2]
```

### Alarms

```
# Memory alarm
memory resource limit alarm set on node rabbit@hostname

# Disk alarm
disk resource limit alarm set on node rabbit@hostname

# Снятие alarm
memory resource limit alarm cleared on node rabbit@hostname
```

## Интеграция с системами логирования

### Syslog

```ini
# rabbitmq.conf
log.syslog = true
log.syslog.level = info
log.syslog.identity = rabbitmq
log.syslog.facility = local0

# Хост syslog сервера
log.syslog.ip = 192.168.1.100
log.syslog.port = 514
```

```bash
# rsyslog конфигурация (/etc/rsyslog.d/rabbitmq.conf)
local0.*    /var/log/rabbitmq/syslog.log
```

### ELK Stack (Elasticsearch, Logstash, Kibana)

```yaml
# Filebeat конфигурация (filebeat.yml)
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/rabbitmq/*.log
    json.keys_under_root: true
    json.add_error_key: true
    fields:
      service: rabbitmq
      environment: production

output.elasticsearch:
  hosts: ["elasticsearch:9200"]
  index: "rabbitmq-logs-%{+yyyy.MM.dd}"
```

```ruby
# Logstash конфигурация
input {
  beats {
    port => 5044
  }
}

filter {
  if [fields][service] == "rabbitmq" {
    json {
      source => "message"
    }
    date {
      match => ["time", "ISO8601"]
      target => "@timestamp"
    }
    mutate {
      remove_field => ["time"]
    }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "rabbitmq-logs-%{+yyyy.MM.dd}"
  }
}
```

### Grafana Loki

```yaml
# Promtail конфигурация (promtail.yml)
server:
  http_listen_port: 9080

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: rabbitmq
    static_configs:
      - targets:
          - localhost
        labels:
          job: rabbitmq
          __path__: /var/log/rabbitmq/*.log
    pipeline_stages:
      - json:
          expressions:
            level: level
            msg: msg
            node: node
      - labels:
          level:
          node:
```

### Fluentd

```xml
<!-- fluent.conf -->
<source>
  @type tail
  path /var/log/rabbitmq/*.log
  pos_file /var/log/td-agent/rabbitmq.pos
  tag rabbitmq.logs
  <parse>
    @type json
    time_key time
    time_format %Y-%m-%dT%H:%M:%S.%N%z
  </parse>
</source>

<match rabbitmq.logs>
  @type elasticsearch
  host elasticsearch
  port 9200
  index_name rabbitmq-logs
  <buffer>
    @type memory
    flush_interval 5s
  </buffer>
</match>
```

## Docker Logging

### Docker Compose

```yaml
version: '3.8'
services:
  rabbitmq:
    image: rabbitmq:3-management
    environment:
      - RABBITMQ_LOGS=-  # Логи в stdout
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

### Docker с Fluentd

```yaml
version: '3.8'
services:
  rabbitmq:
    image: rabbitmq:3-management
    logging:
      driver: fluentd
      options:
        fluentd-address: localhost:24224
        tag: rabbitmq.{{.Name}}
```

## Аудит событий безопасности

### Включение аудита

```ini
# rabbitmq.conf
# Логировать все попытки аутентификации
log.connection.level = info

# Включить event exchange для детального аудита
# rabbitmq-plugins enable rabbitmq_event_exchange
```

### События для аудита

```python
# Подписка на события аутентификации
import pika

connection = pika.BlockingConnection()
channel = connection.channel()

result = channel.queue_declare(queue='', exclusive=True)
queue_name = result.method.queue

# События аутентификации
channel.queue_bind(
    exchange='amq.rabbitmq.event',
    queue=queue_name,
    routing_key='user.authentication.*'
)

def callback(ch, method, properties, body):
    print(f"Auth event: {method.routing_key}")
    print(f"Details: {body}")

channel.basic_consume(
    queue=queue_name,
    on_message_callback=callback,
    auto_ack=True
)
channel.start_consuming()
```

## Ротация логов

### Встроенная ротация

```ini
# rabbitmq.conf
# По размеру
log.file.rotation.size = 10485760  # 10 MB

# По дате
log.file.rotation.date = $D0  # Ежедневно в полночь
# $D0 = ежедневно, $W0 = еженедельно, $M1 = ежемесячно

# Количество файлов
log.file.rotation.count = 10

# Сжатие
log.file.rotation.compress = true
```

### Logrotate (Linux)

```bash
# /etc/logrotate.d/rabbitmq
/var/log/rabbitmq/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
    dateext
    dateformat -%Y%m%d
}
```

### Ручная ротация

```bash
# Команда для ротации логов
rabbitmqctl rotate_logs

# Или через diagnostics
rabbitmq-diagnostics log_rotate
```

## Отладка и troubleshooting

### Временное включение debug

```bash
# Включить debug на 5 минут для диагностики
rabbitmqctl set_log_level debug

# Выполнить проблемную операцию
# ...

# Вернуть нормальный уровень
rabbitmqctl set_log_level info
```

### Фильтрация логов

```bash
# Поиск ошибок
grep -i error /var/log/rabbitmq/rabbit.log

# Поиск по соединению
grep "172.17.0.1" /var/log/rabbitmq/rabbit.log

# Поиск authentication failures
grep "authentication" /var/log/rabbitmq/rabbit.log | grep -i fail

# JSON логи с jq
cat /var/log/rabbitmq/rabbit.log | jq 'select(.level == "error")'

# Последние 100 ошибок
cat /var/log/rabbitmq/rabbit.log | jq 'select(.level == "error")' | tail -100
```

### Crash dumps

```bash
# Расположение crash dumps
ls /var/log/rabbitmq/erl_crash.dump

# Анализ crash dump
# Используйте Erlang crashdump viewer
erl -crashdump_viewer
```

## Best Practices

### Рекомендации по настройке

1. **Production**: используйте уровень `info` или `warning`
2. **Development**: можно использовать `debug` для детальной отладки
3. **Используйте JSON формат** для интеграции с log aggregation
4. **Настройте ротацию** для предотвращения переполнения диска
5. **Отправляйте логи в централизованную систему** (ELK, Loki)

### Что мониторить в логах

| Событие | Важность | Действие |
|---------|----------|----------|
| Memory/Disk alarm | Critical | Немедленное расследование |
| Authentication failure | Warning | Проверка безопасности |
| Network partition | Critical | Проверка сети |
| Connection refused | Warning | Проверка лимитов |
| Queue created/deleted | Info | Аудит |

### Алерты на основе логов

```yaml
# Пример AlertManager rule на основе логов в Loki
groups:
  - name: rabbitmq-logs
    rules:
      - alert: RabbitMQAuthenticationFailure
        expr: |
          count_over_time(
            {job="rabbitmq"} |~ "authentication.*fail" [5m]
          ) > 10
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Multiple authentication failures detected"
```

## Заключение

Правильно настроенное логирование в RabbitMQ обеспечивает:

- **Диагностику** — быстрое обнаружение и устранение проблем
- **Аудит** — отслеживание событий безопасности
- **Мониторинг** — интеграция с системами наблюдения
- **Compliance** — соответствие требованиям хранения логов

Используйте structured logging (JSON) и централизованное хранение для эффективного анализа логов в production-окружениях.

---

[prev: 03-monitoring](./03-monitoring.md) | [next: 05-metrics](./05-metrics.md)
