# Запуск Kafka как службы systemd

## Описание

Systemd — это система инициализации и управления службами в современных Linux-дистрибутивах. Настройка Kafka как службы systemd позволяет автоматически запускать брокеры при загрузке системы, управлять жизненным циклом процессов, настраивать зависимости между службами и обеспечивать автоматический перезапуск при сбоях.

Правильная настройка systemd unit-файлов критически важна для production-окружений, обеспечивая надёжную работу кластера Kafka.

## Ключевые концепции

### Структура systemd unit-файла

Unit-файл состоит из нескольких секций:

| Секция | Назначение |
|--------|------------|
| `[Unit]` | Метаданные и зависимости службы |
| `[Service]` | Параметры запуска и работы службы |
| `[Install]` | Параметры установки и автозапуска |

### Основные параметры

| Параметр | Описание |
|----------|----------|
| `Type` | Тип процесса (simple, forking, notify) |
| `ExecStart` | Команда запуска |
| `ExecStop` | Команда остановки |
| `Restart` | Политика перезапуска |
| `User/Group` | Пользователь и группа |
| `Environment` | Переменные окружения |
| `LimitNOFILE` | Лимит открытых файлов |

## Примеры

### Базовый unit-файл для Kafka

```ini
# /etc/systemd/system/kafka.service

[Unit]
Description=Apache Kafka Message Broker
Documentation=https://kafka.apache.org/documentation/
Requires=network.target
After=network.target

[Service]
Type=simple
User=kafka
Group=kafka

# Переменные окружения
Environment="KAFKA_HOME=/opt/kafka"
Environment="KAFKA_HEAP_OPTS=-Xmx4G -Xms4G"
Environment="KAFKA_JVM_PERFORMANCE_OPTS=-server -XX:+UseG1GC -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35 -XX:+ExplicitGCInvokesConcurrent -XX:MaxInlineLevel=15 -Djava.awt.headless=true"
Environment="KAFKA_LOG4J_OPTS=-Dlog4j.configuration=file:/opt/kafka/config/log4j.properties"

# Запуск и остановка
ExecStart=/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/server.properties
ExecStop=/opt/kafka/bin/kafka-server-stop.sh

# Лимиты ресурсов
LimitNOFILE=100000
LimitNPROC=32000

# Политика перезапуска
Restart=on-failure
RestartSec=10
TimeoutStopSec=60

# Рабочая директория
WorkingDirectory=/opt/kafka

[Install]
WantedBy=multi-user.target
```

### Unit-файл для ZooKeeper

```ini
# /etc/systemd/system/zookeeper.service

[Unit]
Description=Apache ZooKeeper Server
Documentation=https://zookeeper.apache.org/doc/current/
Requires=network.target
After=network.target

[Service]
Type=simple
User=kafka
Group=kafka

Environment="ZOOCFGDIR=/opt/kafka/config"
Environment="ZOO_LOG_DIR=/var/log/zookeeper"
Environment="JAVA_HOME=/usr/lib/jvm/java-17-openjdk"

ExecStart=/opt/kafka/bin/zookeeper-server-start.sh /opt/kafka/config/zookeeper.properties
ExecStop=/opt/kafka/bin/zookeeper-server-stop.sh

LimitNOFILE=65536
Restart=on-failure
RestartSec=5
TimeoutStopSec=30

[Install]
WantedBy=multi-user.target
```

### Unit-файл для Kafka с зависимостью от ZooKeeper

```ini
# /etc/systemd/system/kafka.service

[Unit]
Description=Apache Kafka Message Broker
Documentation=https://kafka.apache.org/documentation/
Requires=zookeeper.service network.target
After=zookeeper.service network.target

[Service]
Type=simple
User=kafka
Group=kafka

Environment="KAFKA_HOME=/opt/kafka"
Environment="JAVA_HOME=/usr/lib/jvm/java-17-openjdk"
Environment="KAFKA_HEAP_OPTS=-Xmx6G -Xms6G"
Environment="KAFKA_JVM_PERFORMANCE_OPTS=-server -XX:+UseG1GC -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35 -XX:+ExplicitGCInvokesConcurrent -Djava.awt.headless=true"

ExecStart=/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/server.properties
ExecStop=/opt/kafka/bin/kafka-server-stop.sh

LimitNOFILE=100000
LimitNPROC=32000

Restart=on-failure
RestartSec=15
TimeoutStartSec=180
TimeoutStopSec=120

# Ожидание готовности ZooKeeper
ExecStartPre=/bin/sleep 5

WorkingDirectory=/opt/kafka

[Install]
WantedBy=multi-user.target
```

### Unit-файл для KRaft mode (без ZooKeeper)

```ini
# /etc/systemd/system/kafka-kraft.service

[Unit]
Description=Apache Kafka in KRaft Mode
Documentation=https://kafka.apache.org/documentation/
Requires=network.target
After=network.target

[Service]
Type=simple
User=kafka
Group=kafka

Environment="KAFKA_HOME=/opt/kafka"
Environment="JAVA_HOME=/usr/lib/jvm/java-17-openjdk"
Environment="KAFKA_HEAP_OPTS=-Xmx6G -Xms6G"
Environment="KAFKA_JVM_PERFORMANCE_OPTS=-server -XX:+UseG1GC -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35"
Environment="KAFKA_CLUSTER_ID=MkU3OEVBNTcwNTJENDM2Qk"

# Форматирование хранилища (выполняется только один раз)
# ExecStartPre=/opt/kafka/bin/kafka-storage.sh format -t $KAFKA_CLUSTER_ID -c /opt/kafka/config/kraft/server.properties

ExecStart=/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/kraft/server.properties
ExecStop=/opt/kafka/bin/kafka-server-stop.sh

LimitNOFILE=100000
LimitNPROC=32000

Restart=on-failure
RestartSec=10
TimeoutStopSec=120

WorkingDirectory=/opt/kafka

[Install]
WantedBy=multi-user.target
```

### Продвинутый unit-файл с расширенными настройками

```ini
# /etc/systemd/system/kafka.service

[Unit]
Description=Apache Kafka Message Broker
Documentation=https://kafka.apache.org/documentation/
Requires=network-online.target
After=network-online.target zookeeper.service
Wants=zookeeper.service

# Условия запуска
ConditionPathExists=/opt/kafka/config/server.properties
ConditionPathExists=/opt/kafka/bin/kafka-server-start.sh

[Service]
Type=simple
User=kafka
Group=kafka

# Файл окружения
EnvironmentFile=-/etc/kafka/kafka.env

# JVM настройки через отдельный файл
ExecStart=/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/server.properties

# Graceful shutdown
ExecStop=/opt/kafka/bin/kafka-server-stop.sh
KillMode=mixed
KillSignal=SIGTERM

# Лимиты ресурсов
LimitNOFILE=100000
LimitNPROC=32000
LimitCORE=infinity
LimitMEMLOCK=infinity

# Политика перезапуска
Restart=on-failure
RestartSec=30
RestartPreventExitStatus=255

# Таймауты
TimeoutStartSec=180
TimeoutStopSec=120
WatchdogSec=0

# Безопасность
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/log/kafka /var/lib/kafka

# Логирование
StandardOutput=journal
StandardError=journal
SyslogIdentifier=kafka

[Install]
WantedBy=multi-user.target
```

### Файл окружения

```bash
# /etc/kafka/kafka.env

KAFKA_HOME=/opt/kafka
JAVA_HOME=/usr/lib/jvm/java-17-openjdk

# JVM настройки
KAFKA_HEAP_OPTS="-Xmx8G -Xms8G"
KAFKA_JVM_PERFORMANCE_OPTS="-server -XX:+UseG1GC -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35 -XX:+ExplicitGCInvokesConcurrent -XX:MaxInlineLevel=15 -Djava.awt.headless=true -XX:+ParallelRefProcEnabled -XX:+DisableExplicitGC"

# GC логирование
KAFKA_GC_LOG_OPTS="-Xlog:gc*:file=/var/log/kafka/gc.log:time,tags:filecount=10,filesize=100M"

# JMX настройки для мониторинга
KAFKA_JMX_OPTS="-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.port=9999 -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false"

# Log4j
KAFKA_LOG4J_OPTS="-Dlog4j.configuration=file:/opt/kafka/config/log4j.properties"
```

### Команды управления службой

```bash
# Перезагрузка конфигурации systemd
sudo systemctl daemon-reload

# Включение автозапуска
sudo systemctl enable zookeeper.service
sudo systemctl enable kafka.service

# Запуск служб
sudo systemctl start zookeeper.service
sudo systemctl start kafka.service

# Остановка служб
sudo systemctl stop kafka.service
sudo systemctl stop zookeeper.service

# Перезапуск
sudo systemctl restart kafka.service

# Проверка статуса
sudo systemctl status kafka.service

# Просмотр логов
sudo journalctl -u kafka.service -f
sudo journalctl -u kafka.service --since "1 hour ago"
sudo journalctl -u kafka.service -n 100

# Просмотр логов с определённого времени
sudo journalctl -u kafka.service --since "2024-01-15 10:00:00"

# Анализ ошибок
sudo journalctl -u kafka.service -p err
```

### Скрипт подготовки системы

```bash
#!/bin/bash
# setup-kafka-systemd.sh

set -e

# Создание пользователя и группы
sudo groupadd -r kafka 2>/dev/null || true
sudo useradd -r -g kafka -d /opt/kafka -s /sbin/nologin kafka 2>/dev/null || true

# Создание директорий
sudo mkdir -p /var/log/kafka
sudo mkdir -p /var/lib/kafka
sudo mkdir -p /opt/kafka
sudo mkdir -p /etc/kafka

# Установка прав
sudo chown -R kafka:kafka /var/log/kafka
sudo chown -R kafka:kafka /var/lib/kafka
sudo chown -R kafka:kafka /opt/kafka
sudo chown -R kafka:kafka /etc/kafka

# Настройка лимитов
cat << 'EOF' | sudo tee /etc/security/limits.d/kafka.conf
kafka soft nofile 100000
kafka hard nofile 100000
kafka soft nproc 32000
kafka hard nproc 32000
EOF

# Копирование unit-файлов
sudo cp kafka.service /etc/systemd/system/
sudo cp zookeeper.service /etc/systemd/system/

# Перезагрузка systemd
sudo systemctl daemon-reload

# Включение служб
sudo systemctl enable zookeeper.service
sudo systemctl enable kafka.service

echo "Kafka systemd services configured successfully"
```

### Unit-файл для Kafka Connect

```ini
# /etc/systemd/system/kafka-connect.service

[Unit]
Description=Apache Kafka Connect
Documentation=https://kafka.apache.org/documentation/
Requires=kafka.service
After=kafka.service

[Service]
Type=simple
User=kafka
Group=kafka

Environment="KAFKA_HOME=/opt/kafka"
Environment="KAFKA_HEAP_OPTS=-Xmx2G -Xms2G"
Environment="KAFKA_LOG4J_OPTS=-Dlog4j.configuration=file:/opt/kafka/config/connect-log4j.properties"

ExecStart=/opt/kafka/bin/connect-distributed.sh /opt/kafka/config/connect-distributed.properties
ExecStop=/bin/kill -TERM $MAINPID

LimitNOFILE=65536
Restart=on-failure
RestartSec=10
TimeoutStopSec=60

[Install]
WantedBy=multi-user.target
```

## Best Practices

### Рекомендации по настройке

1. **Используйте отдельного пользователя** для запуска Kafka
2. **Настройте правильные лимиты** nofile и nproc
3. **Установите зависимости** между службами (Kafka зависит от ZooKeeper)
4. **Используйте файлы окружения** для JVM-настроек
5. **Настройте таймауты** с учётом времени graceful shutdown
6. **Включите мониторинг** через JMX

### Проверка здоровья службы

```ini
# Добавьте в [Service] секцию
ExecStartPost=/opt/kafka/bin/kafka-broker-api-versions.sh --bootstrap-server localhost:9092
```

### Безопасность

```ini
# Параметры безопасности в [Service]
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/log/kafka /var/lib/kafka /opt/kafka/logs
CapabilityBoundingSet=
```

### Мониторинг и алерты

```bash
# Проверка состояния службы
systemctl is-active kafka.service
systemctl is-failed kafka.service

# Интеграция с мониторингом
if ! systemctl is-active --quiet kafka.service; then
    echo "CRITICAL: Kafka service is not running"
    exit 2
fi
```

### Автоматический перезапуск при сбоях

```ini
# Политика перезапуска
Restart=on-failure
RestartSec=30

# Максимальное количество перезапусков
StartLimitIntervalSec=300
StartLimitBurst=5
```

### Логирование в journald

```bash
# Настройка ротации логов journald
# /etc/systemd/journald.conf
[Journal]
SystemMaxUse=2G
SystemKeepFree=1G
MaxRetentionSec=1month
```
