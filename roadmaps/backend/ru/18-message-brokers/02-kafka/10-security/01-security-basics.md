# Основы безопасности Kafka

## Описание

Безопасность Apache Kafka охватывает комплекс мер по защите данных, передаваемых через кластер, от несанкционированного доступа, перехвата и модификации. Kafka изначально разрабатывалась без встроенных механизмов безопасности, но начиная с версии 0.9 были добавлены полноценные возможности аутентификации, авторизации и шифрования.

Модель безопасности Kafka основана на трёх ключевых компонентах:
- **Аутентификация (Authentication)** — проверка личности клиентов и брокеров
- **Авторизация (Authorization)** — контроль доступа к ресурсам
- **Шифрование (Encryption)** — защита данных при передаче и хранении

## Ключевые концепции

### Уровни защиты

```
┌─────────────────────────────────────────────────────────────┐
│                    УРОВНИ БЕЗОПАСНОСТИ                      │
├─────────────────────────────────────────────────────────────┤
│  1. Сетевой уровень                                         │
│     - Firewall / Security Groups                            │
│     - VPN / Private Networks                                │
│     - SSL/TLS шифрование                                    │
├─────────────────────────────────────────────────────────────┤
│  2. Уровень аутентификации                                  │
│     - SASL (PLAIN, SCRAM, GSSAPI, OAUTHBEARER)             │
│     - mTLS (взаимная аутентификация)                        │
├─────────────────────────────────────────────────────────────┤
│  3. Уровень авторизации                                     │
│     - ACLs (Access Control Lists)                           │
│     - RBAC (Role-Based Access Control) - Confluent          │
├─────────────────────────────────────────────────────────────┤
│  4. Уровень данных                                          │
│     - Шифрование данных в покое (at rest)                   │
│     - Маскирование данных                                   │
│     - Аудит логирование                                     │
└─────────────────────────────────────────────────────────────┘
```

### Протоколы безопасности (Security Protocols)

Kafka поддерживает четыре протокола безопасности для listeners:

| Протокол | Шифрование | Аутентификация | Описание |
|----------|------------|----------------|----------|
| PLAINTEXT | Нет | Нет | Без защиты, только для разработки |
| SSL | Да (TLS) | Опционально (mTLS) | Шифрование трафика |
| SASL_PLAINTEXT | Нет | SASL | Аутентификация без шифрования |
| SASL_SSL | Да (TLS) | SASL | Полная защита |

### Архитектура безопасности

```
                    ┌──────────────────────┐
                    │     КЛИЕНТ           │
                    │  (Producer/Consumer) │
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
                    │   АУТЕНТИФИКАЦИЯ     │
                    │   (SSL/SASL)         │
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
                    │    АВТОРИЗАЦИЯ       │
                    │      (ACLs)          │
                    └──────────┬───────────┘
                               │
        ┌──────────────────────┼──────────────────────┐
        │                      │                      │
┌───────▼───────┐    ┌─────────▼────────┐    ┌───────▼───────┐
│   BROKER 1    │    │    BROKER 2      │    │   BROKER 3    │
│   (Leader)    │◄──►│   (Follower)     │◄──►│  (Follower)   │
└───────┬───────┘    └──────────────────┘    └───────────────┘
        │
        │ Репликация (SSL между брокерами)
        │
┌───────▼───────────────────────────────────────────────────┐
│                    ZOOKEEPER                               │
│              (SASL/Digest-MD5)                             │
└───────────────────────────────────────────────────────────┘
```

## Примеры конфигурации

### Базовая конфигурация безопасности брокера

```properties
# server.properties

# Определение listeners с разными протоколами безопасности
listeners=PLAINTEXT://localhost:9092,SSL://localhost:9093,SASL_SSL://localhost:9094

# Advertised listeners для клиентов
advertised.listeners=PLAINTEXT://kafka.example.com:9092,SSL://kafka.example.com:9093,SASL_SSL://kafka.example.com:9094

# Протокол для межброкерного взаимодействия
inter.broker.listener.name=SSL
security.inter.broker.protocol=SSL

# Включение авторизации
authorizer.class.name=kafka.security.authorizer.AclAuthorizer

# Суперпользователи (обходят ACL проверки)
super.users=User:admin;User:kafka

# Разрешить операции если ACL не найден (по умолчанию false)
allow.everyone.if.no.acl.found=false
```

### Конфигурация SSL/TLS

```properties
# SSL для клиентских подключений
ssl.keystore.location=/var/private/ssl/kafka.server.keystore.jks
ssl.keystore.password=keystore-password
ssl.key.password=key-password
ssl.truststore.location=/var/private/ssl/kafka.server.truststore.jks
ssl.truststore.password=truststore-password

# Требовать клиентскую аутентификацию (mTLS)
ssl.client.auth=required

# Допустимые протоколы и шифры
ssl.enabled.protocols=TLSv1.2,TLSv1.3
ssl.protocol=TLSv1.3
ssl.cipher.suites=TLS_AES_256_GCM_SHA384,TLS_CHACHA20_POLY1305_SHA256
```

### Конфигурация SASL

```properties
# SASL механизмы
sasl.enabled.mechanisms=SCRAM-SHA-512,PLAIN
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-512

# JAAS конфигурация для брокера
listener.name.sasl_ssl.scram-sha-512.sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
    username="kafka" \
    password="kafka-secret";
```

### Пример клиентской конфигурации (Python)

```python
from kafka import KafkaProducer, KafkaConsumer

# Конфигурация с SASL_SSL
config = {
    'bootstrap_servers': ['kafka.example.com:9094'],
    'security_protocol': 'SASL_SSL',
    'sasl_mechanism': 'SCRAM-SHA-512',
    'sasl_plain_username': 'myuser',
    'sasl_plain_password': 'mypassword',
    'ssl_cafile': '/path/to/ca-cert.pem',
    'ssl_certfile': '/path/to/client-cert.pem',
    'ssl_keyfile': '/path/to/client-key.pem',
    'ssl_check_hostname': True
}

# Producer с безопасным подключением
producer = KafkaProducer(**config)

# Consumer с безопасным подключением
consumer = KafkaConsumer(
    'my-topic',
    group_id='my-group',
    **config
)
```

### Пример конфигурации (Java)

```java
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.common.config.SaslConfigs;
import org.apache.kafka.common.config.SslConfigs;

import java.util.Properties;

public class SecureKafkaConfig {

    public static Properties getSecureConfig() {
        Properties props = new Properties();

        // Bootstrap servers
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka.example.com:9094");

        // Security protocol
        props.put("security.protocol", "SASL_SSL");

        // SASL configuration
        props.put(SaslConfigs.SASL_MECHANISM, "SCRAM-SHA-512");
        props.put(SaslConfigs.SASL_JAAS_CONFIG,
            "org.apache.kafka.common.security.scram.ScramLoginModule required " +
            "username=\"myuser\" password=\"mypassword\";");

        // SSL configuration
        props.put(SslConfigs.SSL_TRUSTSTORE_LOCATION_CONFIG, "/path/to/truststore.jks");
        props.put(SslConfigs.SSL_TRUSTSTORE_PASSWORD_CONFIG, "truststore-password");
        props.put(SslConfigs.SSL_KEYSTORE_LOCATION_CONFIG, "/path/to/keystore.jks");
        props.put(SslConfigs.SSL_KEYSTORE_PASSWORD_CONFIG, "keystore-password");
        props.put(SslConfigs.SSL_KEY_PASSWORD_CONFIG, "key-password");

        // Hostname verification
        props.put(SslConfigs.SSL_ENDPOINT_IDENTIFICATION_ALGORITHM_CONFIG, "https");

        return props;
    }
}
```

## Best Practices

### 1. Принцип минимальных привилегий

```bash
# Создание пользователя с минимальными правами
kafka-acls.sh --bootstrap-server localhost:9094 \
    --command-config admin.properties \
    --add \
    --allow-principal User:app-producer \
    --operation Write \
    --topic orders

# Не давать лишних прав
# ПЛОХО: --operation All
# ХОРОШО: только необходимые операции (Read, Write, Describe)
```

### 2. Сегментация сети

```yaml
# Docker Compose пример изоляции сети
version: '3.8'
services:
  kafka:
    networks:
      - kafka-internal  # Для межброкерного взаимодействия
      - kafka-clients   # Для клиентов
    ports:
      - "9094:9094"     # Только SASL_SSL наружу

networks:
  kafka-internal:
    internal: true      # Изолированная сеть
  kafka-clients:
    driver: bridge
```

### 3. Ротация секретов

```bash
#!/bin/bash
# Скрипт ротации SCRAM credentials

NEW_PASSWORD=$(openssl rand -base64 32)

# Обновление пароля пользователя
kafka-configs.sh --bootstrap-server localhost:9094 \
    --command-config admin.properties \
    --alter \
    --add-config "SCRAM-SHA-512=[password=$NEW_PASSWORD]" \
    --entity-type users \
    --entity-name myuser

echo "Password rotated. Update client configurations."
```

### 4. Мониторинг безопасности

```yaml
# Prometheus метрики для мониторинга безопасности
- name: kafka_authentication_failures
  help: Number of failed authentication attempts
  type: counter

- name: kafka_authorization_denied
  help: Number of denied authorization requests
  type: counter

# AlertManager правила
groups:
  - name: kafka-security
    rules:
      - alert: HighAuthenticationFailures
        expr: rate(kafka_authentication_failures[5m]) > 10
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: High rate of authentication failures
```

### 5. Аудит и логирование

```properties
# Включение аудит логирования
# log4j.properties
log4j.logger.kafka.authorizer.logger=INFO, authorizerAppender
log4j.appender.authorizerAppender=org.apache.log4j.RollingFileAppender
log4j.appender.authorizerAppender.File=/var/log/kafka/kafka-authorizer.log
log4j.appender.authorizerAppender.MaxFileSize=50MB
log4j.appender.authorizerAppender.MaxBackupIndex=10
```

### Чеклист безопасности

```markdown
## Checklist перед production

### Сеть
- [ ] Kafka не доступен напрямую из интернета
- [ ] Межброкерный трафик в изолированной сети
- [ ] Firewall правила ограничивают доступ

### Шифрование
- [ ] TLS 1.2+ для всех подключений
- [ ] Сильные cipher suites
- [ ] Сертификаты от доверенного CA
- [ ] Hostname verification включена

### Аутентификация
- [ ] SASL_SSL для всех клиентов
- [ ] Уникальные credentials для каждого сервиса
- [ ] Пароли достаточной сложности
- [ ] MFA для административного доступа

### Авторизация
- [ ] ACLs настроены для всех топиков
- [ ] Принцип минимальных привилегий
- [ ] Суперпользователи ограничены
- [ ] allow.everyone.if.no.acl.found=false

### Операционная безопасность
- [ ] Логи безопасности собираются централизованно
- [ ] Алерты на подозрительную активность
- [ ] Регулярная ротация credentials
- [ ] Процедуры incident response
```

## Распространённые угрозы и защита

| Угроза | Защита |
|--------|--------|
| Перехват трафика (MITM) | SSL/TLS шифрование |
| Несанкционированный доступ | SASL аутентификация + ACLs |
| Подмена клиента | mTLS (взаимная аутентификация) |
| Brute force атаки | Rate limiting, блокировка IP |
| Утечка credentials | Vault, секретные менеджеры |
| Доступ к данным на диске | Шифрование at rest |
| Privilege escalation | Минимальные привилегии, аудит |

## Дополнительные ресурсы

- [Apache Kafka Security](https://kafka.apache.org/documentation/#security)
- [Confluent Security Overview](https://docs.confluent.io/platform/current/security/index.html)
- [OWASP Kafka Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Apache_Kafka_Security_Cheat_Sheet.html)
