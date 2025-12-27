# Брандмауэры и сетевая безопасность Kafka

## Описание

Настройка брандмауэров (firewalls) является критически важной частью безопасности кластера Apache Kafka. Правильная конфигурация сетевых правил защищает брокеры от несанкционированного доступа, ограничивает трафик только необходимыми портами и обеспечивает изоляцию между различными компонентами системы.

Kafka использует несколько портов для различных целей: коммуникация между брокерами, подключение клиентов, управление через JMX и взаимодействие с ZooKeeper.

## Ключевые концепции

### Порты Kafka

| Порт | Назначение | Протокол |
|------|------------|----------|
| 9092 | Клиентские подключения (PLAINTEXT) | TCP |
| 9093 | Клиентские подключения (SSL) | TCP |
| 9094 | Клиентские подключения (SASL_PLAINTEXT) | TCP |
| 9095 | Клиентские подключения (SASL_SSL) | TCP |
| 9999 | JMX мониторинг | TCP |
| 2181 | ZooKeeper (клиентский порт) | TCP |
| 2888 | ZooKeeper (peer-to-peer) | TCP |
| 3888 | ZooKeeper (leader election) | TCP |

### Типы трафика

| Тип трафика | Направление | Описание |
|-------------|-------------|----------|
| Client-to-Broker | Входящий | Продюсеры и консьюмеры |
| Broker-to-Broker | Двусторонний | Репликация данных |
| Broker-to-ZooKeeper | Исходящий | Управление метаданными |
| Controller-to-Broker | Двусторонний | Управление кластером |
| Admin-to-Broker | Входящий | Административные операции |

## Примеры

### Настройка iptables для Kafka

```bash
#!/bin/bash
# kafka-iptables.sh - Настройка iptables для Kafka брокера

# Очистка существующих правил
iptables -F
iptables -X

# Политики по умолчанию
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# Разрешаем loopback
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT

# Разрешаем установленные соединения
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# SSH доступ (ограничен определёнными IP)
iptables -A INPUT -p tcp --dport 22 -s 10.0.0.0/8 -j ACCEPT

# Kafka клиентские подключения (PLAINTEXT)
iptables -A INPUT -p tcp --dport 9092 -s 10.0.0.0/8 -j ACCEPT

# Kafka SSL подключения
iptables -A INPUT -p tcp --dport 9093 -s 10.0.0.0/8 -j ACCEPT

# Kafka SASL_SSL подключения
iptables -A INPUT -p tcp --dport 9095 -s 10.0.0.0/8 -j ACCEPT

# Inter-broker communication (только между брокерами)
iptables -A INPUT -p tcp --dport 9092 -s 192.168.1.10 -j ACCEPT
iptables -A INPUT -p tcp --dport 9092 -s 192.168.1.11 -j ACCEPT
iptables -A INPUT -p tcp --dport 9092 -s 192.168.1.12 -j ACCEPT

# JMX мониторинг (только с серверов мониторинга)
iptables -A INPUT -p tcp --dport 9999 -s 10.0.1.100 -j ACCEPT

# ZooKeeper (только между узлами ZK и брокерами)
iptables -A INPUT -p tcp --dport 2181 -s 192.168.1.0/24 -j ACCEPT
iptables -A INPUT -p tcp --dport 2888 -s 192.168.1.20 -j ACCEPT
iptables -A INPUT -p tcp --dport 2888 -s 192.168.1.21 -j ACCEPT
iptables -A INPUT -p tcp --dport 2888 -s 192.168.1.22 -j ACCEPT
iptables -A INPUT -p tcp --dport 3888 -s 192.168.1.20 -j ACCEPT
iptables -A INPUT -p tcp --dport 3888 -s 192.168.1.21 -j ACCEPT
iptables -A INPUT -p tcp --dport 3888 -s 192.168.1.22 -j ACCEPT

# ICMP (ping)
iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT

# Логирование отклонённых пакетов
iptables -A INPUT -j LOG --log-prefix "IPTABLES-DROPPED: " --log-level 4

# Сохранение правил
iptables-save > /etc/iptables/rules.v4

echo "Firewall rules applied successfully"
```

### Настройка firewalld (RHEL/CentOS)

```bash
#!/bin/bash
# kafka-firewalld.sh - Настройка firewalld для Kafka

# Создание зоны для Kafka
firewall-cmd --permanent --new-zone=kafka

# Добавление источников (IP-адреса клиентов)
firewall-cmd --permanent --zone=kafka --add-source=10.0.0.0/8

# Добавление портов
firewall-cmd --permanent --zone=kafka --add-port=9092/tcp  # PLAINTEXT
firewall-cmd --permanent --zone=kafka --add-port=9093/tcp  # SSL
firewall-cmd --permanent --zone=kafka --add-port=9094/tcp  # SASL_PLAINTEXT
firewall-cmd --permanent --zone=kafka --add-port=9095/tcp  # SASL_SSL

# Создание зоны для межброкерной коммуникации
firewall-cmd --permanent --new-zone=kafka-internal

# Добавление IP-адресов других брокеров
firewall-cmd --permanent --zone=kafka-internal --add-source=192.168.1.10
firewall-cmd --permanent --zone=kafka-internal --add-source=192.168.1.11
firewall-cmd --permanent --zone=kafka-internal --add-source=192.168.1.12

firewall-cmd --permanent --zone=kafka-internal --add-port=9092/tcp

# Зона для мониторинга
firewall-cmd --permanent --new-zone=kafka-monitoring
firewall-cmd --permanent --zone=kafka-monitoring --add-source=10.0.1.100
firewall-cmd --permanent --zone=kafka-monitoring --add-port=9999/tcp

# Зона для ZooKeeper
firewall-cmd --permanent --new-zone=zookeeper
firewall-cmd --permanent --zone=zookeeper --add-source=192.168.1.0/24
firewall-cmd --permanent --zone=zookeeper --add-port=2181/tcp
firewall-cmd --permanent --zone=zookeeper --add-port=2888/tcp
firewall-cmd --permanent --zone=zookeeper --add-port=3888/tcp

# Применение настроек
firewall-cmd --reload

# Проверка
firewall-cmd --list-all-zones
```

### Создание сервиса firewalld для Kafka

```xml
<!-- /etc/firewalld/services/kafka.xml -->
<?xml version="1.0" encoding="utf-8"?>
<service>
  <short>Kafka</short>
  <description>Apache Kafka Message Broker</description>
  <port protocol="tcp" port="9092"/>
  <port protocol="tcp" port="9093"/>
  <port protocol="tcp" port="9094"/>
  <port protocol="tcp" port="9095"/>
</service>
```

```xml
<!-- /etc/firewalld/services/kafka-jmx.xml -->
<?xml version="1.0" encoding="utf-8"?>
<service>
  <short>Kafka JMX</short>
  <description>Kafka JMX Monitoring</description>
  <port protocol="tcp" port="9999"/>
</service>
```

```xml
<!-- /etc/firewalld/services/zookeeper.xml -->
<?xml version="1.0" encoding="utf-8"?>
<service>
  <short>ZooKeeper</short>
  <description>Apache ZooKeeper</description>
  <port protocol="tcp" port="2181"/>
  <port protocol="tcp" port="2888"/>
  <port protocol="tcp" port="3888"/>
</service>
```

### Настройка UFW (Ubuntu)

```bash
#!/bin/bash
# kafka-ufw.sh - Настройка UFW для Kafka

# Сброс правил
ufw reset

# Политики по умолчанию
ufw default deny incoming
ufw default allow outgoing

# SSH
ufw allow from 10.0.0.0/8 to any port 22 proto tcp

# Kafka клиентские подключения
ufw allow from 10.0.0.0/8 to any port 9092 proto tcp comment 'Kafka PLAINTEXT'
ufw allow from 10.0.0.0/8 to any port 9093 proto tcp comment 'Kafka SSL'
ufw allow from 10.0.0.0/8 to any port 9095 proto tcp comment 'Kafka SASL_SSL'

# Inter-broker communication
ufw allow from 192.168.1.10 to any port 9092 proto tcp comment 'Kafka broker 1'
ufw allow from 192.168.1.11 to any port 9092 proto tcp comment 'Kafka broker 2'
ufw allow from 192.168.1.12 to any port 9092 proto tcp comment 'Kafka broker 3'

# JMX мониторинг
ufw allow from 10.0.1.100 to any port 9999 proto tcp comment 'Kafka JMX'

# ZooKeeper
ufw allow from 192.168.1.0/24 to any port 2181 proto tcp comment 'ZooKeeper client'
ufw allow from 192.168.1.20 to any port 2888 proto tcp comment 'ZooKeeper peer'
ufw allow from 192.168.1.21 to any port 2888 proto tcp comment 'ZooKeeper peer'
ufw allow from 192.168.1.22 to any port 2888 proto tcp comment 'ZooKeeper peer'
ufw allow from 192.168.1.20 to any port 3888 proto tcp comment 'ZooKeeper election'
ufw allow from 192.168.1.21 to any port 3888 proto tcp comment 'ZooKeeper election'
ufw allow from 192.168.1.22 to any port 3888 proto tcp comment 'ZooKeeper election'

# Включение UFW
ufw enable

# Проверка статуса
ufw status verbose
```

### Создание профиля приложения UFW

```ini
# /etc/ufw/applications.d/kafka

[Kafka]
title=Apache Kafka Message Broker
description=Kafka broker ports for client connections
ports=9092,9093,9094,9095/tcp

[Kafka-Internal]
title=Kafka Internal Communication
description=Kafka inter-broker communication
ports=9092/tcp

[Kafka-JMX]
title=Kafka JMX Monitoring
description=Kafka JMX port for monitoring
ports=9999/tcp

[ZooKeeper]
title=Apache ZooKeeper
description=ZooKeeper coordination service
ports=2181,2888,3888/tcp
```

```bash
# Применение профиля
ufw allow from 10.0.0.0/8 to any app Kafka
ufw allow from 10.0.1.100 to any app Kafka-JMX
```

### Настройка nftables

```bash
#!/usr/sbin/nft -f
# /etc/nftables.conf - Kafka firewall rules

table inet filter {
    chain input {
        type filter hook input priority 0; policy drop;

        # Loopback
        iif lo accept

        # Established connections
        ct state established,related accept

        # ICMP
        ip protocol icmp accept
        ip6 nexthdr icmpv6 accept

        # SSH (from admin network)
        tcp dport 22 ip saddr 10.0.0.0/8 accept

        # Kafka client connections
        tcp dport 9092 ip saddr 10.0.0.0/8 accept comment "Kafka PLAINTEXT"
        tcp dport 9093 ip saddr 10.0.0.0/8 accept comment "Kafka SSL"
        tcp dport 9095 ip saddr 10.0.0.0/8 accept comment "Kafka SASL_SSL"

        # Inter-broker communication
        tcp dport 9092 ip saddr { 192.168.1.10, 192.168.1.11, 192.168.1.12 } accept comment "Kafka inter-broker"

        # JMX monitoring
        tcp dport 9999 ip saddr 10.0.1.100 accept comment "Kafka JMX"

        # ZooKeeper
        tcp dport 2181 ip saddr 192.168.1.0/24 accept comment "ZooKeeper client"
        tcp dport { 2888, 3888 } ip saddr { 192.168.1.20, 192.168.1.21, 192.168.1.22 } accept comment "ZooKeeper cluster"

        # Log dropped packets
        log prefix "nftables-dropped: " flags all counter drop
    }

    chain forward {
        type filter hook forward priority 0; policy drop;
    }

    chain output {
        type filter hook output priority 0; policy accept;
    }
}
```

### AWS Security Groups

```hcl
# terraform/kafka-security-groups.tf

# Security Group для Kafka брокеров
resource "aws_security_group" "kafka_broker" {
  name        = "kafka-broker-sg"
  description = "Security group for Kafka brokers"
  vpc_id      = var.vpc_id

  # SSH доступ
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.admin_cidr]
    description = "SSH access"
  }

  # Kafka PLAINTEXT
  ingress {
    from_port       = 9092
    to_port         = 9092
    protocol        = "tcp"
    security_groups = [aws_security_group.kafka_client.id]
    description     = "Kafka PLAINTEXT client access"
  }

  # Kafka SSL
  ingress {
    from_port       = 9093
    to_port         = 9093
    protocol        = "tcp"
    security_groups = [aws_security_group.kafka_client.id]
    description     = "Kafka SSL client access"
  }

  # Inter-broker communication
  ingress {
    from_port = 9092
    to_port   = 9092
    protocol  = "tcp"
    self      = true
    description = "Kafka inter-broker communication"
  }

  # JMX monitoring
  ingress {
    from_port       = 9999
    to_port         = 9999
    protocol        = "tcp"
    security_groups = [aws_security_group.monitoring.id]
    description     = "JMX monitoring access"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "Allow all outbound"
  }

  tags = {
    Name = "kafka-broker-sg"
  }
}

# Security Group для ZooKeeper
resource "aws_security_group" "zookeeper" {
  name        = "zookeeper-sg"
  description = "Security group for ZooKeeper"
  vpc_id      = var.vpc_id

  # ZooKeeper client port (from Kafka brokers)
  ingress {
    from_port       = 2181
    to_port         = 2181
    protocol        = "tcp"
    security_groups = [aws_security_group.kafka_broker.id]
    description     = "ZooKeeper client access from Kafka"
  }

  # ZooKeeper peer communication
  ingress {
    from_port   = 2888
    to_port     = 2888
    protocol    = "tcp"
    self        = true
    description = "ZooKeeper peer communication"
  }

  # ZooKeeper leader election
  ingress {
    from_port   = 3888
    to_port     = 3888
    protocol    = "tcp"
    self        = true
    description = "ZooKeeper leader election"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "zookeeper-sg"
  }
}

# Security Group для клиентов Kafka
resource "aws_security_group" "kafka_client" {
  name        = "kafka-client-sg"
  description = "Security group for Kafka clients"
  vpc_id      = var.vpc_id

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "kafka-client-sg"
  }
}
```

### Kubernetes Network Policies

```yaml
# kafka-network-policy.yaml

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: kafka-broker-policy
  namespace: kafka
spec:
  podSelector:
    matchLabels:
      app: kafka
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # Клиентские подключения
    - from:
        - namespaceSelector:
            matchLabels:
              kafka-access: "true"
        - podSelector:
            matchLabels:
              kafka-client: "true"
      ports:
        - protocol: TCP
          port: 9092
        - protocol: TCP
          port: 9093

    # Inter-broker communication
    - from:
        - podSelector:
            matchLabels:
              app: kafka
      ports:
        - protocol: TCP
          port: 9092

    # JMX мониторинг
    - from:
        - namespaceSelector:
            matchLabels:
              name: monitoring
        - podSelector:
            matchLabels:
              app: prometheus
      ports:
        - protocol: TCP
          port: 9999

  egress:
    # ZooKeeper
    - to:
        - podSelector:
            matchLabels:
              app: zookeeper
      ports:
        - protocol: TCP
          port: 2181

    # Inter-broker
    - to:
        - podSelector:
            matchLabels:
              app: kafka
      ports:
        - protocol: TCP
          port: 9092

    # DNS
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: zookeeper-policy
  namespace: kafka
spec:
  podSelector:
    matchLabels:
      app: zookeeper
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # Client connections from Kafka
    - from:
        - podSelector:
            matchLabels:
              app: kafka
      ports:
        - protocol: TCP
          port: 2181

    # ZooKeeper cluster communication
    - from:
        - podSelector:
            matchLabels:
              app: zookeeper
      ports:
        - protocol: TCP
          port: 2888
        - protocol: TCP
          port: 3888

  egress:
    # ZooKeeper cluster
    - to:
        - podSelector:
            matchLabels:
              app: zookeeper
      ports:
        - protocol: TCP
          port: 2888
        - protocol: TCP
          port: 3888

    # DNS
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
```

### Скрипт проверки портов

```bash
#!/bin/bash
# check-kafka-ports.sh

BROKER_HOST=${1:-localhost}
PORTS=(9092 9093 9094 9095 9999 2181)

echo "=== Checking Kafka ports on $BROKER_HOST ==="

for port in "${PORTS[@]}"; do
    if nc -z -w2 "$BROKER_HOST" "$port" 2>/dev/null; then
        echo "[OK] Port $port is open"
    else
        echo "[FAIL] Port $port is closed or unreachable"
    fi
done

echo
echo "=== Checking connectivity from this host ==="

# Проверка исходящих соединений
for port in "${PORTS[@]}"; do
    timeout 2 bash -c "echo >/dev/tcp/$BROKER_HOST/$port" 2>/dev/null && \
        echo "[OK] Can connect to $BROKER_HOST:$port" || \
        echo "[FAIL] Cannot connect to $BROKER_HOST:$port"
done
```

## Best Practices

### Принципы безопасности

1. **Принцип минимальных привилегий** — открывайте только необходимые порты
2. **Сегментация сети** — разделяйте клиентский и административный трафик
3. **Defense in depth** — используйте несколько уровней защиты
4. **Логирование** — включите логирование отклонённых пакетов

### Рекомендации

```bash
# Разделение listeners в server.properties
listeners=INTERNAL://0.0.0.0:9092,EXTERNAL://0.0.0.0:9093
advertised.listeners=INTERNAL://broker1.internal:9092,EXTERNAL://broker1.public:9093
listener.security.protocol.map=INTERNAL:PLAINTEXT,EXTERNAL:SSL
inter.broker.listener.name=INTERNAL
```

### Мониторинг сетевой активности

```bash
# Просмотр активных соединений
ss -tuln | grep -E '9092|9093|2181'

# Мониторинг в реальном времени
watch -n1 'ss -tuln | grep -E "9092|9093|2181"'

# Анализ трафика
tcpdump -i any port 9092 -c 100
```

### Регулярный аудит

```bash
# Проверка правил firewall
iptables -L -n -v
firewall-cmd --list-all
ufw status numbered

# Сканирование открытых портов
nmap -sT -p 9092,9093,9094,9095,9999,2181 broker-host
```
