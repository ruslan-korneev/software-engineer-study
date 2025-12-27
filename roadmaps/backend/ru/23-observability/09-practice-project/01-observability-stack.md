# Практический проект: Полный стек наблюдаемости (Observability Stack)

## Введение

В этом практическом проекте мы создадим полноценный стек наблюдаемости для микросервисного приложения. Проект объединяет все изученные концепции: метрики, логи, трейсы, алерты и дашборды.

## Архитектура проекта

```
┌─────────────────────────────────────────────────────────────────┐
│                        Пользователи                              │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                       API Gateway                                │
│                    (порт: 8080)                                  │
└─────────────────────────────────────────────────────────────────┘
                                │
                ┌───────────────┼───────────────┐
                ▼               ▼               ▼
        ┌───────────┐   ┌───────────┐   ┌───────────┐
        │  Order    │   │  Product  │   │  Payment  │
        │  Service  │   │  Service  │   │  Service  │
        │  :8081    │   │  :8082    │   │  :8083    │
        └───────────┘   └───────────┘   └───────────┘
                │               │               │
                └───────────────┼───────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Observability Stack                           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │Prometheus│  │ Grafana  │  │  Jaeger  │  │   Loki   │        │
│  │  :9090   │  │  :3000   │  │  :16686  │  │  :3100   │        │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘        │
│  ┌──────────┐  ┌──────────┐                                     │
│  │AlertMan- │  │ Promtail │                                     │
│  │  ager    │  │          │                                     │
│  │  :9093   │  │          │                                     │
│  └──────────┘  └──────────┘                                     │
└─────────────────────────────────────────────────────────────────┘
```

## Структура проекта

```
observability-project/
├── docker-compose.yml
├── .env
├── services/
│   ├── api-gateway/
│   │   ├── main.py
│   │   ├── requirements.txt
│   │   └── Dockerfile
│   ├── order-service/
│   │   ├── main.py
│   │   ├── requirements.txt
│   │   └── Dockerfile
│   ├── product-service/
│   │   ├── main.py
│   │   ├── requirements.txt
│   │   └── Dockerfile
│   └── payment-service/
│       ├── main.py
│       ├── requirements.txt
│       └── Dockerfile
├── prometheus/
│   ├── prometheus.yml
│   └── rules/
│       └── alerts.yml
├── alertmanager/
│   └── alertmanager.yml
├── grafana/
│   ├── provisioning/
│   │   ├── datasources/
│   │   │   └── datasources.yml
│   │   └── dashboards/
│   │       ├── dashboards.yml
│   │       └── application-dashboard.json
│   └── grafana.ini
├── loki/
│   └── loki-config.yml
├── promtail/
│   └── promtail-config.yml
└── jaeger/
    └── jaeger-config.yml
```

## Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  # ============================================
  # APPLICATION SERVICES
  # ============================================

  api-gateway:
    build: ./services/api-gateway
    container_name: api-gateway
    ports:
      - "8080:8080"
    environment:
      - SERVICE_NAME=api-gateway
      - ORDER_SERVICE_URL=http://order-service:8081
      - PRODUCT_SERVICE_URL=http://product-service:8082
      - PAYMENT_SERVICE_URL=http://payment-service:8083
      - JAEGER_AGENT_HOST=jaeger
      - JAEGER_AGENT_PORT=6831
      - LOG_LEVEL=INFO
    depends_on:
      - order-service
      - product-service
      - payment-service
      - jaeger
    networks:
      - observability-network
    labels:
      logging: "promtail"
      logging_jobname: "api-gateway"
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
        tag: "{{.Name}}"

  order-service:
    build: ./services/order-service
    container_name: order-service
    ports:
      - "8081:8081"
    environment:
      - SERVICE_NAME=order-service
      - JAEGER_AGENT_HOST=jaeger
      - JAEGER_AGENT_PORT=6831
      - LOG_LEVEL=INFO
    depends_on:
      - jaeger
    networks:
      - observability-network
    labels:
      logging: "promtail"
      logging_jobname: "order-service"
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
        tag: "{{.Name}}"

  product-service:
    build: ./services/product-service
    container_name: product-service
    ports:
      - "8082:8082"
    environment:
      - SERVICE_NAME=product-service
      - JAEGER_AGENT_HOST=jaeger
      - JAEGER_AGENT_PORT=6831
      - LOG_LEVEL=INFO
    depends_on:
      - jaeger
    networks:
      - observability-network
    labels:
      logging: "promtail"
      logging_jobname: "product-service"
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
        tag: "{{.Name}}"

  payment-service:
    build: ./services/payment-service
    container_name: payment-service
    ports:
      - "8083:8083"
    environment:
      - SERVICE_NAME=payment-service
      - JAEGER_AGENT_HOST=jaeger
      - JAEGER_AGENT_PORT=6831
      - LOG_LEVEL=INFO
      - FAILURE_RATE=0.1  # 10% ошибок для демонстрации
    depends_on:
      - jaeger
    networks:
      - observability-network
    labels:
      logging: "promtail"
      logging_jobname: "payment-service"
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
        tag: "{{.Name}}"

  # ============================================
  # METRICS STACK (PROMETHEUS)
  # ============================================

  prometheus:
    image: prom/prometheus:v2.47.0
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/rules:/etc/prometheus/rules:ro
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=15d'
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.console.templates=/usr/share/prometheus/consoles'
      - '--web.enable-lifecycle'
    networks:
      - observability-network
    restart: unless-stopped

  alertmanager:
    image: prom/alertmanager:v0.26.0
    container_name: alertmanager
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
      - alertmanager_data:/alertmanager
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'
      - '--cluster.advertise-address=0.0.0.0:9093'
    networks:
      - observability-network
    restart: unless-stopped

  # ============================================
  # LOGGING STACK (LOKI + PROMTAIL)
  # ============================================

  loki:
    image: grafana/loki:2.9.0
    container_name: loki
    ports:
      - "3100:3100"
    volumes:
      - ./loki/loki-config.yml:/etc/loki/local-config.yaml:ro
      - loki_data:/loki
    command: -config.file=/etc/loki/local-config.yaml
    networks:
      - observability-network
    restart: unless-stopped

  promtail:
    image: grafana/promtail:2.9.0
    container_name: promtail
    volumes:
      - ./promtail/promtail-config.yml:/etc/promtail/config.yml:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock
    command: -config.file=/etc/promtail/config.yml
    depends_on:
      - loki
    networks:
      - observability-network
    restart: unless-stopped

  # ============================================
  # TRACING STACK (JAEGER)
  # ============================================

  jaeger:
    image: jaegertracing/all-in-one:1.50
    container_name: jaeger
    ports:
      - "5775:5775/udp"   # Zipkin/Thrift (deprecated)
      - "6831:6831/udp"   # Jaeger Thrift compact
      - "6832:6832/udp"   # Jaeger Thrift binary
      - "5778:5778"       # Configs
      - "16686:16686"     # Web UI
      - "14268:14268"     # Jaeger HTTP
      - "14250:14250"     # gRPC
      - "9411:9411"       # Zipkin compatible
    environment:
      - COLLECTOR_ZIPKIN_HOST_PORT=:9411
      - SPAN_STORAGE_TYPE=badger
      - BADGER_EPHEMERAL=false
      - BADGER_DIRECTORY_VALUE=/badger/data
      - BADGER_DIRECTORY_KEY=/badger/key
    volumes:
      - jaeger_data:/badger
    networks:
      - observability-network
    restart: unless-stopped

  # ============================================
  # VISUALIZATION (GRAFANA)
  # ============================================

  grafana:
    image: grafana/grafana:10.1.0
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
      - GF_FEATURE_TOGGLES_ENABLE=traceqlEditor
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
      - ./grafana/grafana.ini:/etc/grafana/grafana.ini:ro
      - grafana_data:/var/lib/grafana
    depends_on:
      - prometheus
      - loki
      - jaeger
    networks:
      - observability-network
    restart: unless-stopped

  # ============================================
  # LOAD TESTING
  # ============================================

  k6:
    image: grafana/k6:0.46.0
    container_name: k6
    volumes:
      - ./k6:/scripts
    entrypoint: ["tail", "-f", "/dev/null"]
    networks:
      - observability-network

networks:
  observability-network:
    driver: bridge

volumes:
  prometheus_data:
  alertmanager_data:
  loki_data:
  jaeger_data:
  grafana_data:
```

## Сервисы приложения

### Базовый класс сервиса (общий для всех)

```python
# services/common/base_service.py
import os
import time
import json
import logging
import random
from functools import wraps
from typing import Optional, Callable, Any

from flask import Flask, request, g, jsonify
from prometheus_client import Counter, Histogram, Gauge, generate_latest, CONTENT_TYPE_LATEST
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor
from opentelemetry.sdk.resources import Resource
from opentelemetry.semconv.resource import ResourceAttributes
from opentelemetry.trace.propagation.tracecontext import TraceContextTextMapPropagator
import structlog


class ObservableService:
    """Базовый класс для сервисов с полной наблюдаемостью"""

    def __init__(self, service_name: str, port: int):
        self.service_name = service_name
        self.port = port
        self.app = Flask(service_name)

        # Инициализация компонентов наблюдаемости
        self._setup_logging()
        self._setup_tracing()
        self._setup_metrics()
        self._setup_middleware()

    def _setup_logging(self):
        """Настройка структурированного логирования"""
        structlog.configure(
            processors=[
                structlog.stdlib.filter_by_level,
                structlog.stdlib.add_logger_name,
                structlog.stdlib.add_log_level,
                structlog.processors.TimeStamper(fmt="iso"),
                structlog.processors.StackInfoRenderer(),
                structlog.processors.format_exc_info,
                structlog.processors.UnicodeDecoder(),
                structlog.processors.JSONRenderer()
            ],
            wrapper_class=structlog.stdlib.BoundLogger,
            context_class=dict,
            logger_factory=structlog.stdlib.LoggerFactory(),
            cache_logger_on_first_use=True,
        )

        log_level = os.environ.get('LOG_LEVEL', 'INFO')
        logging.basicConfig(
            format="%(message)s",
            level=getattr(logging, log_level.upper()),
        )

        self.logger = structlog.get_logger(self.service_name)

    def _setup_tracing(self):
        """Настройка распределенной трассировки через Jaeger"""
        resource = Resource(attributes={
            ResourceAttributes.SERVICE_NAME: self.service_name,
            ResourceAttributes.SERVICE_VERSION: "1.0.0",
            ResourceAttributes.DEPLOYMENT_ENVIRONMENT: os.environ.get('ENVIRONMENT', 'development')
        })

        provider = TracerProvider(resource=resource)

        jaeger_host = os.environ.get('JAEGER_AGENT_HOST', 'localhost')
        jaeger_port = int(os.environ.get('JAEGER_AGENT_PORT', 6831))

        jaeger_exporter = JaegerExporter(
            agent_host_name=jaeger_host,
            agent_port=jaeger_port,
        )

        provider.add_span_processor(BatchSpanProcessor(jaeger_exporter))
        trace.set_tracer_provider(provider)

        self.tracer = trace.get_tracer(self.service_name)

        # Автоматическая инструментация Flask и requests
        FlaskInstrumentor().instrument_app(self.app)
        RequestsInstrumentor().instrument()

    def _setup_metrics(self):
        """Настройка метрик Prometheus"""
        # RED метрики (Rate, Errors, Duration)
        self.request_count = Counter(
            'http_requests_total',
            'Total HTTP requests',
            ['service', 'method', 'endpoint', 'status']
        )

        self.request_duration = Histogram(
            'http_request_duration_seconds',
            'HTTP request duration in seconds',
            ['service', 'method', 'endpoint'],
            buckets=[.005, .01, .025, .05, .075, .1, .25, .5, .75, 1.0, 2.5, 5.0, 7.5, 10.0]
        )

        self.request_in_progress = Gauge(
            'http_requests_in_progress',
            'Number of HTTP requests in progress',
            ['service', 'method', 'endpoint']
        )

        # Бизнес-метрики
        self.business_events = Counter(
            'business_events_total',
            'Business events counter',
            ['service', 'event_type', 'status']
        )

        # Системные метрики
        self.active_connections = Gauge(
            'active_connections',
            'Number of active connections',
            ['service']
        )

    def _setup_middleware(self):
        """Настройка middleware для сбора метрик"""

        @self.app.before_request
        def before_request():
            g.start_time = time.time()
            g.request_id = request.headers.get('X-Request-ID', str(random.randint(100000, 999999)))

            # Получаем trace context
            current_span = trace.get_current_span()
            if current_span:
                g.trace_id = format(current_span.get_span_context().trace_id, '032x')
                g.span_id = format(current_span.get_span_context().span_id, '016x')
            else:
                g.trace_id = 'unknown'
                g.span_id = 'unknown'

            self.request_in_progress.labels(
                service=self.service_name,
                method=request.method,
                endpoint=request.endpoint or 'unknown'
            ).inc()

            self.logger.info(
                "request_started",
                request_id=g.request_id,
                trace_id=g.trace_id,
                method=request.method,
                path=request.path,
                remote_addr=request.remote_addr
            )

        @self.app.after_request
        def after_request(response):
            duration = time.time() - g.start_time
            endpoint = request.endpoint or 'unknown'

            # Метрики
            self.request_count.labels(
                service=self.service_name,
                method=request.method,
                endpoint=endpoint,
                status=response.status_code
            ).inc()

            self.request_duration.labels(
                service=self.service_name,
                method=request.method,
                endpoint=endpoint
            ).observe(duration)

            self.request_in_progress.labels(
                service=self.service_name,
                method=request.method,
                endpoint=endpoint
            ).dec()

            # Логирование
            log_data = {
                "request_id": g.request_id,
                "trace_id": g.trace_id,
                "span_id": g.span_id,
                "method": request.method,
                "path": request.path,
                "status_code": response.status_code,
                "duration_ms": round(duration * 1000, 2),
                "content_length": response.content_length
            }

            if response.status_code >= 400:
                self.logger.error("request_completed", **log_data)
            else:
                self.logger.info("request_completed", **log_data)

            # Добавляем trace headers в ответ
            response.headers['X-Request-ID'] = g.request_id
            response.headers['X-Trace-ID'] = g.trace_id

            return response

        @self.app.errorhandler(Exception)
        def handle_exception(e):
            self.logger.exception(
                "unhandled_exception",
                request_id=getattr(g, 'request_id', 'unknown'),
                trace_id=getattr(g, 'trace_id', 'unknown'),
                error=str(e)
            )
            return jsonify({"error": "Internal Server Error"}), 500

        # Эндпоинт для метрик Prometheus
        @self.app.route('/metrics')
        def metrics():
            return generate_latest(), 200, {'Content-Type': CONTENT_TYPE_LATEST}

        # Health check эндпоинты
        @self.app.route('/health')
        def health():
            return jsonify({"status": "healthy", "service": self.service_name})

        @self.app.route('/ready')
        def ready():
            return jsonify({"status": "ready", "service": self.service_name})

    def record_business_event(self, event_type: str, status: str = "success"):
        """Запись бизнес-события"""
        self.business_events.labels(
            service=self.service_name,
            event_type=event_type,
            status=status
        ).inc()

        self.logger.info(
            "business_event",
            event_type=event_type,
            status=status,
            trace_id=getattr(g, 'trace_id', 'unknown')
        )

    def run(self):
        """Запуск сервиса"""
        self.logger.info(
            "service_starting",
            service=self.service_name,
            port=self.port
        )
        self.app.run(host='0.0.0.0', port=self.port)
```

### API Gateway

```python
# services/api-gateway/main.py
import os
import requests
from flask import request, jsonify
from opentelemetry import trace
from common.base_service import ObservableService


class APIGateway(ObservableService):
    def __init__(self):
        super().__init__('api-gateway', 8080)

        self.order_service_url = os.environ.get('ORDER_SERVICE_URL', 'http://localhost:8081')
        self.product_service_url = os.environ.get('PRODUCT_SERVICE_URL', 'http://localhost:8082')
        self.payment_service_url = os.environ.get('PAYMENT_SERVICE_URL', 'http://localhost:8083')

        self._register_routes()

    def _register_routes(self):
        @self.app.route('/api/orders', methods=['POST'])
        def create_order():
            with self.tracer.start_as_current_span("create_order") as span:
                data = request.get_json()
                span.set_attribute("order.product_id", data.get('product_id'))
                span.set_attribute("order.quantity", data.get('quantity', 1))

                # Проверяем продукт
                product_response = requests.get(
                    f"{self.product_service_url}/products/{data.get('product_id')}"
                )

                if product_response.status_code != 200:
                    self.record_business_event("order_creation", "product_not_found")
                    span.set_attribute("error", True)
                    return jsonify({"error": "Product not found"}), 404

                product = product_response.json()

                # Создаем заказ
                order_response = requests.post(
                    f"{self.order_service_url}/orders",
                    json={
                        "product_id": data.get('product_id'),
                        "product_name": product.get('name'),
                        "quantity": data.get('quantity', 1),
                        "total_price": product.get('price') * data.get('quantity', 1)
                    }
                )

                if order_response.status_code != 201:
                    self.record_business_event("order_creation", "order_failed")
                    span.set_attribute("error", True)
                    return jsonify({"error": "Failed to create order"}), 500

                order = order_response.json()

                # Обрабатываем платеж
                payment_response = requests.post(
                    f"{self.payment_service_url}/payments",
                    json={
                        "order_id": order.get('order_id'),
                        "amount": order.get('total_price')
                    }
                )

                if payment_response.status_code != 200:
                    self.record_business_event("order_creation", "payment_failed")
                    span.set_attribute("error", True)
                    # В реальном приложении здесь нужна компенсация
                    return jsonify({"error": "Payment failed", "order": order}), 402

                payment = payment_response.json()

                self.record_business_event("order_creation", "success")
                span.set_attribute("order.id", order.get('order_id'))

                return jsonify({
                    "order": order,
                    "payment": payment
                }), 201

        @self.app.route('/api/products', methods=['GET'])
        def list_products():
            with self.tracer.start_as_current_span("list_products"):
                response = requests.get(f"{self.product_service_url}/products")
                self.record_business_event("product_list", "success")
                return jsonify(response.json())

        @self.app.route('/api/orders/<order_id>', methods=['GET'])
        def get_order(order_id):
            with self.tracer.start_as_current_span("get_order") as span:
                span.set_attribute("order.id", order_id)
                response = requests.get(f"{self.order_service_url}/orders/{order_id}")

                if response.status_code == 404:
                    self.record_business_event("order_lookup", "not_found")
                    return jsonify({"error": "Order not found"}), 404

                self.record_business_event("order_lookup", "success")
                return jsonify(response.json())


if __name__ == '__main__':
    gateway = APIGateway()
    gateway.run()
```

### Order Service

```python
# services/order-service/main.py
import os
import uuid
import time
import random
from flask import request, jsonify
from opentelemetry import trace
from common.base_service import ObservableService


class OrderService(ObservableService):
    def __init__(self):
        super().__init__('order-service', 8081)
        self.orders = {}  # В реальности - база данных
        self._register_routes()

    def _register_routes(self):
        @self.app.route('/orders', methods=['POST'])
        def create_order():
            with self.tracer.start_as_current_span("create_order") as span:
                data = request.get_json()

                # Симуляция задержки обработки
                processing_time = random.uniform(0.1, 0.3)
                time.sleep(processing_time)

                order_id = str(uuid.uuid4())
                order = {
                    "order_id": order_id,
                    "product_id": data.get('product_id'),
                    "product_name": data.get('product_name'),
                    "quantity": data.get('quantity'),
                    "total_price": data.get('total_price'),
                    "status": "created",
                    "created_at": time.time()
                }

                self.orders[order_id] = order

                span.set_attribute("order.id", order_id)
                span.set_attribute("order.processing_time_ms", round(processing_time * 1000, 2))

                self.record_business_event("order_created", "success")
                self.logger.info(
                    "order_created",
                    order_id=order_id,
                    product_id=data.get('product_id'),
                    total_price=data.get('total_price')
                )

                return jsonify(order), 201

        @self.app.route('/orders/<order_id>', methods=['GET'])
        def get_order(order_id):
            with self.tracer.start_as_current_span("get_order") as span:
                span.set_attribute("order.id", order_id)

                order = self.orders.get(order_id)

                if not order:
                    self.logger.warning("order_not_found", order_id=order_id)
                    return jsonify({"error": "Order not found"}), 404

                return jsonify(order)

        @self.app.route('/orders', methods=['GET'])
        def list_orders():
            with self.tracer.start_as_current_span("list_orders"):
                return jsonify(list(self.orders.values()))


if __name__ == '__main__':
    service = OrderService()
    service.run()
```

### Product Service

```python
# services/product-service/main.py
import os
import random
import time
from flask import request, jsonify
from opentelemetry import trace
from common.base_service import ObservableService


class ProductService(ObservableService):
    def __init__(self):
        super().__init__('product-service', 8082)

        # Демо-продукты
        self.products = {
            "1": {"id": "1", "name": "Laptop", "price": 999.99, "stock": 50},
            "2": {"id": "2", "name": "Phone", "price": 699.99, "stock": 100},
            "3": {"id": "3", "name": "Tablet", "price": 449.99, "stock": 75},
            "4": {"id": "4", "name": "Watch", "price": 299.99, "stock": 200},
            "5": {"id": "5", "name": "Headphones", "price": 199.99, "stock": 150},
        }

        self._register_routes()

    def _register_routes(self):
        @self.app.route('/products', methods=['GET'])
        def list_products():
            with self.tracer.start_as_current_span("list_products") as span:
                # Симуляция запроса к базе данных
                time.sleep(random.uniform(0.01, 0.05))

                span.set_attribute("products.count", len(self.products))
                self.record_business_event("products_listed", "success")

                return jsonify(list(self.products.values()))

        @self.app.route('/products/<product_id>', methods=['GET'])
        def get_product(product_id):
            with self.tracer.start_as_current_span("get_product") as span:
                span.set_attribute("product.id", product_id)

                # Симуляция запроса к базе данных
                time.sleep(random.uniform(0.01, 0.05))

                product = self.products.get(product_id)

                if not product:
                    self.logger.warning("product_not_found", product_id=product_id)
                    self.record_business_event("product_lookup", "not_found")
                    return jsonify({"error": "Product not found"}), 404

                span.set_attribute("product.name", product['name'])
                span.set_attribute("product.price", product['price'])

                self.record_business_event("product_lookup", "success")
                return jsonify(product)

        @self.app.route('/products/<product_id>/stock', methods=['GET'])
        def check_stock(product_id):
            with self.tracer.start_as_current_span("check_stock") as span:
                span.set_attribute("product.id", product_id)

                product = self.products.get(product_id)

                if not product:
                    return jsonify({"error": "Product not found"}), 404

                span.set_attribute("product.stock", product['stock'])
                return jsonify({"product_id": product_id, "stock": product['stock']})


if __name__ == '__main__':
    service = ProductService()
    service.run()
```

### Payment Service

```python
# services/payment-service/main.py
import os
import uuid
import random
import time
from flask import request, jsonify
from opentelemetry import trace
from common.base_service import ObservableService


class PaymentService(ObservableService):
    def __init__(self):
        super().__init__('payment-service', 8083)
        self.failure_rate = float(os.environ.get('FAILURE_RATE', 0.1))
        self.payments = {}
        self._register_routes()

    def _register_routes(self):
        @self.app.route('/payments', methods=['POST'])
        def process_payment():
            with self.tracer.start_as_current_span("process_payment") as span:
                data = request.get_json()
                order_id = data.get('order_id')
                amount = data.get('amount')

                span.set_attribute("payment.order_id", order_id)
                span.set_attribute("payment.amount", amount)

                # Симуляция обработки платежа
                processing_time = random.uniform(0.2, 0.5)
                time.sleep(processing_time)

                # Симуляция ошибок платежа
                if random.random() < self.failure_rate:
                    self.logger.error(
                        "payment_failed",
                        order_id=order_id,
                        amount=amount,
                        reason="Payment processor declined"
                    )
                    self.record_business_event("payment_processed", "failed")
                    span.set_attribute("error", True)
                    span.set_attribute("payment.error_reason", "Payment processor declined")
                    return jsonify({
                        "error": "Payment declined",
                        "order_id": order_id
                    }), 402

                payment_id = str(uuid.uuid4())
                payment = {
                    "payment_id": payment_id,
                    "order_id": order_id,
                    "amount": amount,
                    "status": "completed",
                    "processing_time_ms": round(processing_time * 1000, 2)
                }

                self.payments[payment_id] = payment

                span.set_attribute("payment.id", payment_id)
                span.set_attribute("payment.processing_time_ms", payment['processing_time_ms'])

                self.logger.info(
                    "payment_completed",
                    payment_id=payment_id,
                    order_id=order_id,
                    amount=amount,
                    processing_time_ms=payment['processing_time_ms']
                )

                self.record_business_event("payment_processed", "success")
                return jsonify(payment)

        @self.app.route('/payments/<payment_id>', methods=['GET'])
        def get_payment(payment_id):
            with self.tracer.start_as_current_span("get_payment") as span:
                span.set_attribute("payment.id", payment_id)

                payment = self.payments.get(payment_id)

                if not payment:
                    return jsonify({"error": "Payment not found"}), 404

                return jsonify(payment)


if __name__ == '__main__':
    service = PaymentService()
    service.run()
```

### Dockerfile для сервисов

```dockerfile
# services/api-gateway/Dockerfile (аналогично для других сервисов)
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

ENV PYTHONPATH=/app
ENV PYTHONUNBUFFERED=1

EXPOSE 8080

CMD ["python", "main.py"]
```

### Requirements.txt

```
# services/api-gateway/requirements.txt
flask==2.3.3
requests==2.31.0
prometheus-client==0.17.1
opentelemetry-api==1.20.0
opentelemetry-sdk==1.20.0
opentelemetry-exporter-jaeger==1.20.0
opentelemetry-instrumentation-flask==0.41b0
opentelemetry-instrumentation-requests==0.41b0
structlog==23.2.0
```

## Конфигурация Prometheus

```yaml
# prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: 'observability-demo'
    env: 'development'

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

rule_files:
  - "/etc/prometheus/rules/*.yml"

scrape_configs:
  # Prometheus self-monitoring
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Application services
  - job_name: 'api-gateway'
    static_configs:
      - targets: ['api-gateway:8080']
    metrics_path: /metrics

  - job_name: 'order-service'
    static_configs:
      - targets: ['order-service:8081']
    metrics_path: /metrics

  - job_name: 'product-service'
    static_configs:
      - targets: ['product-service:8082']
    metrics_path: /metrics

  - job_name: 'payment-service'
    static_configs:
      - targets: ['payment-service:8083']
    metrics_path: /metrics

  # Alertmanager
  - job_name: 'alertmanager'
    static_configs:
      - targets: ['alertmanager:9093']

  # Jaeger
  - job_name: 'jaeger'
    static_configs:
      - targets: ['jaeger:14269']

  # Loki
  - job_name: 'loki'
    static_configs:
      - targets: ['loki:3100']
```

## Правила алертинга

```yaml
# prometheus/rules/alerts.yml
groups:
  - name: service_health
    interval: 30s
    rules:
      # Высокий error rate
      - alert: HighErrorRate
        expr: |
          (
            sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
            /
            sum(rate(http_requests_total[5m])) by (service)
          ) * 100 > 5
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High error rate on {{ $labels.service }}"
          description: "Error rate is {{ printf \"%.2f\" $value }}% on {{ $labels.service }}"
          runbook_url: "https://wiki.example.com/runbooks/high-error-rate"

      # Высокая latency
      - alert: HighLatency
        expr: |
          histogram_quantile(0.95,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
          ) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High latency on {{ $labels.service }}"
          description: "P95 latency is {{ printf \"%.2f\" $value }}s on {{ $labels.service }}"

      # Сервис недоступен
      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.job }} is down"
          description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 1 minute"

  - name: business_metrics
    interval: 1m
    rules:
      # Много неудачных платежей
      - alert: HighPaymentFailureRate
        expr: |
          (
            sum(rate(business_events_total{service="payment-service",status="failed"}[10m]))
            /
            sum(rate(business_events_total{service="payment-service"}[10m]))
          ) * 100 > 15
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High payment failure rate"
          description: "Payment failure rate is {{ printf \"%.2f\" $value }}%"

      # Низкий объем заказов
      - alert: LowOrderVolume
        expr: |
          sum(rate(business_events_total{event_type="order_created",status="success"}[30m])) * 60 < 1
        for: 30m
        labels:
          severity: warning
        annotations:
          summary: "Low order volume"
          description: "Less than 1 order per minute in the last 30 minutes"

  - name: resource_monitoring
    interval: 30s
    rules:
      # Высокое количество активных запросов
      - alert: HighActiveRequests
        expr: http_requests_in_progress > 50
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High number of active requests on {{ $labels.service }}"
          description: "{{ $value }} requests currently in progress on {{ $labels.service }}"
```

## Конфигурация AlertManager

```yaml
# alertmanager/alertmanager.yml
global:
  smtp_smarthost: 'smtp.example.com:587'
  smtp_from: 'alertmanager@example.com'
  smtp_auth_username: 'alertmanager@example.com'
  smtp_auth_password: 'password'

  # Slack webhook (опционально)
  slack_api_url: 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'

route:
  group_by: ['alertname', 'service']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'default-receiver'

  routes:
    # Critical алерты - немедленное уведомление
    - match:
        severity: critical
      receiver: 'critical-receiver'
      group_wait: 10s
      repeat_interval: 1h
      continue: true

    # Payment-related алерты
    - match:
        service: payment-service
      receiver: 'payment-team'
      group_wait: 30s

receivers:
  - name: 'default-receiver'
    webhook_configs:
      - url: 'http://localhost:8090/webhook'
        send_resolved: true

  - name: 'critical-receiver'
    slack_configs:
      - channel: '#alerts-critical'
        username: 'AlertManager'
        icon_emoji: ':fire:'
        title: '{{ .GroupLabels.alertname }}'
        text: >-
          {{ range .Alerts }}
          *Alert:* {{ .Annotations.summary }}
          *Description:* {{ .Annotations.description }}
          *Severity:* {{ .Labels.severity }}
          {{ end }}
        send_resolved: true

  - name: 'payment-team'
    email_configs:
      - to: 'payment-team@example.com'
        send_resolved: true
    slack_configs:
      - channel: '#payment-alerts'
        send_resolved: true

inhibit_rules:
  # Подавляем warning если есть critical для того же сервиса
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['service', 'alertname']

templates:
  - '/etc/alertmanager/templates/*.tmpl'
```

## Конфигурация Loki

```yaml
# loki/loki-config.yml
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

ruler:
  alertmanager_url: http://alertmanager:9093

limits_config:
  enforce_metric_name: false
  reject_old_samples: true
  reject_old_samples_max_age: 168h
  ingestion_rate_mb: 10
  ingestion_burst_size_mb: 20

chunk_store_config:
  max_look_back_period: 0s

table_manager:
  retention_deletes_enabled: false
  retention_period: 0s
```

## Конфигурация Promtail

```yaml
# promtail/promtail-config.yml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  # Docker контейнеры
  - job_name: containers
    static_configs:
      - targets:
          - localhost
        labels:
          job: containerlogs
          __path__: /var/lib/docker/containers/*/*log

    pipeline_stages:
      # Парсим JSON логи Docker
      - json:
          expressions:
            output: log
            stream: stream
            attrs: attrs

      # Извлекаем контейнер тег
      - json:
          expressions:
            tag: attrs.tag
          source: attrs

      # Парсим JSON из самого лога (наши структурированные логи)
      - json:
          expressions:
            level: level
            service: logger
            event: event
            trace_id: trace_id
            request_id: request_id
          source: output

      # Устанавливаем labels
      - labels:
          stream:
          level:
          service:
          tag:

      # Устанавливаем timestamp
      - timestamp:
          source: timestamp
          format: RFC3339Nano

      # Финальный вывод
      - output:
          source: output
```

## Конфигурация источников данных Grafana

```yaml
# grafana/provisioning/datasources/datasources.yml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false
    jsonData:
      httpMethod: POST
      manageAlerts: true
      prometheusType: Prometheus
      prometheusVersion: 2.47.0

  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    editable: false
    jsonData:
      maxLines: 1000
      derivedFields:
        - name: TraceID
          matcherRegex: '"trace_id":"([a-f0-9]+)"'
          url: 'http://localhost:16686/trace/$${__value.raw}'
          datasourceUid: jaeger

  - name: Jaeger
    type: jaeger
    uid: jaeger
    access: proxy
    url: http://jaeger:16686
    editable: false
    jsonData:
      tracesToLogs:
        datasourceUid: loki
        tags: ['service']
        mappedTags: [{ key: 'service.name', value: 'service' }]
        mapTagNamesEnabled: true
        spanStartTimeShift: '-1h'
        spanEndTimeShift: '1h'
        filterByTraceID: true
        filterBySpanID: false

  - name: Alertmanager
    type: alertmanager
    access: proxy
    url: http://alertmanager:9093
    editable: false
    jsonData:
      implementation: prometheus
```

## Dashboard JSON

```json
{
  "annotations": {
    "list": [
      {
        "builtIn": 1,
        "datasource": {
          "type": "grafana",
          "uid": "-- Grafana --"
        },
        "enable": true,
        "hide": true,
        "iconColor": "rgba(0, 211, 255, 1)",
        "name": "Annotations & Alerts",
        "type": "dashboard"
      }
    ]
  },
  "description": "Complete observability dashboard for microservices",
  "editable": true,
  "fiscalYearStartMonth": 0,
  "graphTooltip": 1,
  "id": null,
  "links": [],
  "liveNow": false,
  "panels": [
    {
      "collapsed": false,
      "gridPos": {
        "h": 1,
        "w": 24,
        "x": 0,
        "y": 0
      },
      "id": 1,
      "panels": [],
      "title": "Overview",
      "type": "row"
    },
    {
      "datasource": {
        "type": "prometheus",
        "uid": "prometheus"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "palette-classic"
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              },
              {
                "color": "red",
                "value": 80
              }
            ]
          },
          "unit": "reqps"
        }
      },
      "gridPos": {
        "h": 6,
        "w": 6,
        "x": 0,
        "y": 1
      },
      "id": 2,
      "options": {
        "colorMode": "value",
        "graphMode": "area",
        "justifyMode": "auto",
        "orientation": "auto",
        "reduceOptions": {
          "calcs": ["lastNotNull"],
          "fields": "",
          "values": false
        },
        "textMode": "auto"
      },
      "pluginVersion": "10.1.0",
      "targets": [
        {
          "datasource": {
            "type": "prometheus",
            "uid": "prometheus"
          },
          "expr": "sum(rate(http_requests_total[5m]))",
          "legendFormat": "Total RPS",
          "refId": "A"
        }
      ],
      "title": "Total Request Rate",
      "type": "stat"
    },
    {
      "datasource": {
        "type": "prometheus",
        "uid": "prometheus"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "thresholds"
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              },
              {
                "color": "yellow",
                "value": 1
              },
              {
                "color": "red",
                "value": 5
              }
            ]
          },
          "unit": "percent"
        }
      },
      "gridPos": {
        "h": 6,
        "w": 6,
        "x": 6,
        "y": 1
      },
      "id": 3,
      "options": {
        "colorMode": "value",
        "graphMode": "area",
        "justifyMode": "auto",
        "orientation": "auto",
        "reduceOptions": {
          "calcs": ["lastNotNull"],
          "fields": "",
          "values": false
        }
      },
      "targets": [
        {
          "datasource": {
            "type": "prometheus",
            "uid": "prometheus"
          },
          "expr": "(sum(rate(http_requests_total{status=~\"5..\"}[5m])) / sum(rate(http_requests_total[5m]))) * 100",
          "legendFormat": "Error Rate %",
          "refId": "A"
        }
      ],
      "title": "Error Rate",
      "type": "stat"
    },
    {
      "datasource": {
        "type": "prometheus",
        "uid": "prometheus"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "thresholds"
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              },
              {
                "color": "yellow",
                "value": 0.5
              },
              {
                "color": "red",
                "value": 1
              }
            ]
          },
          "unit": "s"
        }
      },
      "gridPos": {
        "h": 6,
        "w": 6,
        "x": 12,
        "y": 1
      },
      "id": 4,
      "options": {
        "colorMode": "value",
        "graphMode": "area",
        "justifyMode": "auto",
        "orientation": "auto",
        "reduceOptions": {
          "calcs": ["lastNotNull"],
          "fields": "",
          "values": false
        }
      },
      "targets": [
        {
          "datasource": {
            "type": "prometheus",
            "uid": "prometheus"
          },
          "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))",
          "legendFormat": "P95 Latency",
          "refId": "A"
        }
      ],
      "title": "P95 Latency",
      "type": "stat"
    },
    {
      "datasource": {
        "type": "prometheus",
        "uid": "prometheus"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "thresholds"
          },
          "mappings": [
            {
              "options": {
                "0": {
                  "color": "red",
                  "index": 1,
                  "text": "DOWN"
                },
                "1": {
                  "color": "green",
                  "index": 0,
                  "text": "UP"
                }
              },
              "type": "value"
            }
          ],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "red",
                "value": null
              },
              {
                "color": "green",
                "value": 1
              }
            ]
          }
        }
      },
      "gridPos": {
        "h": 6,
        "w": 6,
        "x": 18,
        "y": 1
      },
      "id": 5,
      "options": {
        "colorMode": "value",
        "graphMode": "none",
        "justifyMode": "auto",
        "orientation": "vertical",
        "reduceOptions": {
          "calcs": ["lastNotNull"],
          "fields": "",
          "values": false
        }
      },
      "targets": [
        {
          "datasource": {
            "type": "prometheus",
            "uid": "prometheus"
          },
          "expr": "up{job=~\"api-gateway|order-service|product-service|payment-service\"}",
          "legendFormat": "{{ job }}",
          "refId": "A"
        }
      ],
      "title": "Service Health",
      "type": "stat"
    },
    {
      "collapsed": false,
      "gridPos": {
        "h": 1,
        "w": 24,
        "x": 0,
        "y": 7
      },
      "id": 6,
      "panels": [],
      "title": "Request Metrics",
      "type": "row"
    },
    {
      "datasource": {
        "type": "prometheus",
        "uid": "prometheus"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "palette-classic"
          },
          "custom": {
            "axisCenteredZero": false,
            "axisColorMode": "text",
            "axisLabel": "",
            "axisPlacement": "auto",
            "barAlignment": 0,
            "drawStyle": "line",
            "fillOpacity": 10,
            "gradientMode": "none",
            "hideFrom": {
              "legend": false,
              "tooltip": false,
              "viz": false
            },
            "lineInterpolation": "smooth",
            "lineWidth": 1,
            "pointSize": 5,
            "scaleDistribution": {
              "type": "linear"
            },
            "showPoints": "never",
            "spanNulls": false,
            "stacking": {
              "group": "A",
              "mode": "none"
            },
            "thresholdsStyle": {
              "mode": "off"
            }
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              }
            ]
          },
          "unit": "reqps"
        }
      },
      "gridPos": {
        "h": 8,
        "w": 12,
        "x": 0,
        "y": 8
      },
      "id": 7,
      "options": {
        "legend": {
          "calcs": ["mean", "max"],
          "displayMode": "table",
          "placement": "bottom",
          "showLegend": true
        },
        "tooltip": {
          "mode": "multi",
          "sort": "desc"
        }
      },
      "targets": [
        {
          "datasource": {
            "type": "prometheus",
            "uid": "prometheus"
          },
          "expr": "sum(rate(http_requests_total[5m])) by (service)",
          "legendFormat": "{{ service }}",
          "refId": "A"
        }
      ],
      "title": "Request Rate by Service",
      "type": "timeseries"
    },
    {
      "datasource": {
        "type": "prometheus",
        "uid": "prometheus"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "palette-classic"
          },
          "custom": {
            "axisCenteredZero": false,
            "axisColorMode": "text",
            "axisLabel": "",
            "axisPlacement": "auto",
            "barAlignment": 0,
            "drawStyle": "line",
            "fillOpacity": 10,
            "gradientMode": "none",
            "hideFrom": {
              "legend": false,
              "tooltip": false,
              "viz": false
            },
            "lineInterpolation": "smooth",
            "lineWidth": 1,
            "pointSize": 5,
            "scaleDistribution": {
              "type": "linear"
            },
            "showPoints": "never",
            "spanNulls": false,
            "stacking": {
              "group": "A",
              "mode": "none"
            },
            "thresholdsStyle": {
              "mode": "line+area"
            }
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              },
              {
                "color": "yellow",
                "value": 0.5
              },
              {
                "color": "red",
                "value": 1
              }
            ]
          },
          "unit": "s"
        }
      },
      "gridPos": {
        "h": 8,
        "w": 12,
        "x": 12,
        "y": 8
      },
      "id": 8,
      "options": {
        "legend": {
          "calcs": ["mean", "max", "p99"],
          "displayMode": "table",
          "placement": "bottom",
          "showLegend": true
        },
        "tooltip": {
          "mode": "multi",
          "sort": "desc"
        }
      },
      "targets": [
        {
          "datasource": {
            "type": "prometheus",
            "uid": "prometheus"
          },
          "expr": "histogram_quantile(0.50, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service))",
          "legendFormat": "{{ service }} P50",
          "refId": "A"
        },
        {
          "datasource": {
            "type": "prometheus",
            "uid": "prometheus"
          },
          "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service))",
          "legendFormat": "{{ service }} P95",
          "refId": "B"
        },
        {
          "datasource": {
            "type": "prometheus",
            "uid": "prometheus"
          },
          "expr": "histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service))",
          "legendFormat": "{{ service }} P99",
          "refId": "C"
        }
      ],
      "title": "Latency Distribution by Service",
      "type": "timeseries"
    },
    {
      "collapsed": false,
      "gridPos": {
        "h": 1,
        "w": 24,
        "x": 0,
        "y": 16
      },
      "id": 9,
      "panels": [],
      "title": "Business Metrics",
      "type": "row"
    },
    {
      "datasource": {
        "type": "prometheus",
        "uid": "prometheus"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "palette-classic"
          },
          "custom": {
            "axisCenteredZero": false,
            "axisColorMode": "text",
            "axisLabel": "",
            "axisPlacement": "auto",
            "barAlignment": 0,
            "drawStyle": "bars",
            "fillOpacity": 80,
            "gradientMode": "none",
            "hideFrom": {
              "legend": false,
              "tooltip": false,
              "viz": false
            },
            "lineInterpolation": "linear",
            "lineWidth": 1,
            "pointSize": 5,
            "scaleDistribution": {
              "type": "linear"
            },
            "showPoints": "never",
            "spanNulls": false,
            "stacking": {
              "group": "A",
              "mode": "normal"
            },
            "thresholdsStyle": {
              "mode": "off"
            }
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              }
            ]
          },
          "unit": "short"
        }
      },
      "gridPos": {
        "h": 8,
        "w": 12,
        "x": 0,
        "y": 17
      },
      "id": 10,
      "options": {
        "legend": {
          "calcs": ["sum"],
          "displayMode": "table",
          "placement": "bottom",
          "showLegend": true
        },
        "tooltip": {
          "mode": "multi",
          "sort": "desc"
        }
      },
      "targets": [
        {
          "datasource": {
            "type": "prometheus",
            "uid": "prometheus"
          },
          "expr": "sum(increase(business_events_total{event_type=\"order_created\"}[1h])) by (status)",
          "legendFormat": "Orders {{ status }}",
          "refId": "A"
        }
      ],
      "title": "Orders Created (Hourly)",
      "type": "timeseries"
    },
    {
      "datasource": {
        "type": "prometheus",
        "uid": "prometheus"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "palette-classic"
          },
          "custom": {
            "axisCenteredZero": false,
            "axisColorMode": "text",
            "axisLabel": "",
            "axisPlacement": "auto",
            "barAlignment": 0,
            "drawStyle": "bars",
            "fillOpacity": 80,
            "gradientMode": "none",
            "hideFrom": {
              "legend": false,
              "tooltip": false,
              "viz": false
            },
            "lineInterpolation": "linear",
            "lineWidth": 1,
            "pointSize": 5,
            "scaleDistribution": {
              "type": "linear"
            },
            "showPoints": "never",
            "spanNulls": false,
            "stacking": {
              "group": "A",
              "mode": "normal"
            },
            "thresholdsStyle": {
              "mode": "off"
            }
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              }
            ]
          },
          "unit": "short"
        }
      },
      "gridPos": {
        "h": 8,
        "w": 12,
        "x": 12,
        "y": 17
      },
      "id": 11,
      "options": {
        "legend": {
          "calcs": ["sum"],
          "displayMode": "table",
          "placement": "bottom",
          "showLegend": true
        },
        "tooltip": {
          "mode": "multi",
          "sort": "desc"
        }
      },
      "targets": [
        {
          "datasource": {
            "type": "prometheus",
            "uid": "prometheus"
          },
          "expr": "sum(increase(business_events_total{event_type=\"payment_processed\"}[1h])) by (status)",
          "legendFormat": "Payments {{ status }}",
          "refId": "A"
        }
      ],
      "title": "Payments Processed (Hourly)",
      "type": "timeseries"
    },
    {
      "collapsed": false,
      "gridPos": {
        "h": 1,
        "w": 24,
        "x": 0,
        "y": 25
      },
      "id": 12,
      "panels": [],
      "title": "Logs",
      "type": "row"
    },
    {
      "datasource": {
        "type": "loki",
        "uid": "loki"
      },
      "gridPos": {
        "h": 10,
        "w": 24,
        "x": 0,
        "y": 26
      },
      "id": 13,
      "options": {
        "dedupStrategy": "none",
        "enableLogDetails": true,
        "prettifyLogMessage": false,
        "showCommonLabels": false,
        "showLabels": false,
        "showTime": true,
        "sortOrder": "Descending",
        "wrapLogMessage": false
      },
      "targets": [
        {
          "datasource": {
            "type": "loki",
            "uid": "loki"
          },
          "expr": "{job=\"containerlogs\"} |= \"error\" or {job=\"containerlogs\"} |= \"Error\" or {job=\"containerlogs\"} | json | level = \"error\"",
          "legendFormat": "",
          "refId": "A"
        }
      ],
      "title": "Error Logs",
      "type": "logs"
    }
  ],
  "refresh": "10s",
  "schemaVersion": 38,
  "style": "dark",
  "tags": ["observability", "microservices"],
  "templating": {
    "list": [
      {
        "current": {
          "selected": true,
          "text": ["All"],
          "value": ["$__all"]
        },
        "datasource": {
          "type": "prometheus",
          "uid": "prometheus"
        },
        "definition": "label_values(http_requests_total, service)",
        "hide": 0,
        "includeAll": true,
        "label": "Service",
        "multi": true,
        "name": "service",
        "options": [],
        "query": {
          "query": "label_values(http_requests_total, service)",
          "refId": "StandardVariableQuery"
        },
        "refresh": 1,
        "regex": "",
        "skipUrlSync": false,
        "sort": 1,
        "type": "query"
      }
    ]
  },
  "time": {
    "from": "now-1h",
    "to": "now"
  },
  "timepicker": {},
  "timezone": "",
  "title": "Microservices Observability",
  "uid": "microservices-observability",
  "version": 1,
  "weekStart": ""
}
```

## Скрипт нагрузочного тестирования (k6)

```javascript
// k6/load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

const errorRate = new Rate('errors');
const orderDuration = new Trend('order_duration');

export const options = {
  stages: [
    { duration: '30s', target: 10 },  // Разогрев
    { duration: '1m', target: 50 },   // Рампа до 50 пользователей
    { duration: '3m', target: 50 },   // Стабильная нагрузка
    { duration: '1m', target: 100 },  // Пиковая нагрузка
    { duration: '30s', target: 0 },   // Охлаждение
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],
    errors: ['rate<0.1'],
  },
};

const BASE_URL = 'http://api-gateway:8080';

export default function () {
  // Получаем список продуктов
  const productsRes = http.get(`${BASE_URL}/api/products`);
  check(productsRes, {
    'products status 200': (r) => r.status === 200,
  });

  if (productsRes.status === 200) {
    const products = productsRes.json();

    // Выбираем случайный продукт
    const randomProduct = products[Math.floor(Math.random() * products.length)];

    // Создаем заказ
    const orderPayload = JSON.stringify({
      product_id: randomProduct.id,
      quantity: Math.floor(Math.random() * 3) + 1,
    });

    const orderRes = http.post(`${BASE_URL}/api/orders`, orderPayload, {
      headers: { 'Content-Type': 'application/json' },
    });

    orderDuration.add(orderRes.timings.duration);

    const orderSuccess = check(orderRes, {
      'order status 201 or 402': (r) => r.status === 201 || r.status === 402,
    });

    errorRate.add(!orderSuccess);

    // Если заказ создан, проверяем его
    if (orderRes.status === 201) {
      const order = orderRes.json().order;

      sleep(1);

      const getOrderRes = http.get(`${BASE_URL}/api/orders/${order.order_id}`);
      check(getOrderRes, {
        'get order status 200': (r) => r.status === 200,
      });
    }
  }

  sleep(Math.random() * 2 + 1);
}
```

## Запуск проекта

### 1. Создание структуры проекта

```bash
mkdir -p observability-project/{services/{api-gateway,order-service,product-service,payment-service,common},prometheus/rules,alertmanager,grafana/provisioning/{datasources,dashboards},loki,promtail,k6}
```

### 2. Копирование файлов

Скопируйте все файлы конфигурации в соответствующие директории.

### 3. Запуск стека

```bash
cd observability-project

# Запуск всех сервисов
docker-compose up -d

# Проверка статуса
docker-compose ps

# Просмотр логов
docker-compose logs -f
```

### 4. Доступ к интерфейсам

| Сервис | URL | Учетные данные |
|--------|-----|----------------|
| API Gateway | http://localhost:8080 | - |
| Prometheus | http://localhost:9090 | - |
| Grafana | http://localhost:3000 | admin/admin |
| Jaeger UI | http://localhost:16686 | - |
| AlertManager | http://localhost:9093 | - |

### 5. Тестирование API

```bash
# Получить список продуктов
curl http://localhost:8080/api/products

# Создать заказ
curl -X POST http://localhost:8080/api/orders \
  -H "Content-Type: application/json" \
  -d '{"product_id": "1", "quantity": 2}'

# Получить заказ
curl http://localhost:8080/api/orders/{order_id}
```

### 6. Запуск нагрузочного теста

```bash
docker-compose exec k6 k6 run /scripts/load-test.js
```

## Что наблюдать

### В Prometheus (http://localhost:9090)

1. **Метрики запросов**: `http_requests_total`
2. **Latency**: `http_request_duration_seconds`
3. **Бизнес-метрики**: `business_events_total`
4. **Health checks**: `up`

### В Grafana (http://localhost:3000)

1. Дашборд "Microservices Observability"
2. Связь метрик с логами через Loki
3. Переход к трейсам через Jaeger

### В Jaeger (http://localhost:16686)

1. Распределенные трейсы между сервисами
2. Время выполнения каждого span
3. Зависимости между сервисами

### В AlertManager (http://localhost:9093)

1. Активные алерты
2. История алертов
3. Silences (подавленные алерты)

## Упражнения для практики

### Упражнение 1: Создание алерта

Добавьте новый алерт для мониторинга медленных запросов к конкретному эндпоинту.

### Упражнение 2: Кастомный дашборд

Создайте дашборд для мониторинга конкретного бизнес-процесса (например, checkout flow).

### Упражнение 3: Добавление нового сервиса

Добавьте сервис инвентаря (inventory-service) с полной инструментацией.

### Упражнение 4: Корреляция данных

Используя Grafana Explore, найдите конкретный запрос:
1. Начните с метрик (медленный запрос)
2. Перейдите к логам по trace_id
3. Откройте полный трейс в Jaeger

### Упражнение 5: Хаос-инжиниринг

1. Увеличьте `FAILURE_RATE` для payment-service до 50%
2. Наблюдайте как срабатывают алерты
3. Изучите поведение системы под нагрузкой

## Лучшие практики

### 1. Организация метрик

- Используйте консистентные названия
- Добавляйте осмысленные labels
- Избегайте высокой кардинальности

### 2. Структурированное логирование

- Всегда включайте trace_id
- Используйте уровни логирования правильно
- Логируйте бизнес-события

### 3. Трассировка

- Инструментируйте все внешние вызовы
- Добавляйте полезные атрибуты к spans
- Сэмплируйте трейсы в продакшене

### 4. Алертинг

- Алертите на симптомы, не на причины
- Используйте runbook URLs
- Настройте правильные пороги

## Заключение

Этот практический проект демонстрирует полный стек наблюдаемости для микросервисной архитектуры. Он объединяет:

- **Метрики** (Prometheus) для количественного анализа
- **Логи** (Loki) для детального исследования
- **Трейсы** (Jaeger) для распределенной отладки
- **Алертинг** (AlertManager) для проактивного реагирования
- **Визуализацию** (Grafana) для единой точки наблюдения

Используйте этот проект как основу для построения наблюдаемости в ваших production-системах.
