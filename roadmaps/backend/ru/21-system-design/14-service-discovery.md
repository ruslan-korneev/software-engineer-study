# Service Discovery

[prev: 13-microservices](./13-microservices.md) | [next: 15-api-gateway](./15-api-gateway.md)

---

## Что такое Service Discovery?

**Service Discovery** (обнаружение сервисов) — это механизм автоматического определения сетевого расположения экземпляров сервисов в распределённой системе. В микросервисной архитектуре сервисы должны находить друг друга для взаимодействия, и Service Discovery решает эту задачу.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Service Discovery                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Клиент                    Service Registry                     │
│   ┌─────┐                   ┌─────────────┐                      │
│   │     │ ─── Запрос ──────>│  Service A  │                      │
│   │ App │     "Где Service A?" │ 192.168.1.10:8080│              │
│   │     │<── Ответ ─────────│ 192.168.1.11:8080│                │
│   └─────┘                   │ 192.168.1.12:8080│                │
│      │                      └─────────────┘                      │
│      │                                                           │
│      └──────────────────────────────────────────>  Service A     │
│                  Прямой вызов                      Instance      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Зачем нужен Service Discovery?

### Проблемы без Service Discovery

В традиционных монолитных приложениях адреса сервисов статичны и могут быть захардкожены. В микросервисной архитектуре это невозможно по ряду причин:

1. **Динамические IP-адреса**: контейнеры и виртуальные машины получают IP при запуске
2. **Масштабирование**: количество экземпляров сервиса постоянно меняется
3. **Отказы**: экземпляры падают и заменяются новыми
4. **Развёртывание**: новые версии сервисов запускаются на других адресах

```python
# Плохо: статическая конфигурация
class OrderService:
    def __init__(self):
        # Что если адрес изменится? Что если нужно масштабирование?
        self.payment_service_url = "http://192.168.1.100:8080"
        self.inventory_service_url = "http://192.168.1.101:8080"

    def create_order(self, order):
        # Если сервис недоступен — ошибка
        response = requests.post(f"{self.payment_service_url}/pay", json=order)
        return response.json()
```

```python
# Хорошо: использование Service Discovery
class OrderService:
    def __init__(self, service_registry):
        self.registry = service_registry

    def create_order(self, order):
        # Динамическое получение адреса здорового экземпляра
        payment_instance = self.registry.get_instance("payment-service")
        response = requests.post(f"{payment_instance.url}/pay", json=order)
        return response.json()
```

### Преимущества Service Discovery

| Преимущество | Описание |
|--------------|----------|
| **Динамичность** | Автоматическое обновление списка доступных сервисов |
| **Масштабируемость** | Легкое добавление новых экземпляров |
| **Отказоустойчивость** | Автоматическое исключение неработающих экземпляров |
| **Гибкость** | Поддержка разных окружений (dev, staging, prod) |
| **Load Balancing** | Распределение нагрузки между экземплярами |

---

## Паттерны Service Discovery

Существует два основных паттерна реализации Service Discovery:

### 1. Client-Side Discovery

При client-side discovery клиент сам отвечает за определение местоположения сервиса и балансировку нагрузки.

```
┌────────────────────────────────────────────────────────────┐
│                  Client-Side Discovery                      │
├────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌──────────────┐         ┌─────────────────┐              │
│   │   Client     │ ──1──>  │ Service Registry │             │
│   │  + Library   │ <──2──  │                  │             │
│   │  (Ribbon)    │         │  service-a:      │             │
│   └──────────────┘         │   - 10.0.0.1:8080│             │
│          │                 │   - 10.0.0.2:8080│             │
│          │                 │   - 10.0.0.3:8080│             │
│          │                 └─────────────────┘              │
│          │                                                   │
│          │ 3. Прямой вызов (с балансировкой)                │
│          │                                                   │
│          ├─────────────────>  Service A (10.0.0.1)          │
│          ├─────────────────>  Service A (10.0.0.2)          │
│          └─────────────────>  Service A (10.0.0.3)          │
│                                                             │
└────────────────────────────────────────────────────────────┘

1. Клиент запрашивает список инстансов у Registry
2. Registry возвращает список доступных адресов
3. Клиент сам выбирает инстанс и делает запрос
```

**Пример с Netflix Ribbon (Java):**

```java
@Configuration
public class RibbonConfig {

    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}

@Service
public class OrderService {

    @Autowired
    private RestTemplate restTemplate;

    public Payment processPayment(Order order) {
        // "payment-service" — имя сервиса в реестре
        // Ribbon сам найдёт инстанс и сбалансирует нагрузку
        return restTemplate.postForObject(
            "http://payment-service/api/payments",
            order,
            Payment.class
        );
    }
}
```

**Пример на Python с Consul:**

```python
import consul
import random

class ServiceDiscoveryClient:
    def __init__(self, consul_host='localhost', consul_port=8500):
        self.consul = consul.Consul(host=consul_host, port=consul_port)
        self._cache = {}
        self._cache_ttl = 30  # секунды

    def get_service_instances(self, service_name):
        """Получить все здоровые экземпляры сервиса"""
        _, services = self.consul.health.service(service_name, passing=True)

        instances = []
        for service in services:
            address = service['Service']['Address']
            port = service['Service']['Port']
            instances.append(f"http://{address}:{port}")

        return instances

    def get_instance(self, service_name, strategy='random'):
        """Получить один экземпляр с балансировкой"""
        instances = self.get_service_instances(service_name)

        if not instances:
            raise Exception(f"No healthy instances for {service_name}")

        if strategy == 'random':
            return random.choice(instances)
        elif strategy == 'round_robin':
            # Реализация round-robin
            return self._round_robin(service_name, instances)

        return instances[0]

    def _round_robin(self, service_name, instances):
        if service_name not in self._cache:
            self._cache[service_name] = 0

        index = self._cache[service_name] % len(instances)
        self._cache[service_name] += 1

        return instances[index]


# Использование
discovery = ServiceDiscoveryClient()

class PaymentClient:
    def __init__(self, discovery):
        self.discovery = discovery

    def process_payment(self, order_data):
        url = self.discovery.get_instance('payment-service')
        response = requests.post(f"{url}/api/payments", json=order_data)
        return response.json()
```

**Плюсы Client-Side Discovery:**
- Меньше сетевых хопов (прямое соединение)
- Клиент может реализовать умную балансировку
- Нет единой точки отказа (кроме реестра)

**Минусы:**
- Клиент усложняется логикой обнаружения
- Нужна библиотека для каждого языка
- Клиент должен знать о существовании реестра

---

### 2. Server-Side Discovery

При server-side discovery клиент делает запрос через промежуточный компонент (load balancer или API Gateway), который сам определяет адрес сервиса.

```
┌────────────────────────────────────────────────────────────┐
│                  Server-Side Discovery                      │
├────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌────────┐         ┌──────────────┐                       │
│   │ Client │ ──1──>  │ Load Balancer│                       │
│   │        │ <──4──  │ / API Gateway│                       │
│   └────────┘         └──────────────┘                       │
│                            │  ^                              │
│                         2  │  │ 3                            │
│                            v  │                              │
│                      ┌─────────────────┐                    │
│                      │ Service Registry │                   │
│                      └─────────────────┘                    │
│                                                             │
│                        Service A                            │
│                     ┌─────────────────┐                     │
│                     │  ┌───┐ ┌───┐    │                     │
│                     │  │ 1 │ │ 2 │ ...│                     │
│                     │  └───┘ └───┘    │                     │
│                     └─────────────────┘                     │
│                                                             │
└────────────────────────────────────────────────────────────┘

1. Клиент отправляет запрос на Load Balancer
2. LB запрашивает адреса сервиса у Registry
3. Registry возвращает список инстансов
4. LB проксирует запрос и возвращает ответ
```

**Пример конфигурации nginx с Consul Template:**

```nginx
# nginx.conf (генерируется Consul Template)
upstream payment-service {
    {{range service "payment-service"}}
    server {{.Address}}:{{.Port}} weight=1;
    {{end}}
}

server {
    listen 80;

    location /api/payments {
        proxy_pass http://payment-service;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

**Consul Template конфигурация:**

```hcl
# consul-template.hcl
template {
  source      = "/etc/nginx/nginx.conf.tmpl"
  destination = "/etc/nginx/nginx.conf"
  command     = "nginx -s reload"
}
```

**Пример с AWS ALB и ECS:**

```yaml
# AWS ECS Task Definition с Service Discovery
AWSTemplateFormatVersion: '2010-09-09'

Resources:
  ServiceDiscoveryNamespace:
    Type: AWS::ServiceDiscovery::PrivateDnsNamespace
    Properties:
      Name: production.local
      Vpc: !Ref VPC

  PaymentServiceDiscovery:
    Type: AWS::ServiceDiscovery::Service
    Properties:
      Name: payment-service
      NamespaceId: !Ref ServiceDiscoveryNamespace
      DnsConfig:
        DnsRecords:
          - Type: A
            TTL: 60

  PaymentService:
    Type: AWS::ECS::Service
    Properties:
      ServiceName: payment-service
      ServiceRegistries:
        - RegistryArn: !GetAtt PaymentServiceDiscovery.Arn
      LoadBalancers:
        - TargetGroupArn: !Ref PaymentTargetGroup
          ContainerName: payment
          ContainerPort: 8080
```

**Плюсы Server-Side Discovery:**
- Клиент простой (не знает о реестре)
- Язык-агностик (работает с любым клиентом)
- Централизованное управление балансировкой

**Минусы:**
- Дополнительный сетевой хоп
- Load Balancer может стать узким местом
- Нужно обеспечить HA для Load Balancer

---

## Service Registry (Реестр сервисов)

Service Registry — это база данных, содержащая информацию о сетевых адресах экземпляров сервисов. Это ключевой компонент системы Service Discovery.

### Требования к Service Registry

```
┌─────────────────────────────────────────────────────────────┐
│                    Service Registry                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ service-name: payment-service                        │    │
│  │ instances:                                           │    │
│  │   - id: payment-1                                    │    │
│  │     address: 10.0.1.10                               │    │
│  │     port: 8080                                       │    │
│  │     status: healthy                                  │    │
│  │     metadata:                                        │    │
│  │       version: 2.1.0                                 │    │
│  │       zone: us-east-1a                               │    │
│  │     last_heartbeat: 2024-01-15T10:30:00Z            │    │
│  │                                                      │    │
│  │   - id: payment-2                                    │    │
│  │     address: 10.0.1.11                               │    │
│  │     port: 8080                                       │    │
│  │     status: healthy                                  │    │
│  │     ...                                              │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  Требования:                                                 │
│  ✓ Высокая доступность (HA)                                 │
│  ✓ Консистентность данных                                   │
│  ✓ Низкая задержка запросов                                 │
│  ✓ Поддержка health checks                                  │
│  ✓ Push/Pull обновления                                     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Структура записи в реестре

```python
from dataclasses import dataclass
from typing import Dict, Optional
from datetime import datetime
from enum import Enum

class HealthStatus(Enum):
    HEALTHY = "healthy"
    UNHEALTHY = "unhealthy"
    UNKNOWN = "unknown"

@dataclass
class ServiceInstance:
    """Представление экземпляра сервиса в реестре"""

    # Уникальный идентификатор экземпляра
    instance_id: str

    # Имя сервиса
    service_name: str

    # Сетевая информация
    host: str
    port: int

    # Статус здоровья
    status: HealthStatus = HealthStatus.UNKNOWN

    # Метаданные
    metadata: Dict[str, str] = None

    # Временные метки
    registered_at: datetime = None
    last_heartbeat: datetime = None

    # Веса для балансировки
    weight: int = 100

    @property
    def url(self) -> str:
        return f"http://{self.host}:{self.port}"

    def is_healthy(self) -> bool:
        return self.status == HealthStatus.HEALTHY


# Пример использования
instance = ServiceInstance(
    instance_id="payment-service-abc123",
    service_name="payment-service",
    host="10.0.1.10",
    port=8080,
    status=HealthStatus.HEALTHY,
    metadata={
        "version": "2.1.0",
        "zone": "us-east-1a",
        "environment": "production"
    },
    registered_at=datetime.now(),
    last_heartbeat=datetime.now()
)
```

---

## Механизмы регистрации

### 1. Self-Registration (Само-регистрация)

Сервис сам регистрирует себя в реестре при запуске и отменяет регистрацию при остановке.

```
┌─────────────────────────────────────────────────────────────┐
│                    Self-Registration                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   Service Instance                    Service Registry       │
│   ┌─────────────┐                    ┌─────────────────┐    │
│   │             │ ──1. Register ────>│                 │    │
│   │  Payment    │                    │   Consul/etcd   │    │
│   │  Service    │ ──2. Heartbeat ───>│   /Eureka       │    │
│   │             │ ──3. Deregister ──>│                 │    │
│   └─────────────┘                    └─────────────────┘    │
│                                                              │
│   1. При запуске: регистрация                               │
│   2. Периодически: heartbeat                                │
│   3. При остановке: дерегистрация                           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Пример на Python с Consul:**

```python
import consul
import atexit
import signal
import socket
import threading
import time
from contextlib import contextmanager

class SelfRegistration:
    def __init__(
        self,
        service_name: str,
        service_port: int,
        consul_host: str = 'localhost',
        consul_port: int = 8500,
        heartbeat_interval: int = 10
    ):
        self.service_name = service_name
        self.service_port = service_port
        self.service_id = f"{service_name}-{socket.gethostname()}-{service_port}"
        self.consul = consul.Consul(host=consul_host, port=consul_port)
        self.heartbeat_interval = heartbeat_interval
        self._heartbeat_thread = None
        self._running = False

    def register(self):
        """Регистрация сервиса в Consul"""

        # Определяем health check
        check = consul.Check.tcp(
            socket.gethostname(),
            self.service_port,
            interval="10s",
            timeout="5s",
            deregister="1m"  # Автоматическая дерегистрация при недоступности
        )

        # Регистрируем сервис
        self.consul.agent.service.register(
            name=self.service_name,
            service_id=self.service_id,
            address=socket.gethostname(),
            port=self.service_port,
            check=check,
            tags=["production", "v2.1.0"],
            meta={
                "version": "2.1.0",
                "protocol": "http"
            }
        )

        print(f"Service {self.service_id} registered")

        # Запускаем heartbeat в фоне
        self._start_heartbeat()

        # Регистрируем обработчики для graceful shutdown
        atexit.register(self.deregister)
        signal.signal(signal.SIGTERM, self._handle_signal)
        signal.signal(signal.SIGINT, self._handle_signal)

    def deregister(self):
        """Дерегистрация сервиса"""
        self._running = False

        try:
            self.consul.agent.service.deregister(self.service_id)
            print(f"Service {self.service_id} deregistered")
        except Exception as e:
            print(f"Error deregistering service: {e}")

    def _start_heartbeat(self):
        """Запуск периодических heartbeat"""
        self._running = True
        self._heartbeat_thread = threading.Thread(
            target=self._heartbeat_loop,
            daemon=True
        )
        self._heartbeat_thread.start()

    def _heartbeat_loop(self):
        """Цикл отправки heartbeat"""
        while self._running:
            try:
                # Consul TTL check pass
                check_id = f"service:{self.service_id}"
                self.consul.agent.check.ttl_pass(check_id)
            except Exception as e:
                print(f"Heartbeat failed: {e}")

            time.sleep(self.heartbeat_interval)

    def _handle_signal(self, signum, frame):
        """Обработка сигналов завершения"""
        print(f"Received signal {signum}, deregistering...")
        self.deregister()
        exit(0)


# Использование с Flask
from flask import Flask

app = Flask(__name__)

# При старте приложения
registration = SelfRegistration(
    service_name="payment-service",
    service_port=8080
)

@app.before_first_request
def register_service():
    registration.register()

@app.route('/health')
def health():
    return {'status': 'healthy'}

@app.route('/api/payments', methods=['POST'])
def process_payment():
    return {'status': 'processed'}

if __name__ == '__main__':
    app.run(port=8080)
```

**Пример с Spring Boot и Eureka:**

```java
// build.gradle
dependencies {
    implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-client'
}

// application.yml
spring:
  application:
    name: payment-service

eureka:
  client:
    service-url:
      defaultZone: http://eureka-server:8761/eureka
    register-with-eureka: true
    fetch-registry: true
  instance:
    prefer-ip-address: true
    lease-renewal-interval-in-seconds: 10
    lease-expiration-duration-in-seconds: 30
    metadata-map:
      version: 2.1.0
      zone: us-east-1a
```

```java
// PaymentServiceApplication.java
@SpringBootApplication
@EnableDiscoveryClient  // Включает self-registration
public class PaymentServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(PaymentServiceApplication.class, args);
    }
}
```

**Плюсы Self-Registration:**
- Простота (сервис контролирует свою регистрацию)
- Гибкость метаданных

**Минусы:**
- Связь сервиса с конкретным реестром
- Дублирование логики регистрации в каждом сервисе
- Проблемы при крэше (может не успеть дерегистрироваться)

---

### 2. Third-Party Registration (Внешняя регистрация)

Отдельный компонент (Registrar) отслеживает сервисы и регистрирует их в реестре.

```
┌─────────────────────────────────────────────────────────────┐
│                Third-Party Registration                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   Services            Registrar          Service Registry    │
│   ┌─────────┐        ┌─────────┐        ┌─────────────────┐ │
│   │ Service │<──────>│         │        │                 │ │
│   │    A    │ Health │ Docker  │──Reg──>│   Consul/etcd   │ │
│   └─────────┘ Check  │ /K8s    │        │   /Eureka       │ │
│   ┌─────────┐        │ Plugin  │──DeReg>│                 │ │
│   │ Service │<──────>│         │        │                 │ │
│   │    B    │        └─────────┘        └─────────────────┘ │
│   └─────────┘                                                │
│                                                              │
│   Registrar отслеживает:                                    │
│   - Запуск/остановку контейнеров                            │
│   - Health check результаты                                 │
│   - Автоматически обновляет реестр                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Пример: Registrator для Docker + Consul:**

```bash
# Запуск Registrator
docker run -d \
    --name=registrator \
    --net=host \
    --volume=/var/run/docker.sock:/tmp/docker.sock \
    gliderlabs/registrator:latest \
    consul://localhost:8500
```

```yaml
# docker-compose.yml
version: '3'

services:
  consul:
    image: consul:latest
    ports:
      - "8500:8500"
    command: agent -server -bootstrap -ui -client=0.0.0.0

  registrator:
    image: gliderlabs/registrator:latest
    depends_on:
      - consul
    volumes:
      - /var/run/docker.sock:/tmp/docker.sock
    command: consul://consul:8500
    network_mode: host

  payment-service:
    image: payment-service:latest
    environment:
      - SERVICE_NAME=payment-service
      - SERVICE_TAGS=production,v2.1.0
      - SERVICE_CHECK_HTTP=/health
      - SERVICE_CHECK_INTERVAL=10s
    ports:
      - "8080:8080"
```

**Kubernetes автоматическая регистрация:**

```yaml
# Kubernetes автоматически регистрирует поды через Endpoints
apiVersion: v1
kind: Service
metadata:
  name: payment-service
spec:
  selector:
    app: payment
  ports:
    - port: 80
      targetPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment
  template:
    metadata:
      labels:
        app: payment
    spec:
      containers:
        - name: payment
          image: payment-service:2.1.0
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
```

**Плюсы Third-Party Registration:**
- Сервисы не знают о реестре
- Централизованное управление
- Работает с любыми сервисами

**Минусы:**
- Дополнительный компонент для поддержки
- Registrar — потенциальная точка отказа

---

## Health Checks и Heartbeats

Health checks критически важны для поддержания актуального состояния реестра.

### Типы Health Checks

```
┌─────────────────────────────────────────────────────────────┐
│                     Health Check Types                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. HTTP Check                                               │
│     ┌──────────┐    GET /health    ┌─────────┐              │
│     │ Registry │ ────────────────> │ Service │              │
│     │          │ <──── 200 OK ──── │         │              │
│     └──────────┘                   └─────────┘              │
│                                                              │
│  2. TCP Check                                                │
│     ┌──────────┐    TCP Connect    ┌─────────┐              │
│     │ Registry │ ────────────────> │ Service │              │
│     │          │ <── ACK/Success ─ │:8080    │              │
│     └──────────┘                   └─────────┘              │
│                                                              │
│  3. TTL (Heartbeat)                                          │
│     ┌─────────┐    Heartbeat       ┌──────────┐             │
│     │ Service │ ─────────────────> │ Registry │             │
│     │         │   каждые N сек     │          │             │
│     └─────────┘                    └──────────┘             │
│                                                              │
│  4. Script/Command Check                                     │
│     Registry выполняет скрипт, проверяющий здоровье         │
│                                                              │
│  5. gRPC Health Check                                        │
│     Стандартный протокол для gRPC сервисов                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Реализация Health Endpoint

```python
from flask import Flask, jsonify
from datetime import datetime
import psycopg2
import redis

app = Flask(__name__)

class HealthChecker:
    """Класс для проверки здоровья компонентов"""

    def __init__(self):
        self.checks = {}

    def register_check(self, name, check_func):
        self.checks[name] = check_func

    def run_checks(self):
        results = {}
        all_healthy = True

        for name, check_func in self.checks.items():
            try:
                check_func()
                results[name] = {"status": "healthy"}
            except Exception as e:
                results[name] = {
                    "status": "unhealthy",
                    "error": str(e)
                }
                all_healthy = False

        return all_healthy, results


health_checker = HealthChecker()

# Проверка подключения к PostgreSQL
def check_database():
    conn = psycopg2.connect(
        host="localhost",
        database="payments",
        user="app",
        password="secret"
    )
    cursor = conn.cursor()
    cursor.execute("SELECT 1")
    cursor.close()
    conn.close()

# Проверка подключения к Redis
def check_redis():
    r = redis.Redis(host='localhost', port=6379)
    r.ping()

# Проверка доступного места на диске
def check_disk_space():
    import shutil
    total, used, free = shutil.disk_usage("/")
    if free / total < 0.1:  # Менее 10% свободного места
        raise Exception(f"Low disk space: {free / (1024**3):.2f} GB free")

# Регистрируем проверки
health_checker.register_check("database", check_database)
health_checker.register_check("redis", check_redis)
health_checker.register_check("disk", check_disk_space)


@app.route('/health')
def health():
    """Базовый health check (liveness)"""
    return jsonify({"status": "healthy"}), 200


@app.route('/health/ready')
def ready():
    """Readiness check — готов ли сервис принимать трафик"""
    all_healthy, results = health_checker.run_checks()

    response = {
        "status": "ready" if all_healthy else "not_ready",
        "checks": results,
        "timestamp": datetime.utcnow().isoformat()
    }

    status_code = 200 if all_healthy else 503
    return jsonify(response), status_code


@app.route('/health/live')
def live():
    """Liveness check — работает ли приложение"""
    return jsonify({
        "status": "alive",
        "timestamp": datetime.utcnow().isoformat()
    }), 200
```

### Consul Health Check Configuration

```json
{
  "service": {
    "name": "payment-service",
    "port": 8080,
    "checks": [
      {
        "name": "HTTP Health Check",
        "http": "http://localhost:8080/health",
        "interval": "10s",
        "timeout": "5s"
      },
      {
        "name": "Readiness Check",
        "http": "http://localhost:8080/health/ready",
        "interval": "15s",
        "timeout": "10s"
      },
      {
        "name": "TCP Port Check",
        "tcp": "localhost:8080",
        "interval": "5s",
        "timeout": "3s"
      }
    ]
  }
}
```

### Kubernetes Probes

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: payment-service
spec:
  containers:
    - name: payment
      image: payment-service:2.1.0
      ports:
        - containerPort: 8080

      # Liveness probe — перезапуск при failure
      livenessProbe:
        httpGet:
          path: /health/live
          port: 8080
        initialDelaySeconds: 15
        periodSeconds: 10
        failureThreshold: 3

      # Readiness probe — исключение из балансировки
      readinessProbe:
        httpGet:
          path: /health/ready
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 5
        failureThreshold: 3

      # Startup probe — для медленно стартующих приложений
      startupProbe:
        httpGet:
          path: /health
          port: 8080
        initialDelaySeconds: 0
        periodSeconds: 5
        failureThreshold: 30  # 30 * 5 = 150 секунд на старт
```

---

## DNS-based Service Discovery

DNS — простейший и самый распространённый способ обнаружения сервисов.

### Принцип работы

```
┌─────────────────────────────────────────────────────────────┐
│                  DNS-based Service Discovery                 │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   Client                      DNS Server                     │
│   ┌─────┐                    ┌──────────────────┐           │
│   │     │ ─── Query ────────>│                  │           │
│   │ App │  payment.svc.local │   payment.svc.local:         │
│   │     │ <── Response ──────│    A    10.0.1.10            │
│   └─────┘   10.0.1.10        │    A    10.0.1.11            │
│      │      10.0.1.11        │    A    10.0.1.12            │
│      │                       └──────────────────┘           │
│      │                                                       │
│      └──> Connect to 10.0.1.10:8080                         │
│                                                              │
│   SRV Records (с портами):                                   │
│   _payment._tcp.svc.local                                    │
│     SRV 0 0 8080 payment-1.svc.local                        │
│     SRV 0 0 8080 payment-2.svc.local                        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Kubernetes DNS

```yaml
# В Kubernetes DNS настроен автоматически

# Формат DNS имён:
# <service-name>.<namespace>.svc.cluster.local

# Примеры:
# payment-service.default.svc.cluster.local
# user-service.production.svc.cluster.local

# Headless Service (для StatefulSet)
apiVersion: v1
kind: Service
metadata:
  name: payment-service
  namespace: production
spec:
  clusterIP: None  # Headless service
  selector:
    app: payment
  ports:
    - port: 8080
      targetPort: 8080

---
# StatefulSet получит DNS имена вида:
# payment-0.payment-service.production.svc.cluster.local
# payment-1.payment-service.production.svc.cluster.local
```

**Использование в коде:**

```python
import socket
import requests

class KubernetesDNSClient:
    """Клиент для работы с Kubernetes DNS"""

    def __init__(self, namespace="default"):
        self.namespace = namespace

    def get_service_url(self, service_name, port=80):
        """Получить URL сервиса через DNS"""
        # Kubernetes DNS автоматически резолвит имена
        fqdn = f"{service_name}.{self.namespace}.svc.cluster.local"
        return f"http://{fqdn}:{port}"

    def get_all_endpoints(self, service_name):
        """Получить все endpoints через DNS lookup"""
        fqdn = f"{service_name}.{self.namespace}.svc.cluster.local"
        try:
            # Получаем все IP адреса
            _, _, ip_addresses = socket.gethostbyname_ex(fqdn)
            return ip_addresses
        except socket.gaierror:
            return []

    def call_service(self, service_name, path, method="GET", **kwargs):
        """Вызов сервиса через DNS"""
        url = f"{self.get_service_url(service_name)}{path}"
        response = requests.request(method, url, **kwargs)
        return response.json()


# Использование
dns_client = KubernetesDNSClient(namespace="production")

# Простой вызов
result = dns_client.call_service(
    "payment-service",
    "/api/payments",
    method="POST",
    json={"amount": 100}
)
```

### Consul DNS

```bash
# Consul предоставляет DNS интерфейс на порту 8600

# Запрос A записей
dig @127.0.0.1 -p 8600 payment-service.service.consul

# Запрос SRV записей (с портами)
dig @127.0.0.1 -p 8600 payment-service.service.consul SRV

# Запрос с тегами
dig @127.0.0.1 -p 8600 production.payment-service.service.consul

# Запрос по датацентру
dig @127.0.0.1 -p 8600 payment-service.service.dc1.consul
```

**Настройка системного DNS для Consul:**

```bash
# /etc/dnsmasq.d/consul
server=/consul/127.0.0.1#8600
```

---

## Популярные решения

### 1. HashiCorp Consul

Consul — полнофункциональное решение для service discovery, конфигурации и service mesh.

```
┌─────────────────────────────────────────────────────────────┐
│                        Consul                                │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                   Consul Servers                     │    │
│  │  ┌────────┐ ┌────────┐ ┌────────┐                   │    │
│  │  │Server 1│ │Server 2│ │Server 3│  Raft Consensus   │    │
│  │  │(Leader)│ │        │ │        │                   │    │
│  │  └────────┘ └────────┘ └────────┘                   │    │
│  └─────────────────────────────────────────────────────┘    │
│           ↑                    ↑                             │
│           │       Gossip       │                             │
│  ┌────────┴────────────────────┴────────┐                   │
│  │              Consul Agents            │                   │
│  │  ┌──────┐  ┌──────┐  ┌──────┐        │                   │
│  │  │Agent │  │Agent │  │Agent │        │                   │
│  │  │+ Svc │  │+ Svc │  │+ Svc │        │                   │
│  │  └──────┘  └──────┘  └──────┘        │                   │
│  └──────────────────────────────────────┘                   │
│                                                              │
│  Возможности:                                                │
│  ✓ Service Discovery (HTTP API + DNS)                       │
│  ✓ Health Checking                                          │
│  ✓ Key/Value Store                                          │
│  ✓ Multi-datacenter                                         │
│  ✓ Service Mesh (Consul Connect)                            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Docker Compose для Consul кластера:**

```yaml
version: '3.8'

services:
  consul-server-1:
    image: consul:1.15
    container_name: consul-server-1
    command: agent -server -bootstrap-expect=3 -node=server-1 -client=0.0.0.0 -ui
    ports:
      - "8500:8500"
      - "8600:8600/tcp"
      - "8600:8600/udp"
    volumes:
      - consul-data-1:/consul/data
    networks:
      - consul-net

  consul-server-2:
    image: consul:1.15
    container_name: consul-server-2
    command: agent -server -retry-join=consul-server-1 -node=server-2 -client=0.0.0.0
    volumes:
      - consul-data-2:/consul/data
    networks:
      - consul-net

  consul-server-3:
    image: consul:1.15
    container_name: consul-server-3
    command: agent -server -retry-join=consul-server-1 -node=server-3 -client=0.0.0.0
    volumes:
      - consul-data-3:/consul/data
    networks:
      - consul-net

  payment-service:
    image: payment-service:latest
    environment:
      - CONSUL_HTTP_ADDR=consul-server-1:8500
    depends_on:
      - consul-server-1
    networks:
      - consul-net

volumes:
  consul-data-1:
  consul-data-2:
  consul-data-3:

networks:
  consul-net:
    driver: bridge
```

**Python SDK для Consul:**

```python
import consul
from typing import List, Optional, Dict
import json

class ConsulServiceDiscovery:
    def __init__(self, host='localhost', port=8500):
        self.consul = consul.Consul(host=host, port=port)

    def register_service(
        self,
        name: str,
        service_id: str,
        address: str,
        port: int,
        tags: List[str] = None,
        meta: Dict[str, str] = None,
        health_check_url: str = None
    ):
        """Регистрация сервиса с health check"""

        check = None
        if health_check_url:
            check = consul.Check.http(
                health_check_url,
                interval="10s",
                timeout="5s",
                deregister="1m"
            )

        self.consul.agent.service.register(
            name=name,
            service_id=service_id,
            address=address,
            port=port,
            tags=tags or [],
            meta=meta or {},
            check=check
        )

    def deregister_service(self, service_id: str):
        """Дерегистрация сервиса"""
        self.consul.agent.service.deregister(service_id)

    def get_healthy_services(self, name: str) -> List[Dict]:
        """Получить все здоровые экземпляры сервиса"""
        _, services = self.consul.health.service(name, passing=True)

        return [{
            'id': s['Service']['ID'],
            'address': s['Service']['Address'],
            'port': s['Service']['Port'],
            'tags': s['Service']['Tags'],
            'meta': s['Service']['Meta']
        } for s in services]

    def watch_service(self, name: str, callback):
        """Подписка на изменения сервиса"""
        index = None

        while True:
            index, services = self.consul.health.service(
                name,
                passing=True,
                index=index,  # Блокирующий запрос
                wait="5m"
            )
            callback(services)

    # Key-Value Store
    def put_config(self, key: str, value: dict):
        """Сохранить конфигурацию"""
        self.consul.kv.put(key, json.dumps(value))

    def get_config(self, key: str) -> Optional[dict]:
        """Получить конфигурацию"""
        _, data = self.consul.kv.get(key)
        if data:
            return json.loads(data['Value'])
        return None


# Пример использования
sd = ConsulServiceDiscovery()

# Регистрация
sd.register_service(
    name="payment-service",
    service_id="payment-1",
    address="10.0.1.10",
    port=8080,
    tags=["production", "v2.1.0"],
    meta={"version": "2.1.0"},
    health_check_url="http://10.0.1.10:8080/health"
)

# Обнаружение
services = sd.get_healthy_services("payment-service")
for svc in services:
    print(f"Found: {svc['address']}:{svc['port']}")
```

---

### 2. etcd

etcd — распределённое key-value хранилище, часто используемое для service discovery.

```python
import etcd3
from typing import List, Optional
import json
import threading

class EtcdServiceDiscovery:
    def __init__(self, host='localhost', port=2379):
        self.client = etcd3.client(host=host, port=port)
        self.prefix = "/services/"
        self._lease = None

    def register_service(
        self,
        name: str,
        instance_id: str,
        address: str,
        port: int,
        ttl: int = 30
    ):
        """Регистрация сервиса с TTL"""

        # Создаём lease для автоматического удаления
        self._lease = self.client.lease(ttl)

        key = f"{self.prefix}{name}/{instance_id}"
        value = json.dumps({
            'address': address,
            'port': port,
            'instance_id': instance_id
        })

        self.client.put(key, value, lease=self._lease)

        # Запускаем keep-alive в фоне
        self._start_keepalive()

    def _start_keepalive(self):
        """Поддержание lease активным"""
        def keepalive():
            for response in self.client.refresh_lease(self._lease.id):
                pass

        thread = threading.Thread(target=keepalive, daemon=True)
        thread.start()

    def deregister_service(self, name: str, instance_id: str):
        """Дерегистрация"""
        key = f"{self.prefix}{name}/{instance_id}"
        self.client.delete(key)

    def get_services(self, name: str) -> List[dict]:
        """Получить все экземпляры сервиса"""
        prefix = f"{self.prefix}{name}/"

        services = []
        for value, metadata in self.client.get_prefix(prefix):
            if value:
                services.append(json.loads(value))

        return services

    def watch_services(self, name: str, callback):
        """Подписка на изменения"""
        prefix = f"{self.prefix}{name}/"

        events_iterator, cancel = self.client.watch_prefix(prefix)

        for event in events_iterator:
            if isinstance(event, etcd3.events.PutEvent):
                callback('put', json.loads(event.value))
            elif isinstance(event, etcd3.events.DeleteEvent):
                callback('delete', event.key.decode())


# Использование
sd = EtcdServiceDiscovery()

sd.register_service(
    name="payment-service",
    instance_id="payment-1",
    address="10.0.1.10",
    port=8080,
    ttl=30
)

services = sd.get_services("payment-service")
```

---

### 3. Apache ZooKeeper

ZooKeeper — координационный сервис, используемый для service discovery в экосистеме Java.

```python
from kazoo.client import KazooClient
from kazoo.recipe.watchers import ChildrenWatch
import json

class ZooKeeperServiceDiscovery:
    def __init__(self, hosts='localhost:2181'):
        self.zk = KazooClient(hosts=hosts)
        self.zk.start()
        self.base_path = "/services"

    def register_service(
        self,
        name: str,
        instance_id: str,
        address: str,
        port: int
    ):
        """Регистрация эфемерного узла"""

        service_path = f"{self.base_path}/{name}"
        instance_path = f"{service_path}/{instance_id}"

        # Создаём путь если не существует
        self.zk.ensure_path(service_path)

        data = json.dumps({
            'address': address,
            'port': port,
            'instance_id': instance_id
        }).encode()

        # Эфемерный узел — удалится при disconnect
        self.zk.create(
            instance_path,
            data,
            ephemeral=True,
            makepath=True
        )

    def get_services(self, name: str):
        """Получить все экземпляры"""
        service_path = f"{self.base_path}/{name}"

        if not self.zk.exists(service_path):
            return []

        instances = []
        children = self.zk.get_children(service_path)

        for child in children:
            data, stat = self.zk.get(f"{service_path}/{child}")
            instances.append(json.loads(data))

        return instances

    def watch_services(self, name: str, callback):
        """Подписка на изменения"""
        service_path = f"{self.base_path}/{name}"

        @ChildrenWatch(self.zk, service_path)
        def watch_children(children):
            callback(self.get_services(name))

    def close(self):
        self.zk.stop()
```

---

### 4. Netflix Eureka

Eureka — решение для service discovery от Netflix, популярно в Spring Cloud.

```yaml
# eureka-server application.yml
server:
  port: 8761

eureka:
  instance:
    hostname: localhost
  client:
    register-with-eureka: false
    fetch-registry: false
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
```

```java
// Eureka Server
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

```yaml
# Eureka Client (payment-service) application.yml
spring:
  application:
    name: payment-service

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
    registry-fetch-interval-seconds: 5
  instance:
    prefer-ip-address: true
    lease-renewal-interval-in-seconds: 10
    lease-expiration-duration-in-seconds: 30
    instance-id: ${spring.application.name}:${random.uuid}
    metadata-map:
      version: 2.1.0
```

```java
// Eureka Client с Feign
@FeignClient(name = "payment-service")
public interface PaymentServiceClient {

    @PostMapping("/api/payments")
    Payment processPayment(@RequestBody PaymentRequest request);
}

@Service
public class OrderService {

    @Autowired
    private PaymentServiceClient paymentClient;

    public Order createOrder(OrderRequest request) {
        Payment payment = paymentClient.processPayment(
            new PaymentRequest(request.getAmount())
        );
        // ...
    }
}
```

---

### 5. Kubernetes Service Discovery

Kubernetes предоставляет встроенный service discovery через DNS и Endpoints.

```yaml
# ClusterIP Service — внутренний discovery
apiVersion: v1
kind: Service
metadata:
  name: payment-service
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: payment
  ports:
    - name: http
      port: 80
      targetPort: 8080

---
# Headless Service — для прямого доступа к подам
apiVersion: v1
kind: Service
metadata:
  name: payment-headless
spec:
  clusterIP: None
  selector:
    app: payment
  ports:
    - port: 8080

---
# ExternalName — алиас для внешнего сервиса
apiVersion: v1
kind: Service
metadata:
  name: external-database
spec:
  type: ExternalName
  externalName: db.example.com
```

**Работа с Kubernetes API для service discovery:**

```python
from kubernetes import client, config
from typing import List, Dict

class KubernetesServiceDiscovery:
    def __init__(self, namespace: str = "default"):
        # Загружаем конфиг (in-cluster или из ~/.kube/config)
        try:
            config.load_incluster_config()
        except:
            config.load_kube_config()

        self.v1 = client.CoreV1Api()
        self.namespace = namespace

    def get_service_endpoints(self, service_name: str) -> List[Dict]:
        """Получить все endpoints сервиса"""

        endpoints = self.v1.read_namespaced_endpoints(
            name=service_name,
            namespace=self.namespace
        )

        result = []
        for subset in endpoints.subsets or []:
            for address in subset.addresses or []:
                for port in subset.ports or []:
                    result.append({
                        'ip': address.ip,
                        'port': port.port,
                        'pod_name': address.target_ref.name if address.target_ref else None
                    })

        return result

    def get_pods_by_label(self, label_selector: str) -> List[Dict]:
        """Получить поды по метке"""

        pods = self.v1.list_namespaced_pod(
            namespace=self.namespace,
            label_selector=label_selector
        )

        return [{
            'name': pod.metadata.name,
            'ip': pod.status.pod_ip,
            'status': pod.status.phase,
            'ready': all(
                c.ready for c in (pod.status.container_statuses or [])
            )
        } for pod in pods.items]

    def watch_endpoints(self, service_name: str, callback):
        """Подписка на изменения endpoints"""
        from kubernetes import watch

        w = watch.Watch()

        for event in w.stream(
            self.v1.list_namespaced_endpoints,
            namespace=self.namespace,
            field_selector=f"metadata.name={service_name}"
        ):
            callback(event['type'], event['object'])


# Использование
sd = KubernetesServiceDiscovery(namespace="production")

# Получить все endpoints
endpoints = sd.get_service_endpoints("payment-service")
for ep in endpoints:
    print(f"Endpoint: {ep['ip']}:{ep['port']}")
```

---

## Best Practices

### 1. Выбор решения

```
┌─────────────────────────────────────────────────────────────┐
│              Как выбрать Service Discovery                   │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Kubernetes environment?                                     │
│  ├─ Да  → Используй встроенный DNS + Services               │
│  │        (Consul/etcd только для cross-cluster)            │
│  └─ Нет                                                     │
│     ├─ Java/Spring ecosystem?                               │
│     │  └─ Да → Eureka + Spring Cloud                        │
│     │                                                        │
│     ├─ Нужен multi-datacenter?                              │
│     │  └─ Да → Consul (лучшая поддержка)                    │
│     │                                                        │
│     ├─ Нужен KV store + service mesh?                       │
│     │  └─ Да → Consul                                       │
│     │                                                        │
│     └─ Простой service discovery?                           │
│        └─ etcd (если уже есть) или Consul                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 2. Health Checks

```python
# Best practices для health checks

# 1. Разделяй liveness и readiness
@app.route('/health/live')
def liveness():
    """Только проверка что процесс жив"""
    return {'status': 'alive'}

@app.route('/health/ready')
def readiness():
    """Проверка готовности принимать трафик"""
    if not database_connected():
        return {'status': 'not ready'}, 503
    if not cache_warmed():
        return {'status': 'warming up'}, 503
    return {'status': 'ready'}

# 2. Устанавливай правильные таймауты
# - Health check interval: 10-30 сек
# - Timeout: 3-5 сек
# - Deregister after: 1-3 мин

# 3. Не делай тяжёлые операции в health check
# Плохо:
def health():
    result = run_full_database_test()  # Долго!
    return result

# Хорошо:
def health():
    return db.execute("SELECT 1")  # Быстро
```

### 3. Кэширование и отказоустойчивость

```python
import time
from threading import Lock
from typing import Optional, List

class ResilientServiceDiscovery:
    """Service Discovery с кэшированием и fallback"""

    def __init__(self, primary_registry, fallback_registry=None):
        self.primary = primary_registry
        self.fallback = fallback_registry
        self._cache = {}
        self._cache_ttl = 30  # секунды
        self._lock = Lock()

    def get_instances(self, service_name: str) -> List[dict]:
        """Получить инстансы с кэшированием"""

        # Проверяем кэш
        cached = self._get_from_cache(service_name)
        if cached is not None:
            return cached

        # Пытаемся получить из primary
        try:
            instances = self.primary.get_services(service_name)
            self._update_cache(service_name, instances)
            return instances
        except Exception as e:
            print(f"Primary registry failed: {e}")

        # Fallback на secondary
        if self.fallback:
            try:
                instances = self.fallback.get_services(service_name)
                self._update_cache(service_name, instances)
                return instances
            except Exception as e:
                print(f"Fallback registry failed: {e}")

        # Возвращаем устаревший кэш если есть
        stale = self._get_stale_cache(service_name)
        if stale:
            print(f"Using stale cache for {service_name}")
            return stale

        raise Exception(f"Cannot discover service: {service_name}")

    def _get_from_cache(self, service_name: str) -> Optional[List]:
        with self._lock:
            if service_name in self._cache:
                data, timestamp = self._cache[service_name]
                if time.time() - timestamp < self._cache_ttl:
                    return data
        return None

    def _get_stale_cache(self, service_name: str) -> Optional[List]:
        with self._lock:
            if service_name in self._cache:
                data, _ = self._cache[service_name]
                return data
        return None

    def _update_cache(self, service_name: str, instances: List):
        with self._lock:
            self._cache[service_name] = (instances, time.time())
```

### 4. Graceful Shutdown

```python
import signal
import sys
from contextlib import contextmanager

class GracefulServiceLifecycle:
    """Управление жизненным циклом сервиса"""

    def __init__(self, registry, service_config):
        self.registry = registry
        self.config = service_config
        self._shutting_down = False

    def start(self):
        """Запуск сервиса с регистрацией"""

        # Регистрируем обработчики сигналов
        signal.signal(signal.SIGTERM, self._handle_shutdown)
        signal.signal(signal.SIGINT, self._handle_shutdown)

        # Регистрируемся в реестре
        self.registry.register(
            name=self.config['name'],
            instance_id=self.config['instance_id'],
            address=self.config['address'],
            port=self.config['port']
        )

        print(f"Service {self.config['name']} started and registered")

    def _handle_shutdown(self, signum, frame):
        """Graceful shutdown"""
        if self._shutting_down:
            return

        self._shutting_down = True
        print("Initiating graceful shutdown...")

        # 1. Дерегистрация (перестаём получать новые запросы)
        try:
            self.registry.deregister(self.config['instance_id'])
            print("Deregistered from service registry")
        except Exception as e:
            print(f"Deregistration failed: {e}")

        # 2. Ждём завершения текущих запросов
        print("Waiting for in-flight requests to complete...")
        time.sleep(5)  # Даём время на завершение

        # 3. Закрываем соединения
        self._close_connections()

        print("Shutdown complete")
        sys.exit(0)

    def _close_connections(self):
        """Закрытие соединений с базами и т.д."""
        pass


# Использование с Flask
from flask import Flask

app = Flask(__name__)
lifecycle = GracefulServiceLifecycle(registry, config)

@app.before_first_request
def on_startup():
    lifecycle.start()
```

### 5. Метаданные и версионирование

```python
# Используй метаданные для умной маршрутизации

# Регистрация с метаданными
registry.register(
    name="payment-service",
    instance_id="payment-v2-abc123",
    address="10.0.1.10",
    port=8080,
    metadata={
        "version": "2.1.0",
        "environment": "production",
        "region": "us-east-1",
        "zone": "us-east-1a",
        "canary": "false",
        "weight": "100"
    },
    tags=["production", "v2", "primary"]
)

# Выбор инстанса по метаданным
def get_instance_by_version(service_name: str, version: str):
    instances = registry.get_services(service_name)

    matching = [
        i for i in instances
        if i['metadata'].get('version', '').startswith(version)
    ]

    if not matching:
        raise Exception(f"No instances for version {version}")

    return random.choice(matching)

# Canary deployment
def get_instance_with_canary(service_name: str, canary_percent: int = 10):
    instances = registry.get_services(service_name)

    canary = [i for i in instances if i['metadata'].get('canary') == 'true']
    stable = [i for i in instances if i['metadata'].get('canary') != 'true']

    if canary and random.randint(1, 100) <= canary_percent:
        return random.choice(canary)

    return random.choice(stable)
```

---

## Сравнительная таблица решений

| Характеристика | Consul | etcd | ZooKeeper | Eureka | K8s DNS |
|----------------|--------|------|-----------|--------|---------|
| **Язык** | Go | Go | Java | Java | - |
| **Консенсус** | Raft | Raft | ZAB | - (AP) | - |
| **CP/AP** | CP | CP | CP | AP | - |
| **DNS интерфейс** | Да | Нет | Нет | Нет | Да |
| **HTTP API** | Да | Да | Нет | Да | Да |
| **Health Checks** | Встроенные | Нет | Нет | Да | Да (probes) |
| **KV Store** | Да | Да | Да | Нет | ConfigMap |
| **Multi-DC** | Отлично | Хорошо | Хорошо | Регионы | Federation |
| **Service Mesh** | Connect | Нет | Нет | Нет | Istio/Linkerd |
| **Простота** | Средняя | Средняя | Сложная | Простая | Простая |

---

## Резюме

**Service Discovery** — критически важный компонент микросервисной архитектуры:

1. **Два основных паттерна**: Client-side (клиент сам ищет) и Server-side (через load balancer)

2. **Ключевые компоненты**:
   - Service Registry — хранит информацию о сервисах
   - Health Checks — поддерживают актуальность данных
   - Механизм регистрации — self или third-party

3. **Популярные решения**:
   - Consul — универсальное решение с множеством возможностей
   - etcd — простой и надёжный KV store
   - Kubernetes DNS — встроенный в K8s

4. **Best practices**:
   - Всегда используй health checks
   - Кэшируй результаты discovery
   - Реализуй graceful shutdown
   - Используй метаданные для умной маршрутизации
