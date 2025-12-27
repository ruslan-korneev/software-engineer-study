# Безопасность ZooKeeper в Apache Kafka

## Описание

ZooKeeper играет критическую роль в архитектуре Apache Kafka (до версии 3.0+, где появился KRaft mode), храня метаданные кластера, конфигурации топиков, информацию о брокерах и ACLs. Компрометация ZooKeeper означает полную компрометацию Kafka кластера, поэтому его безопасность крайне важна.

Начиная с Kafka 3.0, ZooKeeper можно заменить на встроенный механизм консенсуса KRaft (Kafka Raft), который упрощает архитектуру и устраняет необходимость в отдельном ZooKeeper кластере. Однако многие production-кластеры всё ещё используют ZooKeeper, и понимание его безопасности остаётся актуальным.

## Ключевые концепции

### Роль ZooKeeper в Kafka

```
┌─────────────────────────────────────────────────────────────────────┐
│                     ZOOKEEPER В KAFKA                               │
└─────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│                        ZOOKEEPER CLUSTER                             │
│  ┌────────────┐    ┌────────────┐    ┌────────────┐                 │
│  │  ZK Node 1 │◄──►│  ZK Node 2 │◄──►│  ZK Node 3 │                 │
│  │  (Leader)  │    │ (Follower) │    │ (Follower) │                 │
│  └────────────┘    └────────────┘    └────────────┘                 │
│                                                                      │
│  Хранимые данные:                                                    │
│  ├── /brokers/ids/{broker_id}     - информация о брокерах           │
│  ├── /brokers/topics/{topic}      - метаданные топиков               │
│  ├── /controller                  - текущий controller               │
│  ├── /config/topics/{topic}       - конфигурация топиков            │
│  ├── /config/users/{user}         - SCRAM credentials               │
│  ├── /kafka-acl/Topic/{topic}     - ACLs для топиков                │
│  └── /consumers/{group}           - (legacy) consumer offsets        │
└──────────────────────────────────────────────────────────────────────┘
                            │
                            │ Подключение брокеров
                            │ (SASL/Digest-MD5 или mTLS)
                            ▼
┌──────────────────────────────────────────────────────────────────────┐
│                        KAFKA BROKERS                                 │
│  ┌────────────┐    ┌────────────┐    ┌────────────┐                 │
│  │  Broker 1  │    │  Broker 2  │    │  Broker 3  │                 │
│  └────────────┘    └────────────┘    └────────────┘                 │
└──────────────────────────────────────────────────────────────────────┘
```

### Угрозы безопасности ZooKeeper

| Угроза | Описание | Влияние |
|--------|----------|---------|
| Несанкционированный доступ | Прямое подключение к ZK без аутентификации | Чтение/модификация всех метаданных |
| Перехват трафика | Прослушивание незашифрованного трафика | Утечка credentials и конфигураций |
| Man-in-the-Middle | Подмена ZK узла | Перенаправление брокеров, инъекция данных |
| DoS атаки | Перегрузка ZK запросами | Недоступность Kafka кластера |
| Privilege escalation | Изменение ACLs в ZK | Получение доступа к защищённым топикам |

### Механизмы защиты

```
┌─────────────────────────────────────────────────────────────────────┐
│                   УРОВНИ ЗАЩИТЫ ZOOKEEPER                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. СЕТЕВОЙ УРОВЕНЬ                                                 │
│     ├── Firewall: только разрешённые IP                             │
│     ├── Изолированная сеть (VPC/VLAN)                               │
│     └── Отсутствие публичного доступа                               │
│                                                                     │
│  2. ТРАНСПОРТНЫЙ УРОВЕНЬ                                            │
│     ├── TLS шифрование (ZK 3.5+)                                    │
│     └── mTLS для взаимной аутентификации                            │
│                                                                     │
│  3. УРОВЕНЬ АУТЕНТИФИКАЦИИ                                          │
│     ├── SASL/Digest-MD5 (рекомендуется)                             │
│     ├── SASL/Kerberos (enterprise)                                  │
│     └── Client Certificate (mTLS)                                   │
│                                                                     │
│  4. УРОВЕНЬ АВТОРИЗАЦИИ                                             │
│     ├── ZooKeeper ACLs (znode permissions)                          │
│     └── Super user для администрирования                            │
│                                                                     │
│  5. МОНИТОРИНГ И АУДИТ                                              │
│     ├── Логирование подключений                                      │
│     ├── Мониторинг аномалий                                         │
│     └── Алерты на подозрительную активность                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## Примеры конфигурации

### Настройка SASL аутентификации между Kafka и ZooKeeper

**zoo.cfg** (конфигурация ZooKeeper):

```properties
# Основные настройки
dataDir=/var/lib/zookeeper
clientPort=2181
tickTime=2000
initLimit=5
syncLimit=2

# Кластерная конфигурация
server.1=zk1.example.com:2888:3888
server.2=zk2.example.com:2888:3888
server.3=zk3.example.com:2888:3888

# Включение SASL аутентификации
authProvider.sasl=org.apache.zookeeper.server.auth.SASLAuthenticationProvider
requireClientAuthScheme=sasl

# Для Kerberos
# authProvider.1=org.apache.zookeeper.server.auth.SASLAuthenticationProvider
# kerberos.removeHostFromPrincipal=true
# kerberos.removeRealmFromPrincipal=true

# Отключение анонимного доступа
skipACL=false
```

**zookeeper_server_jaas.conf** (JAAS конфигурация для ZooKeeper):

```
Server {
    org.apache.zookeeper.server.auth.DigestLoginModule required
    user_admin="admin-secret"
    user_kafka="kafka-secret";
};
```

**Запуск ZooKeeper с SASL:**

```bash
export SERVER_JVMFLAGS="-Djava.security.auth.login.config=/etc/zookeeper/zookeeper_server_jaas.conf"
zkServer.sh start
```

### Настройка Kafka для SASL подключения к ZooKeeper

**server.properties** (конфигурация Kafka брокера):

```properties
# ZooKeeper connection string
zookeeper.connect=zk1.example.com:2181,zk2.example.com:2181,zk3.example.com:2181

# Таймауты
zookeeper.connection.timeout.ms=30000
zookeeper.session.timeout.ms=18000

# SASL для ZooKeeper (устаревший способ, но всё ещё используется)
zookeeper.set.acl=true
```

**kafka_server_jaas.conf** (JAAS конфигурация для Kafka):

```
# Аутентификация Kafka -> ZooKeeper
Client {
    org.apache.zookeeper.server.auth.DigestLoginModule required
    username="kafka"
    password="kafka-secret";
};

# Аутентификация клиентов -> Kafka (SCRAM)
KafkaServer {
    org.apache.kafka.common.security.scram.ScramLoginModule required
    username="admin"
    password="admin-secret";
};
```

**Запуск Kafka с SASL для ZooKeeper:**

```bash
export KAFKA_OPTS="-Djava.security.auth.login.config=/etc/kafka/kafka_server_jaas.conf"
kafka-server-start.sh /etc/kafka/server.properties
```

### Настройка TLS для ZooKeeper (версия 3.5+)

**zoo.cfg** с TLS:

```properties
# Порт для TLS
secureClientPort=2182

# SSL настройки
ssl.keyStore.location=/var/ssl/zookeeper.keystore.jks
ssl.keyStore.password=keystore-password
ssl.trustStore.location=/var/ssl/zookeeper.truststore.jks
ssl.trustStore.password=truststore-password

# Требовать клиентскую аутентификацию
ssl.clientAuth=need

# Отключение plaintext порта
# clientPort=2181  # Закомментировать для только TLS

# Протоколы
ssl.protocol=TLSv1.2
```

**Конфигурация Kafka для TLS подключения к ZooKeeper:**

```properties
# server.properties
zookeeper.connect=zk1.example.com:2182,zk2.example.com:2182,zk3.example.com:2182

# SSL настройки для ZooKeeper
zookeeper.ssl.client.enable=true
zookeeper.clientCnxnSocket=org.apache.zookeeper.ClientCnxnSocketNetty
zookeeper.ssl.keystore.location=/var/ssl/kafka.keystore.jks
zookeeper.ssl.keystore.password=keystore-password
zookeeper.ssl.truststore.location=/var/ssl/kafka.truststore.jks
zookeeper.ssl.truststore.password=truststore-password
```

### Kerberos аутентификация для ZooKeeper

**zookeeper_server_jaas.conf** (Kerberos):

```
Server {
    com.sun.security.auth.module.Krb5LoginModule required
    useKeyTab=true
    keyTab="/etc/security/keytabs/zookeeper.service.keytab"
    storeKey=true
    useTicketCache=false
    principal="zookeeper/zk1.example.com@EXAMPLE.COM";
};
```

**kafka_server_jaas.conf** (Kafka с Kerberos для ZooKeeper):

```
Client {
    com.sun.security.auth.module.Krb5LoginModule required
    useKeyTab=true
    keyTab="/etc/security/keytabs/kafka.service.keytab"
    storeKey=true
    useTicketCache=false
    principal="kafka/broker1.example.com@EXAMPLE.COM";
};
```

### ZooKeeper ACLs

ZooKeeper имеет собственную систему ACLs для защиты znodes:

```bash
# Подключение к ZooKeeper CLI с аутентификацией
zkCli.sh -server localhost:2181

# Аутентификация
addauth digest admin:admin-secret

# Создание znode с ACL
create /secure-data "secret" digest:admin:encrypted-password:cdrwa

# Просмотр ACL
getAcl /secure-data

# Установка ACL на существующий znode
setAcl /brokers/topics digest:kafka:encrypted-password:cdrwa
```

**Права доступа ZooKeeper ACL:**

| Право | Описание |
|-------|----------|
| c (CREATE) | Создание дочерних znodes |
| d (DELETE) | Удаление дочерних znodes |
| r (READ) | Чтение данных и списка дочерних |
| w (WRITE) | Запись данных в znode |
| a (ADMIN) | Изменение ACL |

### Автоматическая установка ACLs при старте Kafka

```properties
# server.properties

# Включить установку ACLs на znodes, создаваемые Kafka
zookeeper.set.acl=true
```

При `zookeeper.set.acl=true` Kafka автоматически устанавливает ACLs на создаваемые znodes, ограничивая доступ только аутентифицированным брокерам.

## Миграция на KRaft (без ZooKeeper)

Начиная с Kafka 3.3, KRaft (Kafka Raft) готов для production:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    KRAFT vs ZOOKEEPER                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ZOOKEEPER MODE                      KRAFT MODE                     │
│  ──────────────                      ──────────                     │
│                                                                     │
│  ┌──────────────┐                    ┌──────────────┐               │
│  │  ZooKeeper   │                    │   Kafka      │               │
│  │   Cluster    │                    │  Controller  │               │
│  │  (отдельный) │                    │ (встроенный) │               │
│  └──────┬───────┘                    └──────┬───────┘               │
│         │                                   │                       │
│         ▼                                   ▼                       │
│  ┌──────────────┐                    ┌──────────────┐               │
│  │    Kafka     │                    │    Kafka     │               │
│  │   Brokers    │                    │   Brokers    │               │
│  └──────────────┘                    │ + Controller │               │
│                                      └──────────────┘               │
│                                                                     │
│  Преимущества KRaft:                                                │
│  ✓ Меньше компонентов для управления                                │
│  ✓ Быстрее переключение controller                                  │
│  ✓ Более простая конфигурация безопасности                          │
│  ✓ Лучшая масштабируемость метаданных                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Конфигурация KRaft

```properties
# server.properties для KRaft mode

# Режим процесса
process.roles=broker,controller

# Node ID (уникальный для каждого узла)
node.id=1

# Controller кворум
controller.quorum.voters=1@controller1:9093,2@controller2:9093,3@controller3:9093

# Listeners
listeners=PLAINTEXT://:9092,CONTROLLER://:9093
inter.broker.listener.name=PLAINTEXT
controller.listener.names=CONTROLLER

# Директории
log.dirs=/var/kafka-logs
metadata.log.dir=/var/kafka-metadata

# Безопасность controller
# (Аналогично настройке брокеров - SASL_SSL)
listener.name.controller.sasl.enabled.mechanisms=SCRAM-SHA-512
listener.name.controller.scram-sha-512.sasl.jaas.config=...
```

## Best Practices

### 1. Изоляция сети ZooKeeper

```yaml
# Docker Compose пример изоляции
version: '3.8'

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    networks:
      - zk-internal    # Только внутренняя сеть
    # Нет port mapping наружу!

  kafka:
    image: confluentinc/cp-kafka:7.4.0
    networks:
      - zk-internal    # Доступ к ZK
      - kafka-clients  # Доступ для клиентов
    ports:
      - "9094:9094"    # Только защищённый порт наружу

networks:
  zk-internal:
    internal: true     # Изолированная сеть
  kafka-clients:
    driver: bridge
```

### 2. Firewall правила

```bash
# iptables правила для ZooKeeper

# Разрешить только Kafka брокерам
iptables -A INPUT -p tcp --dport 2181 -s 10.0.1.0/24 -j ACCEPT
iptables -A INPUT -p tcp --dport 2181 -j DROP

# Разрешить ZK-to-ZK трафик (кластерные порты)
iptables -A INPUT -p tcp --dport 2888 -s 10.0.0.0/24 -j ACCEPT
iptables -A INPUT -p tcp --dport 3888 -s 10.0.0.0/24 -j ACCEPT

# Логирование отклонённых подключений
iptables -A INPUT -p tcp --dport 2181 -j LOG --log-prefix "ZK-DENIED: "
```

### 3. Мониторинг ZooKeeper

```yaml
# Prometheus AlertManager правила
groups:
  - name: zookeeper-security
    rules:
      - alert: ZookeeperUnauthorizedConnection
        expr: zk_connection_rejected_total > 0
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Unauthorized ZooKeeper connection attempt"

      - alert: ZookeeperHighLatency
        expr: zk_avg_latency > 100
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High ZooKeeper latency - possible DoS"

      - alert: ZookeeperTooManyConnections
        expr: zk_num_alive_connections > 100
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Too many ZooKeeper connections"
```

```bash
# ZooKeeper 4-letter commands для мониторинга
# (Должны быть ограничены в production)

# Статус
echo "stat" | nc localhost 2181

# Конфигурация
echo "conf" | nc localhost 2181

# Список подключений
echo "cons" | nc localhost 2181

# Ограничение 4-letter commands в zoo.cfg
# 4lw.commands.whitelist=stat,ruok,conf
```

### 4. Аудит логирование

```properties
# log4j.properties для ZooKeeper

# Аудит логирование
log4j.logger.org.apache.zookeeper.server.NIOServerCnxnFactory=INFO
log4j.logger.org.apache.zookeeper.server.NIOServerCnxn=INFO
log4j.logger.org.apache.zookeeper.server.auth=DEBUG

# Формат с timestamp и IP
log4j.appender.AUDIT=org.apache.log4j.RollingFileAppender
log4j.appender.AUDIT.File=/var/log/zookeeper/audit.log
log4j.appender.AUDIT.MaxFileSize=100MB
log4j.appender.AUDIT.MaxBackupIndex=10
log4j.appender.AUDIT.layout=org.apache.log4j.PatternLayout
log4j.appender.AUDIT.layout.ConversionPattern=%d{ISO8601} [%t] %-5p %c{2} - %m%n
```

### 5. Регулярная ротация credentials

```bash
#!/bin/bash
# rotate-zk-credentials.sh

NEW_KAFKA_PASSWORD=$(openssl rand -base64 32)

# 1. Обновить JAAS конфигурацию ZooKeeper
# Добавить нового пользователя, сохраняя старого
cat > /etc/zookeeper/zookeeper_server_jaas.conf << EOF
Server {
    org.apache.zookeeper.server.auth.DigestLoginModule required
    user_admin="admin-secret"
    user_kafka="$NEW_KAFKA_PASSWORD"
    user_kafka_old="old-kafka-password";  # Временно для graceful rotation
};
EOF

# 2. Перезагрузить ZooKeeper (rolling restart)
systemctl reload zookeeper

# 3. Обновить JAAS конфигурацию Kafka брокеров
# 4. Rolling restart Kafka брокеров

# 5. Удалить старый пароль из ZooKeeper конфигурации
```

### 6. Чеклист безопасности ZooKeeper

```markdown
## Checklist

### Сеть
- [ ] ZooKeeper не доступен из интернета
- [ ] Firewall ограничивает доступ только Kafka брокерами
- [ ] ZK-to-ZK трафик в изолированной сети
- [ ] 4-letter commands отключены или ограничены

### Аутентификация
- [ ] SASL/Digest-MD5 или Kerberos включён
- [ ] Уникальные credentials для каждого брокера (опционально)
- [ ] TLS включён (ZK 3.5+)
- [ ] Сертификаты от доверенного CA

### Авторизация
- [ ] zookeeper.set.acl=true на брокерах
- [ ] ACLs установлены на критичные znodes
- [ ] Минимальные права для брокеров

### Мониторинг
- [ ] Логирование подключений
- [ ] Алерты на аномальную активность
- [ ] Метрики собираются в Prometheus

### Операционная безопасность
- [ ] Регулярная ротация credentials
- [ ] Резервное копирование ZK данных
- [ ] Документированные процедуры восстановления
```

## Troubleshooting

```bash
# Проверка SASL аутентификации
# Включить отладку
export CLIENT_JVMFLAGS="-Djava.security.auth.login.config=/path/to/jaas.conf \
    -Dzookeeper.clientCnxnSocket=org.apache.zookeeper.ClientCnxnSocketNIO \
    -Djava.security.debug=all"

zkCli.sh -server localhost:2181

# Типичные ошибки
# KeeperErrorCode = AuthFailed - неверные credentials
# KeeperErrorCode = NoAuth - отсутствует аутентификация

# Проверка TLS
openssl s_client -connect zk1.example.com:2182 -showcerts

# Проверка ACLs
zkCli.sh -server localhost:2181
getAcl /brokers

# Логи ZooKeeper
tail -f /var/log/zookeeper/zookeeper.log | grep -i "auth\|sasl\|ssl"
```

## Дополнительные ресурсы

- [ZooKeeper Administrator's Guide](https://zookeeper.apache.org/doc/current/zookeeperAdmin.html)
- [ZooKeeper Security](https://zookeeper.apache.org/doc/current/zookeeperProgrammers.html#sc_ZooKeeperAccessControl)
- [Kafka ZooKeeper Security](https://kafka.apache.org/documentation/#zk_authz)
- [KRaft Documentation](https://kafka.apache.org/documentation/#kraft)
