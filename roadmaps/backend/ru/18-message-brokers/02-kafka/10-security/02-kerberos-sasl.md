# Kerberos и SASL в Apache Kafka

## Описание

SASL (Simple Authentication and Security Layer) — это фреймворк для аутентификации, который позволяет подключать различные механизмы проверки подлинности. Apache Kafka поддерживает несколько SASL механизмов, включая интеграцию с Kerberos для корпоративных сред.

Kerberos — это протокол сетевой аутентификации, разработанный MIT, который использует "билеты" для безопасной идентификации пользователей и сервисов в незащищённой сети. Он широко используется в корпоративных средах (Active Directory) и обеспечивает Single Sign-On (SSO).

## Ключевые концепции

### Поддерживаемые SASL механизмы

| Механизм | Описание | Сложность | Использование |
|----------|----------|-----------|---------------|
| PLAIN | Простая передача логина/пароля | Низкая | Разработка, простые среды |
| SCRAM-SHA-256/512 | Challenge-response протокол | Средняя | Production без Kerberos |
| GSSAPI (Kerberos) | Корпоративная аутентификация | Высокая | Enterprise с AD/KDC |
| OAUTHBEARER | OAuth 2.0 токены | Средняя | Облачные среды, микросервисы |

### Архитектура Kerberos

```
┌─────────────────────────────────────────────────────────────────────┐
│                        KERBEROS FLOW                                 │
└─────────────────────────────────────────────────────────────────────┘

  ┌──────────┐                              ┌─────────────────────────┐
  │  CLIENT  │                              │   KDC (Key Distribution │
  │          │                              │        Center)          │
  │ Principal│                              │  ┌─────────────────────┐│
  │ user@REALM│                             │  │  Authentication     ││
  └────┬─────┘                              │  │  Server (AS)        ││
       │                                    │  └─────────────────────┘│
       │ 1. AS_REQ (username)               │  ┌─────────────────────┐│
       │─────────────────────────────────►  │  │  Ticket Granting    ││
       │                                    │  │  Server (TGS)       ││
       │ 2. AS_REP (TGT)                    │  └─────────────────────┘│
       │◄─────────────────────────────────  └─────────────────────────┘
       │
       │ 3. TGS_REQ (TGT + service)
       │─────────────────────────────────►  KDC
       │
       │ 4. TGS_REP (Service Ticket)
       │◄─────────────────────────────────
       │
       │ 5. AP_REQ (Service Ticket)         ┌──────────────────────────┐
       │─────────────────────────────────►  │      KAFKA BROKER        │
       │                                    │                          │
       │ 6. AP_REP (Mutual Auth)            │ Principal:               │
       │◄─────────────────────────────────  │ kafka/broker1.example.com│
       │                                    │ @EXAMPLE.COM             │
       │                                    └──────────────────────────┘

TGT = Ticket Granting Ticket
KDC = Key Distribution Center
AP = Application Protocol
```

### SCRAM (Salted Challenge Response Authentication Mechanism)

```
┌─────────────────────────────────────────────────────────────────────┐
│                         SCRAM FLOW                                   │
└─────────────────────────────────────────────────────────────────────┘

  CLIENT                                              SERVER
    │                                                    │
    │  1. ClientFirstMessage                             │
    │     (username, client nonce)                       │
    │ ──────────────────────────────────────────────────►│
    │                                                    │
    │  2. ServerFirstMessage                             │
    │     (salt, iteration count, server nonce)          │
    │ ◄──────────────────────────────────────────────────│
    │                                                    │
    │  [Client вычисляет proof]                          │
    │                                                    │
    │  3. ClientFinalMessage                             │
    │     (client proof)                                 │
    │ ──────────────────────────────────────────────────►│
    │                                                    │
    │                                      [Server верифицирует]
    │                                                    │
    │  4. ServerFinalMessage                             │
    │     (server signature)                             │
    │ ◄──────────────────────────────────────────────────│
    │                                                    │
    │  [Client верифицирует сервер]                      │
    │                                                    │

Преимущества SCRAM:
- Пароль никогда не передаётся по сети
- Взаимная аутентификация
- Защита от replay атак
- Устойчивость к rainbow tables (salt)
```

## Примеры конфигурации

### SASL/PLAIN

Самый простой механизм, рекомендуется только с SSL:

```properties
# server.properties (брокер)
listeners=SASL_SSL://0.0.0.0:9094
advertised.listeners=SASL_SSL://kafka.example.com:9094

security.inter.broker.protocol=SASL_SSL
sasl.mechanism.inter.broker.protocol=PLAIN
sasl.enabled.mechanisms=PLAIN

# JAAS конфигурация для брокера
listener.name.sasl_ssl.plain.sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
    username="admin" \
    password="admin-secret" \
    user_admin="admin-secret" \
    user_producer="producer-secret" \
    user_consumer="consumer-secret";
```

```properties
# client.properties (клиент)
security.protocol=SASL_SSL
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
    username="producer" \
    password="producer-secret";
```

### SASL/SCRAM-SHA-512

Более безопасный механизм с поддержкой динамического управления пользователями:

```bash
# Создание пользователей в ZooKeeper
# Администратор
kafka-configs.sh --zookeeper localhost:2181 \
    --alter \
    --add-config 'SCRAM-SHA-512=[password=admin-secret]' \
    --entity-type users \
    --entity-name admin

# Producer
kafka-configs.sh --zookeeper localhost:2181 \
    --alter \
    --add-config 'SCRAM-SHA-512=[password=producer-secret]' \
    --entity-type users \
    --entity-name producer

# Consumer
kafka-configs.sh --zookeeper localhost:2181 \
    --alter \
    --add-config 'SCRAM-SHA-512=[password=consumer-secret]' \
    --entity-type users \
    --entity-name consumer

# Просмотр пользователей
kafka-configs.sh --zookeeper localhost:2181 \
    --describe \
    --entity-type users
```

```properties
# server.properties (брокер)
listeners=SASL_SSL://0.0.0.0:9094
advertised.listeners=SASL_SSL://kafka.example.com:9094

security.inter.broker.protocol=SASL_SSL
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-512
sasl.enabled.mechanisms=SCRAM-SHA-512

# JAAS для межброкерного взаимодействия
listener.name.sasl_ssl.scram-sha-512.sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
    username="admin" \
    password="admin-secret";
```

```properties
# producer.properties
security.protocol=SASL_SSL
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
    username="producer" \
    password="producer-secret";

ssl.truststore.location=/path/to/truststore.jks
ssl.truststore.password=truststore-password
```

### SASL/GSSAPI (Kerberos)

Для корпоративных сред с Active Directory или MIT Kerberos:

```properties
# server.properties (брокер)
listeners=SASL_SSL://0.0.0.0:9094
advertised.listeners=SASL_SSL://kafka.example.com:9094

security.inter.broker.protocol=SASL_SSL
sasl.mechanism.inter.broker.protocol=GSSAPI
sasl.enabled.mechanisms=GSSAPI
sasl.kerberos.service.name=kafka

# JAAS конфигурация
listener.name.sasl_ssl.gssapi.sasl.jaas.config=com.sun.security.auth.module.Krb5LoginModule required \
    useKeyTab=true \
    storeKey=true \
    keyTab="/etc/security/keytabs/kafka.service.keytab" \
    principal="kafka/kafka.example.com@EXAMPLE.COM";
```

**kafka_server_jaas.conf** (отдельный файл JAAS):

```
KafkaServer {
    com.sun.security.auth.module.Krb5LoginModule required
    useKeyTab=true
    storeKey=true
    keyTab="/etc/security/keytabs/kafka.service.keytab"
    principal="kafka/kafka.example.com@EXAMPLE.COM";
};

KafkaClient {
    com.sun.security.auth.module.Krb5LoginModule required
    useKeyTab=true
    storeKey=true
    keyTab="/etc/security/keytabs/kafka.service.keytab"
    principal="kafka/kafka.example.com@EXAMPLE.COM";
};

Client {
    com.sun.security.auth.module.Krb5LoginModule required
    useKeyTab=true
    storeKey=true
    keyTab="/etc/security/keytabs/kafka.service.keytab"
    principal="kafka/kafka.example.com@EXAMPLE.COM";
};
```

**krb5.conf** (конфигурация Kerberos):

```ini
[libdefaults]
    default_realm = EXAMPLE.COM
    dns_lookup_realm = false
    dns_lookup_kdc = false
    ticket_lifetime = 24h
    renew_lifetime = 7d
    forwardable = true
    rdns = false

[realms]
    EXAMPLE.COM = {
        kdc = kdc.example.com:88
        admin_server = kdc.example.com:749
    }

[domain_realm]
    .example.com = EXAMPLE.COM
    example.com = EXAMPLE.COM
```

Запуск брокера с Kerberos:

```bash
export KAFKA_OPTS="-Djava.security.krb5.conf=/etc/krb5.conf \
    -Djava.security.auth.login.config=/etc/kafka/kafka_server_jaas.conf"

kafka-server-start.sh /etc/kafka/server.properties
```

### SASL/OAUTHBEARER

Для интеграции с OAuth 2.0 провайдерами (Keycloak, Okta, Azure AD):

```properties
# server.properties
listeners=SASL_SSL://0.0.0.0:9094
sasl.enabled.mechanisms=OAUTHBEARER
sasl.mechanism.inter.broker.protocol=OAUTHBEARER

# Custom callback handler для валидации токенов
listener.name.sasl_ssl.oauthbearer.sasl.jaas.config=org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required;
listener.name.sasl_ssl.oauthbearer.sasl.server.callback.handler.class=com.example.OAuthBearerValidatorCallbackHandler
listener.name.sasl_ssl.oauthbearer.sasl.login.callback.handler.class=com.example.OAuthBearerTokenCallbackHandler
```

Пример callback handler для Keycloak:

```java
public class KeycloakOAuthBearerValidatorCallbackHandler
        implements AuthenticateCallbackHandler {

    private String jwksUrl;
    private String issuer;

    @Override
    public void configure(Map<String, ?> configs, String mechanism,
                         List<AppConfigurationEntry> jaasConfigEntries) {
        this.jwksUrl = (String) configs.get("oauth.jwks.url");
        this.issuer = (String) configs.get("oauth.issuer");
    }

    @Override
    public void handle(Callback[] callbacks) throws IOException, UnsupportedCallbackException {
        for (Callback callback : callbacks) {
            if (callback instanceof OAuthBearerValidatorCallback) {
                OAuthBearerValidatorCallback validatorCallback =
                    (OAuthBearerValidatorCallback) callback;

                String token = validatorCallback.tokenValue();
                OAuthBearerToken validated = validateToken(token);
                validatorCallback.token(validated);
            }
        }
    }

    private OAuthBearerToken validateToken(String token) {
        // Валидация JWT токена через Keycloak JWKS
        // Проверка подписи, issuer, audience, expiration
        // ...
    }
}
```

### Множественные SASL механизмы

```properties
# server.properties - поддержка нескольких механизмов
listeners=SASL_SSL://0.0.0.0:9094
sasl.enabled.mechanisms=SCRAM-SHA-512,PLAIN,OAUTHBEARER
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-512

# JAAS для каждого механизма
listener.name.sasl_ssl.scram-sha-512.sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
    username="admin" \
    password="admin-secret";

listener.name.sasl_ssl.plain.sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
    username="admin" \
    password="admin-secret" \
    user_admin="admin-secret" \
    user_legacy="legacy-secret";

listener.name.sasl_ssl.oauthbearer.sasl.jaas.config=org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required;
```

## Примеры клиентского кода

### Python с SASL/SCRAM

```python
from kafka import KafkaProducer, KafkaConsumer

# Конфигурация SCRAM-SHA-512
config = {
    'bootstrap_servers': ['kafka.example.com:9094'],
    'security_protocol': 'SASL_SSL',
    'sasl_mechanism': 'SCRAM-SHA-512',
    'sasl_plain_username': 'producer',
    'sasl_plain_password': 'producer-secret',
    'ssl_cafile': '/path/to/ca-cert.pem',
}

producer = KafkaProducer(**config)
producer.send('my-topic', b'Hello, Kafka!')
producer.flush()
```

### Java с Kerberos

```java
import org.apache.kafka.clients.producer.*;
import java.util.Properties;

public class KerberosProducer {
    public static void main(String[] args) {
        // Установка системных свойств Kerberos
        System.setProperty("java.security.krb5.conf", "/etc/krb5.conf");
        System.setProperty("java.security.auth.login.config",
                          "/etc/kafka/kafka_client_jaas.conf");

        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka.example.com:9094");
        props.put("security.protocol", "SASL_SSL");
        props.put("sasl.mechanism", "GSSAPI");
        props.put("sasl.kerberos.service.name", "kafka");
        props.put("ssl.truststore.location", "/path/to/truststore.jks");
        props.put("ssl.truststore.password", "truststore-password");

        try (KafkaProducer<String, String> producer = new KafkaProducer<>(props)) {
            ProducerRecord<String, String> record =
                new ProducerRecord<>("my-topic", "key", "value");
            producer.send(record).get();
        }
    }
}
```

**kafka_client_jaas.conf**:

```
KafkaClient {
    com.sun.security.auth.module.Krb5LoginModule required
    useTicketCache=true
    renewTicket=true
    serviceName="kafka";
};
```

### Go с SASL/SCRAM

```go
package main

import (
    "github.com/IBM/sarama"
    "log"
)

func main() {
    config := sarama.NewConfig()
    config.Net.SASL.Enable = true
    config.Net.SASL.Mechanism = sarama.SASLTypeSCRAMSHA512
    config.Net.SASL.User = "producer"
    config.Net.SASL.Password = "producer-secret"
    config.Net.SASL.SCRAMClientGeneratorFunc = func() sarama.SCRAMClient {
        return &XDGSCRAMClient{HashGeneratorFcn: SHA512}
    }

    config.Net.TLS.Enable = true
    config.Net.TLS.Config = createTLSConfig()

    producer, err := sarama.NewSyncProducer([]string{"kafka.example.com:9094"}, config)
    if err != nil {
        log.Fatal(err)
    }
    defer producer.Close()

    msg := &sarama.ProducerMessage{
        Topic: "my-topic",
        Value: sarama.StringEncoder("Hello, Kafka!"),
    }

    _, _, err = producer.SendMessage(msg)
    if err != nil {
        log.Fatal(err)
    }
}
```

## Best Practices

### 1. Выбор SASL механизма

```markdown
## Рекомендации по выбору

### SCRAM-SHA-512 (рекомендуется для большинства случаев)
✅ Безопасный, пароль не передаётся
✅ Простая настройка
✅ Динамическое управление пользователями
✅ Не требует внешней инфраструктуры

### GSSAPI/Kerberos
✅ Корпоративные среды с AD
✅ Single Sign-On
✅ Централизованное управление
❌ Сложная настройка
❌ Требует KDC инфраструктуру

### OAUTHBEARER
✅ Микросервисные архитектуры
✅ Интеграция с IdP (Keycloak, Okta)
✅ Короткоживущие токены
❌ Требует OAuth сервер
❌ Дополнительная разработка callback handlers

### PLAIN
⚠️ Только для разработки
⚠️ Обязательно с SSL
❌ Пароль передаётся в открытом виде
```

### 2. Безопасное хранение credentials

```bash
# Использование HashiCorp Vault
vault kv put secret/kafka/producer \
    username=producer \
    password=$(openssl rand -base64 32)

# Получение в приложении
vault kv get -format=json secret/kafka/producer | jq -r '.data.data.password'
```

```yaml
# Kubernetes Secrets
apiVersion: v1
kind: Secret
metadata:
  name: kafka-credentials
type: Opaque
stringData:
  sasl-username: producer
  sasl-password: <base64-encoded-password>
---
# Использование в Pod
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: kafka-client
      env:
        - name: KAFKA_SASL_USERNAME
          valueFrom:
            secretKeyRef:
              name: kafka-credentials
              key: sasl-username
        - name: KAFKA_SASL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: kafka-credentials
              key: sasl-password
```

### 3. Ротация credentials

```bash
#!/bin/bash
# rotate-kafka-credentials.sh

USER=$1
OLD_PASSWORD=$2

# Генерация нового пароля
NEW_PASSWORD=$(openssl rand -base64 32)

# Обновление в Kafka
kafka-configs.sh --bootstrap-server kafka:9094 \
    --command-config admin.properties \
    --alter \
    --add-config "SCRAM-SHA-512=[password=$NEW_PASSWORD]" \
    --entity-type users \
    --entity-name $USER

# Обновление в Vault
vault kv put secret/kafka/$USER \
    username=$USER \
    password=$NEW_PASSWORD \
    previous_password=$OLD_PASSWORD

echo "Credentials rotated for $USER"
echo "Previous password kept for graceful transition"
```

### 4. Мониторинг аутентификации

```java
// Метрики аутентификации
// JMX beans для мониторинга
kafka.server:type=BrokerTopicMetrics,name=FailedAuthenticationRate
kafka.server:type=BrokerTopicMetrics,name=SuccessfulAuthenticationRate
kafka.server:type=BrokerTopicMetrics,name=FailedReauthenticationRate
```

```yaml
# Prometheus правила алертов
groups:
  - name: kafka-sasl-alerts
    rules:
      - alert: KafkaHighAuthenticationFailureRate
        expr: kafka_server_broker_topic_metrics_failed_authentication_rate > 5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High authentication failure rate on {{ $labels.instance }}"

      - alert: KafkaAuthenticationServiceDown
        expr: up{job="kafka-auth-metrics"} == 0
        for: 1m
        labels:
          severity: critical
```

### 5. Troubleshooting SASL

```bash
# Включение отладки SASL
export KAFKA_OPTS="-Dsun.security.krb5.debug=true \
    -Djava.security.debug=gssloginconfig,configfile,configparser,logincontext"

# Проверка Kerberos ticket
klist -e

# Получение нового ticket
kinit -kt /path/to/keytab principal@REALM

# Тестирование подключения
kafka-console-producer.sh \
    --bootstrap-server kafka:9094 \
    --producer.config client.properties \
    --topic test
```

## Сравнение механизмов

| Критерий | PLAIN | SCRAM | GSSAPI | OAUTHBEARER |
|----------|-------|-------|--------|-------------|
| Безопасность | Низкая | Высокая | Очень высокая | Высокая |
| Сложность | Низкая | Средняя | Высокая | Средняя |
| Инфраструктура | - | ZooKeeper | KDC/AD | OAuth Server |
| SSO | Нет | Нет | Да | Да |
| Токены | Нет | Нет | Нет | Да |
| Динамические пользователи | Нет | Да | Через AD | Через IdP |
| Рекомендация | Dev only | Production | Enterprise | Cloud/Microservices |

## Дополнительные ресурсы

- [Apache Kafka SASL Configuration](https://kafka.apache.org/documentation/#security_sasl)
- [MIT Kerberos Documentation](https://web.mit.edu/kerberos/)
- [RFC 5802 - SCRAM](https://tools.ietf.org/html/rfc5802)
- [RFC 6749 - OAuth 2.0](https://tools.ietf.org/html/rfc6749)
