# Общие инструменты мониторинга Kafka

## Описание

Для эффективного мониторинга Apache Kafka существует множество инструментов — от специализированных решений для Kafka до универсальных платформ мониторинга. Выбор инструментов зависит от масштаба инфраструктуры, бюджета, требований к функциональности и существующего технологического стека.

Правильно настроенный мониторинг позволяет оперативно выявлять проблемы, планировать ёмкость, оптимизировать производительность и обеспечивать высокую доступность кластера Kafka.

## Ключевые концепции

### Категории инструментов

| Категория | Примеры | Назначение |
|-----------|---------|------------|
| Специализированные для Kafka | AKHQ, Kafdrop, Kafka UI | Управление и визуализация |
| Универсальные платформы | Prometheus + Grafana, Datadog | Метрики и алерты |
| Коммерческие решения | Confluent Control Center, Lenses | Полный функционал |
| Облачные сервисы | AWS CloudWatch, GCP Monitoring | Managed Kafka |

### Критерии выбора

| Критерий | Вопросы для оценки |
|----------|-------------------|
| Функциональность | Какие метрики поддерживаются? |
| Масштабируемость | Справится ли с большим кластером? |
| Интеграция | Работает ли с существующим стеком? |
| Стоимость | Какова TCO решения? |
| Поддержка | Open-source или коммерческий? |

## Примеры

### Prometheus + Grafana

#### Установка через Docker Compose

```yaml
# docker-compose-monitoring.yml

version: '3.8'
services:
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./alert-rules.yml:/etc/prometheus/alert-rules.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=15d'
      - '--web.enable-lifecycle'

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./grafana/datasources:/etc/grafana/provisioning/datasources

  alertmanager:
    image: prom/alertmanager:latest
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml

  kafka-exporter:
    image: danielqsj/kafka-exporter:latest
    ports:
      - "9308:9308"
    command:
      - '--kafka.server=kafka:9092'
      - '--topic.filter=.*'
      - '--group.filter=.*'

volumes:
  prometheus_data:
  grafana_data:
```

#### Конфигурация Prometheus

```yaml
# prometheus.yml

global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: 'kafka-prod'

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

rule_files:
  - '/etc/prometheus/alert-rules.yml'

scrape_configs:
  # JMX Exporter на брокерах
  - job_name: 'kafka-broker'
    static_configs:
      - targets:
          - 'broker1:7071'
          - 'broker2:7071'
          - 'broker3:7071'
    relabel_configs:
      - source_labels: [__address__]
        target_label: broker
        regex: '([^:]+):\d+'
        replacement: '$1'

  # Kafka Exporter для consumer lag
  - job_name: 'kafka-exporter'
    static_configs:
      - targets: ['kafka-exporter:9308']

  # ZooKeeper
  - job_name: 'zookeeper'
    static_configs:
      - targets:
          - 'zk1:7072'
          - 'zk2:7072'
          - 'zk3:7072'

  # Node exporter для системных метрик
  - job_name: 'node'
    static_configs:
      - targets:
          - 'broker1:9100'
          - 'broker2:9100'
          - 'broker3:9100'
```

#### Grafana Dashboards

```json
{
  "dashboard": {
    "title": "Kafka Overview",
    "panels": [
      {
        "title": "Messages In Per Second",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(rate(kafka_server_broker_topic_metrics_messages_in_total[5m])) by (topic)",
            "legendFormat": "{{topic}}"
          }
        ]
      },
      {
        "title": "Under Replicated Partitions",
        "type": "stat",
        "targets": [
          {
            "expr": "sum(kafka_server_replica_manager_under_replicated_partitions)"
          }
        ],
        "thresholds": {
          "mode": "absolute",
          "steps": [
            {"color": "green", "value": null},
            {"color": "red", "value": 1}
          ]
        }
      },
      {
        "title": "Consumer Lag by Group",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(kafka_consumergroup_lag) by (consumergroup, topic)",
            "legendFormat": "{{consumergroup}} - {{topic}}"
          }
        ]
      }
    ]
  }
}
```

### AKHQ (Kafka HQ)

#### Установка и настройка

```yaml
# docker-compose-akhq.yml

version: '3.8'
services:
  akhq:
    image: tchiotludo/akhq:latest
    ports:
      - "8080:8080"
    environment:
      AKHQ_CONFIGURATION: |
        akhq:
          connections:
            production:
              properties:
                bootstrap.servers: "broker1:9092,broker2:9092,broker3:9092"
              schema-registry:
                url: "http://schema-registry:8081"
              connect:
                - name: "connect-cluster"
                  url: "http://kafka-connect:8083"

            staging:
              properties:
                bootstrap.servers: "staging-broker:9092"
                security.protocol: SASL_SSL
                sasl.mechanism: PLAIN
                sasl.jaas.config: >
                  org.apache.kafka.common.security.plain.PlainLoginModule required
                  username="admin"
                  password="password";

          security:
            default-group: admin
            groups:
              admin:
                - role: admin
              reader:
                - role: reader
                  patterns:
                    - "^public-.*"
            basic-auth:
              - username: admin
                password: "$2a$10$..."
                groups:
                  - admin
              - username: reader
                password: "$2a$10$..."
                groups:
                  - reader
```

#### Расширенная конфигурация AKHQ

```yaml
# application.yml для AKHQ

akhq:
  server:
    access-log:
      enabled: true
      name: access-log
      format: combined

  connections:
    production:
      properties:
        bootstrap.servers: "broker1:9092,broker2:9092"
        security.protocol: SSL
        ssl.truststore.location: /certs/truststore.jks
        ssl.truststore.password: password
        ssl.keystore.location: /certs/keystore.jks
        ssl.keystore.password: password

      deserialization:
        protobuf:
          descriptors-folder: /app/protobuf/

      ui-options:
        topic:
          default-view: HIDE_INTERNAL
          skip-consumer-groups: false
        topic-data:
          sort: NEWEST

  pagination:
    page-size: 25
    threads: 16

  topic:
    retention: 172800000

  topic-data:
    poll-timeout: 1000
    size: 50
```

### Kafdrop

```yaml
# docker-compose-kafdrop.yml

version: '3.8'
services:
  kafdrop:
    image: obsidiandynamics/kafdrop:latest
    ports:
      - "9000:9000"
    environment:
      KAFKA_BROKERCONNECT: "broker1:9092,broker2:9092,broker3:9092"
      JVM_OPTS: "-Xms32M -Xmx64M"
      SERVER_SERVLET_CONTEXTPATH: "/"
      # Для SSL
      # KAFKA_PROPERTIES: |
      #   security.protocol=SSL
      #   ssl.truststore.location=/certs/truststore.jks

    # Для Schema Registry
    environment:
      SCHEMAREGISTRY_CONNECT: "http://schema-registry:8081"
```

### Kafka UI

```yaml
# docker-compose-kafka-ui.yml

version: '3.8'
services:
  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    ports:
      - "8080:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: production
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: broker1:9092,broker2:9092,broker3:9092
      KAFKA_CLUSTERS_0_SCHEMAREGISTRY: http://schema-registry:8081
      KAFKA_CLUSTERS_0_KAFKACONNECT_0_NAME: connect-cluster
      KAFKA_CLUSTERS_0_KAFKACONNECT_0_ADDRESS: http://kafka-connect:8083

      KAFKA_CLUSTERS_1_NAME: staging
      KAFKA_CLUSTERS_1_BOOTSTRAPSERVERS: staging-broker:9092
      KAFKA_CLUSTERS_1_PROPERTIES_SECURITY_PROTOCOL: SASL_SSL
      KAFKA_CLUSTERS_1_PROPERTIES_SASL_MECHANISM: PLAIN
      KAFKA_CLUSTERS_1_PROPERTIES_SASL_JAAS_CONFIG: >
        org.apache.kafka.common.security.plain.PlainLoginModule required
        username="admin" password="password";

      # Аутентификация
      AUTH_TYPE: LOGIN_FORM
      SPRING_SECURITY_USER_NAME: admin
      SPRING_SECURITY_USER_PASSWORD: admin

      # Дополнительные настройки
      DYNAMIC_CONFIG_ENABLED: 'true'
      KAFKA_CLUSTERS_0_AUDIT_TOPICAUDITENABLED: 'true'
```

### Confluent Control Center

```yaml
# docker-compose-control-center.yml

version: '3.8'
services:
  control-center:
    image: confluentinc/cp-enterprise-control-center:latest
    ports:
      - "9021:9021"
    environment:
      CONTROL_CENTER_BOOTSTRAP_SERVERS: 'broker1:9092,broker2:9092,broker3:9092'
      CONTROL_CENTER_ZOOKEEPER_CONNECT: 'zk1:2181,zk2:2181,zk3:2181'
      CONTROL_CENTER_SCHEMA_REGISTRY_URL: 'http://schema-registry:8081'
      CONTROL_CENTER_CONNECT_CLUSTER: 'http://kafka-connect:8083'
      CONTROL_CENTER_KSQL_URL: 'http://ksqldb-server:8088'
      CONTROL_CENTER_KSQL_ADVERTISED_URL: 'http://localhost:8088'
      CONTROL_CENTER_REPLICATION_FACTOR: 3
      CONTROL_CENTER_INTERNAL_TOPICS_PARTITIONS: 3
      CONTROL_CENTER_MONITORING_INTERCEPTOR_TOPIC_PARTITIONS: 3
      CONFLUENT_METRICS_TOPIC_REPLICATION: 3
      PORT: 9021
```

### Datadog

```yaml
# datadog-agent.yaml для Kubernetes

apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: datadog-agent
spec:
  selector:
    matchLabels:
      app: datadog-agent
  template:
    metadata:
      labels:
        app: datadog-agent
    spec:
      containers:
        - name: datadog-agent
          image: datadog/agent:latest
          env:
            - name: DD_API_KEY
              valueFrom:
                secretKeyRef:
                  name: datadog-secret
                  key: api-key
            - name: DD_LOGS_ENABLED
              value: "true"
            - name: DD_LOGS_CONFIG_CONTAINER_COLLECT_ALL
              value: "true"
          volumeMounts:
            - name: kafka-config
              mountPath: /etc/datadog-agent/conf.d/kafka.d/
      volumes:
        - name: kafka-config
          configMap:
            name: datadog-kafka-config

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: datadog-kafka-config
data:
  conf.yaml: |
    init_config:
    instances:
      - host: kafka-broker
        port: 9999
        tags:
          - env:production
          - service:kafka
    logs:
      - type: file
        path: /var/log/kafka/server.log
        source: kafka
        service: kafka-broker
```

### Burrow (LinkedIn)

```yaml
# docker-compose-burrow.yml

version: '3.8'
services:
  burrow:
    image: linkedin/burrow:latest
    ports:
      - "8000:8000"
    volumes:
      - ./burrow.toml:/etc/burrow/burrow.toml

---
# burrow.toml
[general]
pidfile = "/var/run/burrow.pid"
stdout-logfile = "/var/log/burrow.log"
access-control-allow-origin = "*"

[logging]
filename = "/var/log/burrow/burrow.log"
level = "info"

[zookeeper]
servers = ["zk1:2181", "zk2:2181", "zk3:2181"]
timeout = 6
root-path = "/burrow"

[client-profile.kafka-profile]
kafka-version = "2.8.0"
client-id = "burrow-client"

[cluster.production]
class-name = "kafka"
servers = ["broker1:9092", "broker2:9092", "broker3:9092"]
client-profile = "kafka-profile"
topic-refresh = 60
offset-refresh = 30

[consumer.production]
class-name = "kafka"
cluster = "production"
servers = ["broker1:9092", "broker2:9092", "broker3:9092"]
client-profile = "kafka-profile"
group-denylist = "^console-.*$"
start-latest = true

[httpserver.default]
address = ":8000"

[storage.default]
class-name = "inmemory"
workers = 20
intervals = 15
expire-group = 604800
min-distance = 1

[evaluator.default]
class-name = "caching"
expire-cache = 10

[notifier.default]
class-name = "http"
url-open = "http://alertmanager:9093/api/v1/alerts"
template-open = "/etc/burrow/alert-open.tmpl"
send-close = false
```

### Cruise Control (LinkedIn)

```yaml
# docker-compose-cruise-control.yml

version: '3.8'
services:
  cruise-control:
    image: linkedin/cruise-control:latest
    ports:
      - "9090:9090"
    volumes:
      - ./cruisecontrol.properties:/opt/cruise-control/config/cruisecontrol.properties
    environment:
      KAFKA_HEAP_OPTS: "-Xmx2G"

---
# cruisecontrol.properties
bootstrap.servers=broker1:9092,broker2:9092,broker3:9092
zookeeper.connect=zk1:2181,zk2:2181,zk3:2181

# Metric reporter
metric.reporters=com.linkedin.kafka.cruisecontrol.metricsreporter.CruiseControlMetricsReporter
cruise.control.metrics.reporter.bootstrap.servers=broker1:9092,broker2:9092,broker3:9092

# Capacity settings
capacity.config.file=/opt/cruise-control/config/capacity.json
broker.capacity.config.resolver.class=com.linkedin.kafka.cruisecontrol.config.JsonFileCapacityConfigResolver

# Goals
default.goals=com.linkedin.kafka.cruisecontrol.analyzer.goals.RackAwareGoal,\
  com.linkedin.kafka.cruisecontrol.analyzer.goals.ReplicaCapacityGoal,\
  com.linkedin.kafka.cruisecontrol.analyzer.goals.DiskCapacityGoal,\
  com.linkedin.kafka.cruisecontrol.analyzer.goals.NetworkInboundCapacityGoal,\
  com.linkedin.kafka.cruisecontrol.analyzer.goals.NetworkOutboundCapacityGoal,\
  com.linkedin.kafka.cruisecontrol.analyzer.goals.CpuCapacityGoal

# Anomaly detection
anomaly.detection.goals=com.linkedin.kafka.cruisecontrol.analyzer.goals.RackAwareGoal,\
  com.linkedin.kafka.cruisecontrol.analyzer.goals.ReplicaCapacityGoal

webserver.http.port=9090
webserver.api.urlprefix=/kafkacruisecontrol/*
```

### Скрипт сравнения инструментов

```bash
#!/bin/bash
# compare-monitoring-tools.sh

echo "=== Kafka Monitoring Tools Comparison ==="
echo

declare -A TOOLS

TOOLS["Prometheus+Grafana"]="open-source,metrics,alerting,dashboards"
TOOLS["AKHQ"]="open-source,ui,topic-management,consumer-groups"
TOOLS["Kafdrop"]="open-source,ui,simple,lightweight"
TOOLS["Kafka UI"]="open-source,ui,modern,feature-rich"
TOOLS["Control Center"]="commercial,enterprise,full-featured,expensive"
TOOLS["Datadog"]="saas,apm,logs,metrics"
TOOLS["Burrow"]="open-source,consumer-lag,alerting"
TOOLS["Cruise Control"]="open-source,auto-rebalance,capacity-planning"

echo "Tool | Type | Key Features"
echo "------|------|-------------"
for tool in "${!TOOLS[@]}"; do
    echo "$tool | ${TOOLS[$tool]}"
done
```

## Best Practices

### Рекомендации по выбору

1. **Для небольших команд**: Kafka UI или Kafdrop + Prometheus + Grafana
2. **Для enterprise**: Confluent Control Center или Lenses
3. **Для облачных решений**: Нативные инструменты облачного провайдера
4. **Для DevOps-культуры**: Prometheus + Grafana + PagerDuty/Opsgenie

### Минимальный набор инструментов

```yaml
Обязательно:
  - Сбор метрик (Prometheus/Datadog)
  - Визуализация (Grafana/встроенные дашборды)
  - Алертинг (Alertmanager/PagerDuty)
  - Consumer lag мониторинг (Kafka Exporter/Burrow)

Рекомендуется:
  - UI для управления (AKHQ/Kafka UI)
  - Log aggregation (ELK/Loki)
  - Distributed tracing (Jaeger/Zipkin)
```

### Архитектура мониторинга

```
┌─────────────────────────────────────────────────────────┐
│                    Kafka Cluster                        │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐                │
│  │ Broker 1│  │ Broker 2│  │ Broker 3│                │
│  │ JMX:9999│  │ JMX:9999│  │ JMX:9999│                │
│  └────┬────┘  └────┬────┘  └────┬────┘                │
└───────┼────────────┼────────────┼──────────────────────┘
        │            │            │
        └────────────┼────────────┘
                     │
              ┌──────┴──────┐
              │ JMX Exporter│
              │   :7071     │
              └──────┬──────┘
                     │
        ┌────────────┼────────────┐
        │            │            │
   ┌────┴────┐ ┌─────┴─────┐ ┌────┴────┐
   │Prometheus│ │Kafka     │ │Alertmgr │
   │  :9090   │ │Exporter  │ │  :9093  │
   └────┬────┘ │  :9308    │ └────┬────┘
        │      └───────────┘      │
        │                         │
   ┌────┴────┐               ┌────┴────┐
   │ Grafana │               │PagerDuty│
   │  :3000  │               │ Slack   │
   └─────────┘               └─────────┘
```

### Чек-лист настройки мониторинга

```markdown
- [ ] JMX включен на всех брокерах
- [ ] JMX Exporter развёрнут и настроен
- [ ] Prometheus собирает метрики
- [ ] Grafana dashboards настроены
- [ ] Kafka Exporter для consumer lag
- [ ] Alertmanager настроен
- [ ] Алерты отправляются в нужные каналы
- [ ] UI для управления развёрнут
- [ ] Документация по on-call обновлена
```
