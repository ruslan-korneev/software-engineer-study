# TLS/SSL в RabbitMQ

[prev: 02-authorization](./02-authorization.md) | [next: 04-access-control](./04-access-control.md)

---

## Введение

TLS (Transport Layer Security) обеспечивает шифрование данных между клиентами и RabbitMQ, а также между узлами кластера. Это критически важно для защиты учётных данных и сообщений от перехвата.

## Основные концепции

### Компоненты PKI

| Компонент | Описание |
|-----------|----------|
| **CA Certificate** | Сертификат центра сертификации (корневой или промежуточный) |
| **Server Certificate** | Сертификат сервера RabbitMQ |
| **Server Private Key** | Закрытый ключ сервера |
| **Client Certificate** | Сертификат клиента (для mutual TLS) |
| **Client Private Key** | Закрытый ключ клиента |

### Режимы TLS

1. **Server-side TLS** — только сервер предоставляет сертификат
2. **Mutual TLS (mTLS)** — и сервер, и клиент предоставляют сертификаты

## Генерация сертификатов

### Создание CA (Certificate Authority)

```bash
# Создание директории для сертификатов
mkdir -p /etc/rabbitmq/ssl
cd /etc/rabbitmq/ssl

# Генерация приватного ключа CA
openssl genrsa -out ca_key.pem 4096

# Создание самоподписанного CA сертификата
openssl req -x509 -new -nodes \
    -key ca_key.pem \
    -sha256 \
    -days 3650 \
    -out ca_certificate.pem \
    -subj "/C=RU/ST=Moscow/L=Moscow/O=MyCompany/CN=RabbitMQ CA"
```

### Создание серверного сертификата

```bash
# Генерация приватного ключа сервера
openssl genrsa -out server_key.pem 4096

# Создание CSR (Certificate Signing Request)
openssl req -new \
    -key server_key.pem \
    -out server_csr.pem \
    -subj "/C=RU/ST=Moscow/L=Moscow/O=MyCompany/CN=rabbitmq.example.com"

# Файл расширений для SAN (Subject Alternative Names)
cat > server_extensions.cnf << EOF
basicConstraints = CA:FALSE
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1 = rabbitmq.example.com
DNS.2 = rabbitmq-node1.example.com
DNS.3 = rabbitmq-node2.example.com
DNS.4 = localhost
IP.1 = 192.168.1.100
IP.2 = 127.0.0.1
EOF

# Подписание сертификата CA
openssl x509 -req \
    -in server_csr.pem \
    -CA ca_certificate.pem \
    -CAkey ca_key.pem \
    -CAcreateserial \
    -out server_certificate.pem \
    -days 365 \
    -sha256 \
    -extfile server_extensions.cnf
```

### Создание клиентского сертификата (для mTLS)

```bash
# Генерация приватного ключа клиента
openssl genrsa -out client_key.pem 4096

# Создание CSR
openssl req -new \
    -key client_key.pem \
    -out client_csr.pem \
    -subj "/C=RU/ST=Moscow/L=Moscow/O=MyCompany/CN=order-service"

# Файл расширений для клиента
cat > client_extensions.cnf << EOF
basicConstraints = CA:FALSE
keyUsage = digitalSignature
extendedKeyUsage = clientAuth
EOF

# Подписание сертификата
openssl x509 -req \
    -in client_csr.pem \
    -CA ca_certificate.pem \
    -CAkey ca_key.pem \
    -CAcreateserial \
    -out client_certificate.pem \
    -days 365 \
    -sha256 \
    -extfile client_extensions.cnf
```

## Настройка RabbitMQ

### Базовая конфигурация TLS

```erlang
% rabbitmq.conf

% Отключение незашифрованного порта (опционально)
listeners.tcp = none

% Включение TLS порта
listeners.ssl.default = 5671

% Пути к сертификатам
ssl_options.cacertfile = /etc/rabbitmq/ssl/ca_certificate.pem
ssl_options.certfile = /etc/rabbitmq/ssl/server_certificate.pem
ssl_options.keyfile = /etc/rabbitmq/ssl/server_key.pem

% Минимальная версия TLS
ssl_options.versions.1 = tlsv1.3
ssl_options.versions.2 = tlsv1.2
```

### Настройка Mutual TLS

```erlang
% rabbitmq.conf

% Требование клиентского сертификата
ssl_options.verify = verify_peer
ssl_options.fail_if_no_peer_cert = true

% Глубина проверки цепочки сертификатов
ssl_options.depth = 2

% Использование сертификата для аутентификации
auth_mechanisms.1 = EXTERNAL
auth_mechanisms.2 = PLAIN

% Извлечение имени пользователя из сертификата
ssl_cert_login_from = common_name
```

### Настройка cipher suites

```erlang
% rabbitmq.conf

% Только надёжные шифры
ssl_options.ciphers.1 = TLS_AES_256_GCM_SHA384
ssl_options.ciphers.2 = TLS_AES_128_GCM_SHA256
ssl_options.ciphers.3 = TLS_CHACHA20_POLY1305_SHA256
ssl_options.ciphers.4 = ECDHE-RSA-AES256-GCM-SHA384
ssl_options.ciphers.5 = ECDHE-RSA-AES128-GCM-SHA256

% Порядок шифров определяется сервером
ssl_options.honor_cipher_order = true
ssl_options.honor_ecc_order = true
```

## TLS для Management UI

```erlang
% rabbitmq.conf

% HTTPS для Management UI
management.ssl.port = 15671
management.ssl.cacertfile = /etc/rabbitmq/ssl/ca_certificate.pem
management.ssl.certfile = /etc/rabbitmq/ssl/server_certificate.pem
management.ssl.keyfile = /etc/rabbitmq/ssl/server_key.pem

% Отключение HTTP
management.tcp.port = none
```

## TLS между узлами кластера

```erlang
% rabbitmq.conf

% Включение TLS для inter-node communication
cluster_formation.peer_discovery_backend = rabbit_peer_discovery_classic_config
cluster_formation.classic_config.nodes.1 = rabbit@node1
cluster_formation.classic_config.nodes.2 = rabbit@node2

% TLS для распределённого Erlang
ssl_options.cacertfile = /etc/rabbitmq/ssl/ca_certificate.pem
ssl_options.certfile = /etc/rabbitmq/ssl/server_certificate.pem
ssl_options.keyfile = /etc/rabbitmq/ssl/server_key.pem
```

Дополнительный файл конфигурации Erlang (`/etc/rabbitmq/inter_node_tls.config`):

```erlang
[
  {server, [
    {cacertfile, "/etc/rabbitmq/ssl/ca_certificate.pem"},
    {certfile, "/etc/rabbitmq/ssl/server_certificate.pem"},
    {keyfile, "/etc/rabbitmq/ssl/server_key.pem"},
    {verify, verify_peer},
    {fail_if_no_peer_cert, true}
  ]},
  {client, [
    {cacertfile, "/etc/rabbitmq/ssl/ca_certificate.pem"},
    {certfile, "/etc/rabbitmq/ssl/server_certificate.pem"},
    {keyfile, "/etc/rabbitmq/ssl/server_key.pem"},
    {verify, verify_peer}
  ]}
].
```

Запуск с TLS между узлами:

```bash
export RABBITMQ_CTL_ERL_ARGS="-proto_dist inet_tls"
export RABBITMQ_SERVER_ADDITIONAL_ERL_ARGS="-proto_dist inet_tls \
    -ssl_dist_optfile /etc/rabbitmq/inter_node_tls.config"
```

## Подключение клиентов

### Python (pika)

```python
import ssl
import pika

# Создание SSL контекста
context = ssl.SSLContext(ssl.PROTOCOL_TLS_CLIENT)
context.minimum_version = ssl.TLSVersion.TLSv1_2
context.verify_mode = ssl.CERT_REQUIRED
context.load_verify_locations('/path/to/ca_certificate.pem')

# Для mTLS — загрузка клиентского сертификата
context.load_cert_chain(
    certfile='/path/to/client_certificate.pem',
    keyfile='/path/to/client_key.pem'
)

# Подключение с TLS
credentials = pika.PlainCredentials('user', 'password')
# Или для mTLS:
# credentials = pika.credentials.ExternalCredentials()

parameters = pika.ConnectionParameters(
    host='rabbitmq.example.com',
    port=5671,
    virtual_host='/',
    credentials=credentials,
    ssl_options=pika.SSLOptions(context, 'rabbitmq.example.com')
)

connection = pika.BlockingConnection(parameters)
channel = connection.channel()
```

### Java (Spring AMQP)

```java
import org.springframework.amqp.rabbit.connection.CachingConnectionFactory;
import org.springframework.amqp.rabbit.connection.RabbitConnectionFactoryBean;

@Configuration
public class RabbitMQConfig {

    @Bean
    public CachingConnectionFactory connectionFactory() throws Exception {
        RabbitConnectionFactoryBean factoryBean = new RabbitConnectionFactoryBean();
        factoryBean.setHost("rabbitmq.example.com");
        factoryBean.setPort(5671);
        factoryBean.setUsername("user");
        factoryBean.setPassword("password");

        // TLS настройки
        factoryBean.setUseSSL(true);
        factoryBean.setSslAlgorithm("TLSv1.3");
        factoryBean.setKeyStore("/path/to/client_keystore.jks");
        factoryBean.setKeyStorePassphrase("keystorepassword");
        factoryBean.setTrustStore("/path/to/truststore.jks");
        factoryBean.setTrustStorePassphrase("truststorepassword");

        factoryBean.afterPropertiesSet();

        CachingConnectionFactory connectionFactory =
            new CachingConnectionFactory(factoryBean.getObject());
        return connectionFactory;
    }
}
```

### Go (amqp091-go)

```go
package main

import (
    "crypto/tls"
    "crypto/x509"
    "io/ioutil"
    "log"

    amqp "github.com/rabbitmq/amqp091-go"
)

func main() {
    // Загрузка CA сертификата
    caCert, err := ioutil.ReadFile("/path/to/ca_certificate.pem")
    if err != nil {
        log.Fatal(err)
    }

    caCertPool := x509.NewCertPool()
    caCertPool.AppendCertsFromPEM(caCert)

    // Загрузка клиентского сертификата (для mTLS)
    cert, err := tls.LoadX509KeyPair(
        "/path/to/client_certificate.pem",
        "/path/to/client_key.pem",
    )
    if err != nil {
        log.Fatal(err)
    }

    // TLS конфигурация
    tlsConfig := &tls.Config{
        RootCAs:      caCertPool,
        Certificates: []tls.Certificate{cert},
        MinVersion:   tls.VersionTLS12,
        ServerName:   "rabbitmq.example.com",
    }

    // Подключение
    conn, err := amqp.DialTLS(
        "amqps://user:password@rabbitmq.example.com:5671/",
        tlsConfig,
    )
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Close()
}
```

## Проверка конфигурации

### Проверка сертификатов

```bash
# Проверка серверного сертификата
openssl x509 -in server_certificate.pem -text -noout

# Проверка цепочки сертификатов
openssl verify -CAfile ca_certificate.pem server_certificate.pem

# Проверка соответствия ключа и сертификата
openssl x509 -noout -modulus -in server_certificate.pem | openssl md5
openssl rsa -noout -modulus -in server_key.pem | openssl md5
# MD5 хэши должны совпадать
```

### Тестирование TLS соединения

```bash
# Проверка TLS handshake
openssl s_client -connect rabbitmq.example.com:5671 \
    -CAfile ca_certificate.pem \
    -cert client_certificate.pem \
    -key client_key.pem \
    -verify 2

# Проверка поддерживаемых протоколов
nmap --script ssl-enum-ciphers -p 5671 rabbitmq.example.com
```

### Диагностика через RabbitMQ

```bash
# Проверка статуса TLS
rabbitmqctl eval 'ssl:versions().'

# Проверка активных соединений
rabbitmqctl list_connections name ssl ssl_protocol ssl_cipher
```

## Best Practices

### 1. Используйте только современные протоколы

```erlang
% Только TLS 1.2 и 1.3
ssl_options.versions.1 = tlsv1.3
ssl_options.versions.2 = tlsv1.2

% Отключите устаревшие
% НЕ используйте: tlsv1.1, tlsv1, sslv3
```

### 2. Регулярно обновляйте сертификаты

```bash
#!/bin/bash
# Скрипт проверки срока действия сертификатов
CERT_FILE="/etc/rabbitmq/ssl/server_certificate.pem"
DAYS_WARNING=30

EXPIRY_DATE=$(openssl x509 -enddate -noout -in "$CERT_FILE" | cut -d= -f2)
EXPIRY_EPOCH=$(date -d "$EXPIRY_DATE" +%s)
CURRENT_EPOCH=$(date +%s)
DAYS_LEFT=$(( (EXPIRY_EPOCH - CURRENT_EPOCH) / 86400 ))

if [ $DAYS_LEFT -lt $DAYS_WARNING ]; then
    echo "WARNING: Certificate expires in $DAYS_LEFT days"
    # Отправка уведомления
fi
```

### 3. Защитите приватные ключи

```bash
# Установка правильных прав доступа
chmod 600 /etc/rabbitmq/ssl/*_key.pem
chown rabbitmq:rabbitmq /etc/rabbitmq/ssl/*

# Никогда не храните ключи в репозитории!
```

### 4. Используйте Hardware Security Modules (HSM) в production

```erlang
% Использование PKCS#11 для хранения ключей
ssl_options.keyfile = engine:pkcs11:slot_0-id_1
```

## Распространённые проблемы

### Ошибка: certificate verify failed

**Причина**: CA сертификат не добавлен в trust store.

**Решение**: Убедитесь, что `cacertfile` указывает на правильный CA сертификат.

### Ошибка: no suitable cipher

**Причина**: Несовместимые cipher suites между клиентом и сервером.

**Решение**: Проверьте и согласуйте списки поддерживаемых шифров.

### Ошибка: hostname verification failed

**Причина**: CN или SAN в сертификате не соответствует hostname.

**Решение**: Добавьте правильные DNS имена в SAN сертификата.

## Мониторинг TLS

```bash
# Prometheus метрики через Management Plugin
curl -s http://localhost:15692/metrics | grep ssl

# Логирование TLS событий
# В rabbitmq.conf:
log.connection.level = info
```

## Резюме

- Всегда используйте TLS в production для защиты данных
- Предпочитайте Mutual TLS для аутентификации сервисов
- Используйте только TLS 1.2+ и современные cipher suites
- Регулярно проверяйте и обновляйте сертификаты
- Защищайте приватные ключи правильными правами доступа
- Включайте TLS для всех компонентов: AMQP, Management UI, inter-node

---

[prev: 02-authorization](./02-authorization.md) | [next: 04-access-control](./04-access-control.md)
