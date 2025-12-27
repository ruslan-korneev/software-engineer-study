# Квоты в Apache Kafka

## Описание

Квоты (Quotas) в Kafka — это механизм ограничения ресурсов, который позволяет контролировать пропускную способность клиентов и защищать кластер от перегрузки. Квоты предотвращают ситуации, когда один "шумный" клиент потребляет все ресурсы кластера, влияя на работу других клиентов.

Система квот в Kafka позволяет ограничивать:
- **Скорость производства** (producer throughput) — байт/сек
- **Скорость потребления** (consumer throughput) — байт/сек
- **Скорость запросов** (request rate) — процент CPU
- **Количество подключений** (connection rate) — соединений/сек

## Ключевые концепции

### Типы квот

```
┌─────────────────────────────────────────────────────────────────────┐
│                        ТИПЫ КВОТ KAFKA                              │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  1. NETWORK QUOTAS (Сетевые квоты)                                  │
│     ├── producer_byte_rate  — лимит байт/сек для отправки           │
│     └── consumer_byte_rate  — лимит байт/сек для получения          │
│                                                                     │
│  2. REQUEST QUOTAS (Квоты запросов)                                 │
│     └── request_percentage  — % CPU времени на обработку запросов   │
│                                                                     │
│  3. CONNECTION QUOTAS (Квоты подключений)                           │
│     └── connection_creation_rate — соединений/сек с одного IP       │
│                                                                     │
│  4. CONTROLLER MUTATION QUOTAS (Квоты мутаций)                      │
│     └── controller_mutation_rate — операций/сек на controller       │
└─────────────────────────────────────────────────────────────────────┘
```

### Уровни применения квот

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ИЕРАРХИЯ КВОТ                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Приоритет (от высшего к низшему):                                  │
│                                                                     │
│  1. /config/users/<user>/clients/<client-id>                        │
│     └── Квота для конкретного user + client-id                      │
│                                                                     │
│  2. /config/users/<user>/clients/<default>                          │
│     └── Квота по умолчанию для user (любой client-id)               │
│                                                                     │
│  3. /config/users/<user>                                            │
│     └── Квота для user                                              │
│                                                                     │
│  4. /config/users/<default>/clients/<client-id>                     │
│     └── Квота для client-id (любой user)                            │
│                                                                     │
│  5. /config/users/<default>/clients/<default>                       │
│     └── Глобальная квота по умолчанию                               │
│                                                                     │
│  6. /config/users/<default>                                         │
│     └── Квота по умолчанию для всех users                           │
│                                                                     │
│  7. /config/clients/<client-id>                                     │
│     └── Квота для client-id (legacy)                                │
│                                                                     │
│  8. /config/clients/<default>                                       │
│     └── Квота по умолчанию для всех client-ids (legacy)             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Механизм throttling

```
┌─────────────────────────────────────────────────────────────────────┐
│                    КАК РАБОТАЕТ THROTTLING                          │
└─────────────────────────────────────────────────────────────────────┘

   CLIENT                                              BROKER
     │                                                    │
     │  Produce Request (1MB)                             │
     │ ──────────────────────────────────────────────────►│
     │                                                    │
     │                          [Проверка квоты]          │
     │                          Лимит: 500KB/s            │
     │                          Превышение: да            │
     │                                                    │
     │  Produce Response                                  │
     │  + throttle_time_ms=2000                          │
     │ ◄──────────────────────────────────────────────────│
     │                                                    │
     │  [Клиент ждёт 2 секунды]                          │
     │                                                    │
     │  Следующий Produce Request                         │
     │ ──────────────────────────────────────────────────►│
     │                                                    │

Формула: throttle_time = (excess_bytes / quota_rate) * 1000

Пример:
- Отправлено: 1MB = 1,048,576 байт
- Квота: 500KB/s = 512,000 байт/сек
- Превышение: 1,048,576 - 512,000 = 536,576 байт
- Throttle time: (536,576 / 512,000) * 1000 ≈ 1048 мс
```

## Примеры конфигурации

### Конфигурация брокера для квот

```properties
# server.properties

# Дефолтные квоты (применяются если не заданы специфичные)
# Эти параметры deprecated, рекомендуется использовать kafka-configs.sh

# Максимум байт/сек для producers без специфичной квоты
# quota.producer.default=10485760  # 10 MB/s

# Максимум байт/сек для consumers без специфичной квоты
# quota.consumer.default=10485760  # 10 MB/s

# Окно измерения квот (по умолчанию 1 секунда)
quota.window.num=11
quota.window.size.seconds=1

# Количество сэмплов для усреднения
num.quota.samples=11

# Лимит подключений с одного IP (по умолчанию без лимита)
# max.connections.per.ip=100

# Лимит подключений с определённых IP
# max.connections.per.ip.overrides=192.168.1.100:50,192.168.1.101:200
```

### Установка квот через kafka-configs.sh

```bash
# ═══════════════════════════════════════════════════════════════════
#                    USER-BASED QUOTAS
# ═══════════════════════════════════════════════════════════════════

# Установить квоту для конкретного пользователя
# producer_byte_rate = 1MB/s, consumer_byte_rate = 2MB/s
kafka-configs.sh --bootstrap-server kafka:9094 \
    --command-config admin.properties \
    --alter \
    --add-config 'producer_byte_rate=1048576,consumer_byte_rate=2097152' \
    --entity-type users \
    --entity-name order-service

# Установить request_percentage для пользователя (10% CPU)
kafka-configs.sh --bootstrap-server kafka:9094 \
    --command-config admin.properties \
    --alter \
    --add-config 'request_percentage=10' \
    --entity-type users \
    --entity-name batch-processor

# Квота по умолчанию для всех пользователей
kafka-configs.sh --bootstrap-server kafka:9094 \
    --command-config admin.properties \
    --alter \
    --add-config 'producer_byte_rate=5242880,consumer_byte_rate=10485760' \
    --entity-type users \
    --entity-default

# ═══════════════════════════════════════════════════════════════════
#                    CLIENT-ID BASED QUOTAS
# ═══════════════════════════════════════════════════════════════════

# Квота для конкретного client-id
kafka-configs.sh --bootstrap-server kafka:9094 \
    --command-config admin.properties \
    --alter \
    --add-config 'producer_byte_rate=524288' \
    --entity-type clients \
    --entity-name analytics-consumer

# Квота по умолчанию для всех client-ids
kafka-configs.sh --bootstrap-server kafka:9094 \
    --command-config admin.properties \
    --alter \
    --add-config 'consumer_byte_rate=10485760' \
    --entity-type clients \
    --entity-default

# ═══════════════════════════════════════════════════════════════════
#                    USER + CLIENT-ID QUOTAS
# ═══════════════════════════════════════════════════════════════════

# Квота для комбинации user + client-id
kafka-configs.sh --bootstrap-server kafka:9094 \
    --command-config admin.properties \
    --alter \
    --add-config 'producer_byte_rate=2097152' \
    --entity-type users \
    --entity-name order-service \
    --entity-type clients \
    --entity-name order-producer-1

# Квота по умолчанию для всех client-ids конкретного user
kafka-configs.sh --bootstrap-server kafka:9094 \
    --command-config admin.properties \
    --alter \
    --add-config 'consumer_byte_rate=1048576' \
    --entity-type users \
    --entity-name analytics-service \
    --entity-type clients \
    --entity-default

# ═══════════════════════════════════════════════════════════════════
#                    CONNECTION QUOTAS
# ═══════════════════════════════════════════════════════════════════

# Ограничение скорости создания подключений для IP
kafka-configs.sh --bootstrap-server kafka:9094 \
    --command-config admin.properties \
    --alter \
    --add-config 'connection_creation_rate=10' \
    --entity-type ips \
    --entity-name 192.168.1.100

# По умолчанию для всех IP
kafka-configs.sh --bootstrap-server kafka:9094 \
    --command-config admin.properties \
    --alter \
    --add-config 'connection_creation_rate=50' \
    --entity-type ips \
    --entity-default
```

### Просмотр и удаление квот

```bash
# Просмотр всех квот
kafka-configs.sh --bootstrap-server kafka:9094 \
    --command-config admin.properties \
    --describe \
    --entity-type users

# Просмотр квот для конкретного пользователя
kafka-configs.sh --bootstrap-server kafka:9094 \
    --command-config admin.properties \
    --describe \
    --entity-type users \
    --entity-name order-service

# Просмотр квот по умолчанию
kafka-configs.sh --bootstrap-server kafka:9094 \
    --command-config admin.properties \
    --describe \
    --entity-type users \
    --entity-default

# Удаление квоты
kafka-configs.sh --bootstrap-server kafka:9094 \
    --command-config admin.properties \
    --alter \
    --delete-config 'producer_byte_rate' \
    --entity-type users \
    --entity-name order-service

# Удаление всех квот для пользователя
kafka-configs.sh --bootstrap-server kafka:9094 \
    --command-config admin.properties \
    --alter \
    --delete-config 'producer_byte_rate,consumer_byte_rate,request_percentage' \
    --entity-type users \
    --entity-name order-service
```

### Программное управление квотами

**Java Admin Client:**

```java
import org.apache.kafka.clients.admin.*;
import org.apache.kafka.common.quota.*;

import java.util.*;
import java.util.concurrent.ExecutionException;

public class KafkaQuotaManager {

    private final AdminClient adminClient;

    public KafkaQuotaManager(Properties props) {
        this.adminClient = AdminClient.create(props);
    }

    /**
     * Установка квот для пользователя
     */
    public void setUserQuota(String user, long producerBytesPerSec,
                            long consumerBytesPerSec)
            throws ExecutionException, InterruptedException {

        // Создание entity
        ClientQuotaEntity entity = new ClientQuotaEntity(
            Map.of(ClientQuotaEntity.USER, user)
        );

        // Создание операций изменения квот
        Collection<ClientQuotaAlteration> alterations = Collections.singletonList(
            new ClientQuotaAlteration(
                entity,
                List.of(
                    new ClientQuotaAlteration.Op("producer_byte_rate",
                        (double) producerBytesPerSec),
                    new ClientQuotaAlteration.Op("consumer_byte_rate",
                        (double) consumerBytesPerSec)
                )
            )
        );

        // Применение
        AlterClientQuotasResult result = adminClient.alterClientQuotas(alterations);
        result.all().get();

        System.out.println("Quota set for user: " + user);
    }

    /**
     * Установка request_percentage квоты
     */
    public void setRequestQuota(String user, double requestPercentage)
            throws ExecutionException, InterruptedException {

        ClientQuotaEntity entity = new ClientQuotaEntity(
            Map.of(ClientQuotaEntity.USER, user)
        );

        Collection<ClientQuotaAlteration> alterations = Collections.singletonList(
            new ClientQuotaAlteration(
                entity,
                List.of(
                    new ClientQuotaAlteration.Op("request_percentage", requestPercentage)
                )
            )
        );

        adminClient.alterClientQuotas(alterations).all().get();
    }

    /**
     * Установка квоты для user + client-id
     */
    public void setUserClientQuota(String user, String clientId,
                                   long producerBytesPerSec)
            throws ExecutionException, InterruptedException {

        ClientQuotaEntity entity = new ClientQuotaEntity(
            Map.of(
                ClientQuotaEntity.USER, user,
                ClientQuotaEntity.CLIENT_ID, clientId
            )
        );

        Collection<ClientQuotaAlteration> alterations = Collections.singletonList(
            new ClientQuotaAlteration(
                entity,
                List.of(
                    new ClientQuotaAlteration.Op("producer_byte_rate",
                        (double) producerBytesPerSec)
                )
            )
        );

        adminClient.alterClientQuotas(alterations).all().get();
    }

    /**
     * Установка квоты по умолчанию
     */
    public void setDefaultQuota(long producerBytesPerSec, long consumerBytesPerSec)
            throws ExecutionException, InterruptedException {

        // Пустой user = default
        ClientQuotaEntity entity = new ClientQuotaEntity(
            Map.of(ClientQuotaEntity.USER, "")  // default
        );

        Collection<ClientQuotaAlteration> alterations = Collections.singletonList(
            new ClientQuotaAlteration(
                entity,
                List.of(
                    new ClientQuotaAlteration.Op("producer_byte_rate",
                        (double) producerBytesPerSec),
                    new ClientQuotaAlteration.Op("consumer_byte_rate",
                        (double) consumerBytesPerSec)
                )
            )
        );

        adminClient.alterClientQuotas(alterations).all().get();
    }

    /**
     * Получение текущих квот
     */
    public void describeQuotas(String user) throws ExecutionException, InterruptedException {
        ClientQuotaFilter filter = ClientQuotaFilter.containsOnly(
            List.of(
                ClientQuotaFilterComponent.ofEntity(ClientQuotaEntity.USER, user)
            )
        );

        DescribeClientQuotasResult result = adminClient.describeClientQuotas(filter);
        Map<ClientQuotaEntity, Map<String, Double>> quotas = result.entities().get();

        for (Map.Entry<ClientQuotaEntity, Map<String, Double>> entry : quotas.entrySet()) {
            System.out.println("Entity: " + entry.getKey());
            for (Map.Entry<String, Double> quota : entry.getValue().entrySet()) {
                System.out.println("  " + quota.getKey() + " = " + quota.getValue());
            }
        }
    }

    /**
     * Удаление квоты
     */
    public void removeQuota(String user, String quotaName)
            throws ExecutionException, InterruptedException {

        ClientQuotaEntity entity = new ClientQuotaEntity(
            Map.of(ClientQuotaEntity.USER, user)
        );

        Collection<ClientQuotaAlteration> alterations = Collections.singletonList(
            new ClientQuotaAlteration(
                entity,
                List.of(
                    new ClientQuotaAlteration.Op(quotaName, null)  // null = удаление
                )
            )
        );

        adminClient.alterClientQuotas(alterations).all().get();
    }

    public void close() {
        adminClient.close();
    }
}
```

**Python пример:**

```python
from confluent_kafka.admin import AdminClient, ConfigResource

class QuotaManager:
    def __init__(self, config):
        self.admin = AdminClient(config)

    def set_user_quota(self, user: str, producer_bytes: int, consumer_bytes: int):
        """Установка квот для пользователя через kafka-configs"""
        # confluent-kafka-python пока не имеет прямого API для квот
        # Используем subprocess для вызова kafka-configs.sh
        import subprocess

        cmd = [
            'kafka-configs.sh',
            '--bootstrap-server', self.admin._config.get('bootstrap.servers'),
            '--command-config', 'admin.properties',
            '--alter',
            '--add-config', f'producer_byte_rate={producer_bytes},consumer_byte_rate={consumer_bytes}',
            '--entity-type', 'users',
            '--entity-name', user
        ]

        result = subprocess.run(cmd, capture_output=True, text=True)
        if result.returncode != 0:
            raise Exception(f"Failed to set quota: {result.stderr}")

        print(f"Quota set for {user}: producer={producer_bytes}, consumer={consumer_bytes}")
```

## Мониторинг квот

### JMX метрики

```java
// Ключевые метрики для мониторинга throttling
kafka.server:type=Produce,user=([-.\w]+),client-id=([-.\w]+),name=throttle-time
kafka.server:type=Fetch,user=([-.\w]+),client-id=([-.\w]+),name=throttle-time
kafka.server:type=Request,user=([-.\w]+),client-id=([-.\w]+),name=throttle-time

// Метрики byte-rate
kafka.server:type=Produce,user=([-.\w]+),client-id=([-.\w]+),name=byte-rate
kafka.server:type=Fetch,user=([-.\w]+),client-id=([-.\w]+),name=byte-rate
```

### Prometheus конфигурация

```yaml
# prometheus-jmx-exporter config
rules:
  - pattern: kafka.server<type=(.+), user=(.+), client-id=(.+)><>throttle-time
    name: kafka_server_quota_throttle_time
    type: GAUGE
    labels:
      type: "$1"
      user: "$2"
      client_id: "$3"

  - pattern: kafka.server<type=(.+), user=(.+), client-id=(.+)><>byte-rate
    name: kafka_server_quota_byte_rate
    type: GAUGE
    labels:
      type: "$1"
      user: "$2"
      client_id: "$3"
```

### Алерты

```yaml
# AlertManager правила
groups:
  - name: kafka-quotas
    rules:
      - alert: KafkaClientThrottled
        expr: kafka_server_quota_throttle_time > 1000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Client {{ $labels.user }}/{{ $labels.client_id }} is being throttled"
          description: "Throttle time: {{ $value }}ms"

      - alert: KafkaClientNearQuotaLimit
        expr: kafka_server_quota_byte_rate / kafka_quota_limit > 0.8
        for: 10m
        labels:
          severity: info
        annotations:
          summary: "Client {{ $labels.user }} is at 80% of quota limit"
```

## Best Practices

### 1. Стратегия установки квот

```markdown
## Рекомендации

### Уровни квот
1. **Глобальный default** — защита от неизвестных клиентов
   - producer_byte_rate: 1 MB/s
   - consumer_byte_rate: 5 MB/s

2. **User default** — базовые лимиты для известных сервисов
   - Рассчитываются исходя из SLA сервиса

3. **Специфичные квоты** — для критичных или особых случаев
   - Высокие лимиты для batch-процессов
   - Низкие лимиты для ненадёжных клиентов
```

### 2. Расчёт квот

```python
def calculate_quotas(
    cluster_bandwidth_mbps: int,
    num_brokers: int,
    target_utilization: float,
    num_clients: int
) -> dict:
    """
    Расчёт рекомендуемых квот на основе возможностей кластера

    Args:
        cluster_bandwidth_mbps: Пропускная способность сети в Mbps
        num_brokers: Количество брокеров
        target_utilization: Целевая утилизация (0.7 = 70%)
        num_clients: Ожидаемое количество клиентов
    """
    # Доступная пропускная способность
    total_bandwidth_bytes = (cluster_bandwidth_mbps * 1024 * 1024 / 8) * target_utilization

    # Пропускная способность на клиента (с запасом 20%)
    per_client_bandwidth = (total_bandwidth_bytes / num_clients) * 0.8

    # Producers обычно потребляют меньше, чем consumers (из-за репликации)
    replication_factor = 3
    producer_quota = per_client_bandwidth / replication_factor
    consumer_quota = per_client_bandwidth

    return {
        'producer_byte_rate': int(producer_quota),
        'consumer_byte_rate': int(consumer_quota),
        'explanation': f"""
        Total cluster bandwidth: {total_bandwidth_bytes / 1024 / 1024:.2f} MB/s (at {target_utilization*100}% utilization)
        Per-client bandwidth: {per_client_bandwidth / 1024 / 1024:.2f} MB/s
        Recommended producer quota: {producer_quota / 1024 / 1024:.2f} MB/s
        Recommended consumer quota: {consumer_quota / 1024 / 1024:.2f} MB/s
        """
    }

# Пример
quotas = calculate_quotas(
    cluster_bandwidth_mbps=10000,  # 10 Gbps
    num_brokers=5,
    target_utilization=0.7,
    num_clients=50
)
print(quotas['explanation'])
```

### 3. Постепенное внедрение квот

```bash
#!/bin/bash
# Скрипт постепенного внедрения квот

# 1. Начать с мягких квот (высокие лимиты)
kafka-configs.sh --bootstrap-server kafka:9094 \
    --command-config admin.properties \
    --alter \
    --add-config 'producer_byte_rate=104857600,consumer_byte_rate=209715200' \
    --entity-type users \
    --entity-default

# 2. Мониторить метрики throttling
echo "Monitoring throttle metrics for 24 hours..."
sleep 86400

# 3. Анализ и корректировка
# - Проверить throttle-time метрики
# - Идентифицировать клиентов с высоким потреблением
# - Установить индивидуальные квоты

# 4. Постепенно уменьшать default квоты
kafka-configs.sh --bootstrap-server kafka:9094 \
    --command-config admin.properties \
    --alter \
    --add-config 'producer_byte_rate=52428800,consumer_byte_rate=104857600' \
    --entity-type users \
    --entity-default
```

### 4. Документирование квот

```yaml
# quota-config.yaml — документация квот
default_quotas:
  producer_byte_rate: 10485760  # 10 MB/s
  consumer_byte_rate: 20971520  # 20 MB/s
  request_percentage: 25         # 25% CPU

service_quotas:
  order-service:
    description: "Критичный сервис заказов"
    sla: "99.9%"
    producer_byte_rate: 52428800  # 50 MB/s
    consumer_byte_rate: 104857600 # 100 MB/s
    justification: "Высокий объём транзакций в пиковые часы"

  analytics-batch:
    description: "Batch обработка для аналитики"
    sla: "99%"
    producer_byte_rate: 10485760   # 10 MB/s
    consumer_byte_rate: 209715200  # 200 MB/s
    justification: "Потребляет большие объёмы для ETL, но не критичен по latency"
    schedule: "Ночные часы (00:00-06:00 UTC)"

  external-partner:
    description: "Внешний партнёр (ограниченное доверие)"
    producer_byte_rate: 1048576   # 1 MB/s
    consumer_byte_rate: 5242880   # 5 MB/s
    justification: "Защита от потенциального злоупотребления"
```

### 5. Реакция на throttling

```python
from kafka import KafkaProducer
from kafka.errors import KafkaError
import time

class AdaptiveProducer:
    """Producer с адаптацией к throttling"""

    def __init__(self, config):
        self.producer = KafkaProducer(**config)
        self.throttle_count = 0
        self.max_retries = 3

    def send_with_backoff(self, topic: str, value: bytes, key: bytes = None):
        """Отправка с экспоненциальной задержкой при throttling"""
        retries = 0
        backoff = 1.0

        while retries < self.max_retries:
            try:
                future = self.producer.send(topic, value=value, key=key)
                record_metadata = future.get(timeout=30)

                # Проверка throttle time в метриках (если доступно)
                if hasattr(record_metadata, 'throttle_time_ms'):
                    if record_metadata.throttle_time_ms > 0:
                        self.throttle_count += 1
                        self._log_throttling(record_metadata.throttle_time_ms)

                        # Добавить задержку
                        time.sleep(record_metadata.throttle_time_ms / 1000)

                return record_metadata

            except KafkaError as e:
                retries += 1
                if retries < self.max_retries:
                    time.sleep(backoff)
                    backoff *= 2
                else:
                    raise

    def _log_throttling(self, throttle_ms: int):
        print(f"Producer throttled for {throttle_ms}ms. "
              f"Total throttle events: {self.throttle_count}")
```

### Чеклист управления квотами

```markdown
## Checklist

### Планирование
- [ ] Определены SLA для каждого сервиса
- [ ] Рассчитаны квоты исходя из capacity planning
- [ ] Документирована стратегия квот
- [ ] Настроен мониторинг throttling

### Внедрение
- [ ] Установлены консервативные default квоты
- [ ] Критичные сервисы имеют индивидуальные квоты
- [ ] Batch-процессы имеют отдельные квоты
- [ ] Внешние клиенты ограничены

### Мониторинг
- [ ] Метрики throttle-time собираются
- [ ] Алерты на чрезмерный throttling
- [ ] Дашборды для визуализации использования квот
- [ ] Регулярный пересмотр квот (ежеквартально)
```

## Дополнительные ресурсы

- [Apache Kafka Quotas](https://kafka.apache.org/documentation/#design_quotas)
- [KIP-124 - Request Quotas](https://cwiki.apache.org/confluence/display/KAFKA/KIP-124+-+Request+rate+quotas)
- [KIP-219 - Connection Rate Quotas](https://cwiki.apache.org/confluence/display/KAFKA/KIP-219+-+Improve+connection+quota+throttling)
