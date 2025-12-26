# Performance Testing

## Введение

Performance Testing (тестирование производительности) — это практика оценки скорости, отзывчивости и стабильности системы под различными нагрузками. Цель — выявить узкие места до того, как они повлияют на пользователей в production.

## Типы тестирования производительности

```
┌─────────────────────────────────────────────────────────────────┐
│                   Типы Performance Testing                       │
├─────────────────────────────────────────────────────────────────┤
│ Load Testing        │ Нормальная ожидаемая нагрузка             │
│ Stress Testing      │ Нагрузка выше нормы (до точки отказа)     │
│ Spike Testing       │ Резкие всплески нагрузки                  │
│ Endurance Testing   │ Длительная нагрузка (часы/дни)            │
│ Scalability Testing │ Масштабируемость при росте                │
│ Volume Testing      │ Большие объёмы данных                     │
└─────────────────────────────────────────────────────────────────┘
```

## Locust — нагрузочное тестирование на Python

### Базовый пример

```python
# locustfile.py
from locust import HttpUser, task, between, events
from locust.runners import MasterRunner
import json
import random

class APIUser(HttpUser):
    """
    Симуляция пользователя API

    wait_time — время между запросами
    weight — относительная частота использования этого класса
    """
    wait_time = between(1, 3)  # 1-3 секунды между запросами
    weight = 1

    def on_start(self):
        """Выполняется при старте каждого пользователя"""
        # Авторизация
        response = self.client.post("/api/auth/login", json={
            "email": f"user{self.environment.runner.user_count}@test.com",
            "password": "password123"
        })
        if response.status_code == 200:
            self.token = response.json().get("token")
        else:
            self.token = None

    def on_stop(self):
        """Выполняется при остановке пользователя"""
        if self.token:
            self.client.post(
                "/api/auth/logout",
                headers={"Authorization": f"Bearer {self.token}"}
            )

    @task(3)  # Вес 3 — выполняется чаще
    def get_products(self):
        """Получение списка продуктов"""
        with self.client.get(
            "/api/products",
            headers=self._auth_headers(),
            catch_response=True
        ) as response:
            if response.status_code == 200:
                data = response.json()
                if len(data.get("items", [])) > 0:
                    response.success()
                else:
                    response.failure("Empty product list")
            else:
                response.failure(f"Status: {response.status_code}")

    @task(2)  # Вес 2
    def get_product_detail(self):
        """Получение детальной информации о продукте"""
        product_id = random.randint(1, 1000)
        self.client.get(
            f"/api/products/{product_id}",
            headers=self._auth_headers(),
            name="/api/products/:id"  # Группировка в статистике
        )

    @task(1)  # Вес 1 — выполняется реже
    def create_order(self):
        """Создание заказа"""
        order_data = {
            "product_id": random.randint(1, 1000),
            "quantity": random.randint(1, 5),
            "shipping_address": "123 Test St"
        }
        with self.client.post(
            "/api/orders",
            json=order_data,
            headers=self._auth_headers(),
            catch_response=True
        ) as response:
            if response.status_code == 201:
                response.success()
            elif response.status_code == 429:
                response.failure("Rate limited")
            else:
                response.failure(f"Order failed: {response.status_code}")

    def _auth_headers(self):
        if self.token:
            return {"Authorization": f"Bearer {self.token}"}
        return {}

# Запуск: locust -f locustfile.py --host=http://localhost:8000
# Web UI: http://localhost:8089
```

### Продвинутые сценарии

```python
from locust import HttpUser, task, between, SequentialTaskSet, events
from locust.runners import MasterRunner, WorkerRunner
import logging
import time

class UserBehavior(SequentialTaskSet):
    """
    Последовательный сценарий поведения пользователя

    Задачи выполняются в порядке определения
    """

    @task
    def browse_catalog(self):
        """Шаг 1: Просмотр каталога"""
        self.client.get("/api/categories")

        for _ in range(3):
            category_id = random.randint(1, 10)
            self.client.get(
                f"/api/categories/{category_id}/products",
                name="/api/categories/:id/products"
            )
            time.sleep(random.uniform(0.5, 2))

    @task
    def view_product(self):
        """Шаг 2: Просмотр товара"""
        product_id = random.randint(1, 100)
        self.client.get(
            f"/api/products/{product_id}",
            name="/api/products/:id"
        )
        self.client.get(
            f"/api/products/{product_id}/reviews",
            name="/api/products/:id/reviews"
        )

    @task
    def add_to_cart(self):
        """Шаг 3: Добавление в корзину"""
        self.client.post("/api/cart/items", json={
            "product_id": random.randint(1, 100),
            "quantity": 1
        })

    @task
    def checkout(self):
        """Шаг 4: Оформление заказа"""
        # Получаем корзину
        response = self.client.get("/api/cart")
        if response.status_code != 200:
            return

        # Оформляем заказ
        self.client.post("/api/orders", json={
            "shipping_address": "Test Address",
            "payment_method": "card"
        })

        # После checkout — прерываем и начинаем заново
        self.interrupt()

class EcommerceUser(HttpUser):
    tasks = [UserBehavior]
    wait_time = between(1, 5)

# Кастомные события для метрик
@events.request.add_listener
def on_request(request_type, name, response_time, response_length, exception, **kwargs):
    """Обработка каждого запроса"""
    if exception:
        logging.error(f"Request failed: {name} - {exception}")

@events.test_start.add_listener
def on_test_start(environment, **kwargs):
    """Выполняется при старте теста"""
    logging.info("Performance test starting...")

    if isinstance(environment.runner, MasterRunner):
        logging.info("Running in distributed mode (master)")

@events.test_stop.add_listener
def on_test_stop(environment, **kwargs):
    """Выполняется при остановке теста"""
    stats = environment.runner.stats
    logging.info(f"Total requests: {stats.total.num_requests}")
    logging.info(f"Failures: {stats.total.num_failures}")
    logging.info(f"Average response time: {stats.total.avg_response_time}ms")
```

### Distributed Load Testing

```python
# locustfile.py для распределённого тестирования

from locust import HttpUser, task, between, LoadTestShape
import math

class StepLoadShape(LoadTestShape):
    """
    Кастомная форма нагрузки — ступенчатое увеличение

    Пример: начинаем с 10 пользователей, каждые 2 минуты добавляем ещё 10
    """

    step_time = 120  # Секунд на каждый шаг
    step_load = 10   # Пользователей на шаг
    spawn_rate = 5   # Скорость добавления пользователей
    max_users = 100  # Максимум пользователей

    def tick(self):
        run_time = self.get_run_time()

        # Текущий шаг
        current_step = math.floor(run_time / self.step_time) + 1

        # Целевое количество пользователей
        target_users = min(current_step * self.step_load, self.max_users)

        return (target_users, self.spawn_rate)

class SpikeLoadShape(LoadTestShape):
    """
    Spike Testing — резкие всплески нагрузки
    """

    stages = [
        {"duration": 60, "users": 10, "spawn_rate": 10},    # Разогрев
        {"duration": 30, "users": 100, "spawn_rate": 50},   # Spike!
        {"duration": 60, "users": 10, "spawn_rate": 50},    # Восстановление
        {"duration": 30, "users": 100, "spawn_rate": 50},   # Второй spike
        {"duration": 60, "users": 10, "spawn_rate": 50},    # Восстановление
    ]

    def tick(self):
        run_time = self.get_run_time()

        elapsed = 0
        for stage in self.stages:
            elapsed += stage["duration"]
            if run_time < elapsed:
                return (stage["users"], stage["spawn_rate"])

        return None  # Тест завершён

# Запуск distributed:
# Master: locust -f locustfile.py --master --host=http://api.example.com
# Workers: locust -f locustfile.py --worker --master-host=master-ip
```

## k6 — современный инструмент нагрузочного тестирования

### Установка и базовый тест

```javascript
// k6_test.js
import http from 'k6/http';
import { check, sleep, group } from 'k6';
import { Rate, Trend, Counter } from 'k6/metrics';

// Кастомные метрики
const errorRate = new Rate('error_rate');
const orderDuration = new Trend('order_duration');
const ordersCreated = new Counter('orders_created');

// Конфигурация теста
export const options = {
  stages: [
    { duration: '1m', target: 10 },   // Разогрев
    { duration: '3m', target: 50 },   // Рампа до 50 пользователей
    { duration: '5m', target: 50 },   // Удержание
    { duration: '2m', target: 100 },  // Пиковая нагрузка
    { duration: '2m', target: 0 },    // Спад
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],    // 95% запросов < 500ms
    http_req_failed: ['rate<0.01'],       // < 1% ошибок
    error_rate: ['rate<0.05'],            // Кастомная метрика
  },
};

// Данные для теста
const testData = JSON.parse(open('./test_data.json'));

// Setup — выполняется один раз
export function setup() {
  const loginRes = http.post('http://api.example.com/auth/login', JSON.stringify({
    email: 'loadtest@example.com',
    password: 'password123'
  }), {
    headers: { 'Content-Type': 'application/json' }
  });

  return { token: loginRes.json('token') };
}

// Основной сценарий
export default function(data) {
  const headers = {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${data.token}`
  };

  group('Browse Products', function() {
    // Получение списка продуктов
    const productsRes = http.get('http://api.example.com/api/products', { headers });

    check(productsRes, {
      'products status 200': (r) => r.status === 200,
      'products not empty': (r) => r.json('items').length > 0,
    });

    errorRate.add(productsRes.status !== 200);

    sleep(1);

    // Детали случайного продукта
    const productId = Math.floor(Math.random() * 100) + 1;
    const productRes = http.get(`http://api.example.com/api/products/${productId}`, { headers });

    check(productRes, {
      'product status 200': (r) => r.status === 200,
    });

    sleep(0.5);
  });

  group('Create Order', function() {
    const startTime = Date.now();

    const orderRes = http.post('http://api.example.com/api/orders', JSON.stringify({
      product_id: Math.floor(Math.random() * 100) + 1,
      quantity: Math.floor(Math.random() * 5) + 1,
    }), { headers });

    const duration = Date.now() - startTime;
    orderDuration.add(duration);

    const success = check(orderRes, {
      'order created': (r) => r.status === 201,
    });

    if (success) {
      ordersCreated.add(1);
    }

    errorRate.add(!success);

    sleep(2);
  });
}

// Teardown — выполняется после всех итераций
export function teardown(data) {
  // Logout или очистка
  http.post('http://api.example.com/auth/logout', null, {
    headers: { 'Authorization': `Bearer ${data.token}` }
  });
}

// Запуск: k6 run k6_test.js
// С выводом в InfluxDB: k6 run --out influxdb=http://localhost:8086/k6 k6_test.js
```

### Сценарии тестирования

```javascript
// scenarios.js
import http from 'k6/http';
import { sleep } from 'k6';

export const options = {
  scenarios: {
    // Сценарий 1: Постоянная нагрузка
    constant_load: {
      executor: 'constant-vus',
      vus: 50,
      duration: '5m',
      exec: 'browseProducts',
    },

    // Сценарий 2: Рампа
    ramping_load: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '2m', target: 100 },
        { duration: '5m', target: 100 },
        { duration: '2m', target: 0 },
      ],
      exec: 'checkout',
    },

    // Сценарий 3: Фиксированное количество запросов
    fixed_iterations: {
      executor: 'shared-iterations',
      vus: 10,
      iterations: 1000,
      maxDuration: '10m',
      exec: 'createOrder',
    },

    // Сценарий 4: Заданный RPS
    constant_arrival_rate: {
      executor: 'constant-arrival-rate',
      rate: 100,           // 100 запросов в секунду
      timeUnit: '1s',
      duration: '5m',
      preAllocatedVUs: 50,
      maxVUs: 200,
      exec: 'apiCall',
    },

    // Сценарий 5: Spike test
    spike_test: {
      executor: 'ramping-arrival-rate',
      startRate: 10,
      timeUnit: '1s',
      stages: [
        { target: 10, duration: '1m' },
        { target: 200, duration: '10s' },  // Spike!
        { target: 10, duration: '1m' },
      ],
      preAllocatedVUs: 100,
      maxVUs: 500,
      exec: 'handleSpike',
    },
  },
};

export function browseProducts() {
  http.get('http://api.example.com/api/products');
  sleep(1);
}

export function checkout() {
  http.post('http://api.example.com/api/checkout', JSON.stringify({
    cart_id: 'test-cart'
  }));
  sleep(2);
}

export function createOrder() {
  http.post('http://api.example.com/api/orders', JSON.stringify({
    product_id: 1,
    quantity: 1
  }));
}

export function apiCall() {
  http.get('http://api.example.com/api/health');
}

export function handleSpike() {
  http.get('http://api.example.com/api/products');
}
```

## pytest-benchmark для микробенчмарков

```python
# test_performance.py
import pytest
from your_app import process_data, calculate_hash, serialize_data

class TestPerformance:
    """Микробенчмарки с pytest-benchmark"""

    def test_process_data_performance(self, benchmark):
        """Тестирование производительности обработки данных"""
        data = list(range(10000))

        # benchmark автоматически запустит функцию много раз
        result = benchmark(process_data, data)

        assert result is not None

    def test_hash_calculation(self, benchmark):
        """Тестирование производительности хеширования"""
        data = "test data" * 1000

        result = benchmark(calculate_hash, data)

        assert len(result) == 64  # SHA-256

    def test_serialization_comparison(self, benchmark):
        """Сравнение разных методов сериализации"""
        data = {"items": [{"id": i, "value": f"item_{i}"} for i in range(1000)]}

        # Группировка для сравнения
        benchmark.group = "serialization"

        result = benchmark(serialize_data, data)

        assert result is not None

    @pytest.mark.parametrize("size", [100, 1000, 10000])
    def test_scaling(self, benchmark, size):
        """Тестирование масштабирования"""
        data = list(range(size))

        benchmark.group = "scaling"
        result = benchmark(process_data, data)

        assert len(result) == size

# Фикстуры для benchmark
@pytest.fixture
def large_dataset():
    """Большой датасет для тестирования"""
    return [{"id": i, "data": f"item_{i}" * 100} for i in range(10000)]

def test_with_fixture(benchmark, large_dataset):
    """Тест с предварительно подготовленными данными"""
    # pedantic=True для более точных измерений
    result = benchmark.pedantic(
        process_data,
        args=(large_dataset,),
        iterations=10,
        rounds=100
    )

# Запуск:
# pytest test_performance.py --benchmark-only
# pytest test_performance.py --benchmark-compare  # Сравнение с предыдущими результатами
# pytest test_performance.py --benchmark-save=baseline  # Сохранение baseline
```

## Автоматизированное тестирование в CI/CD

### GitHub Actions

```yaml
# .github/workflows/performance.yml
name: Performance Tests

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * *'  # Ежедневно в 2:00

jobs:
  performance-test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install locust k6

      - name: Start application
        run: |
          uvicorn app:app --host 0.0.0.0 --port 8000 &
          sleep 10  # Ждём старта

      - name: Run Locust test
        run: |
          locust -f tests/locustfile.py \
            --headless \
            --host http://localhost:8000 \
            --users 50 \
            --spawn-rate 10 \
            --run-time 5m \
            --csv results/locust \
            --html results/locust_report.html

      - name: Run k6 test
        run: |
          k6 run tests/k6_test.js \
            --out json=results/k6_results.json \
            --summary-export=results/k6_summary.json

      - name: Check thresholds
        run: |
          python scripts/check_performance.py \
            --locust-csv results/locust_stats.csv \
            --k6-json results/k6_summary.json \
            --thresholds config/performance_thresholds.yaml

      - name: Upload results
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: performance-results
          path: results/

      - name: Comment on PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v6
        with:
          script: |
            const fs = require('fs');
            const summary = JSON.parse(fs.readFileSync('results/k6_summary.json'));

            const body = `
            ## Performance Test Results

            | Metric | Value |
            |--------|-------|
            | Total Requests | ${summary.metrics.http_reqs.values.count} |
            | Avg Response Time | ${summary.metrics.http_req_duration.values.avg.toFixed(2)}ms |
            | P95 Response Time | ${summary.metrics.http_req_duration.values['p(95)'].toFixed(2)}ms |
            | Error Rate | ${(summary.metrics.http_req_failed.values.rate * 100).toFixed(2)}% |
            `;

            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: body
            });
```

### Скрипт проверки порогов

```python
# scripts/check_performance.py
import argparse
import yaml
import json
import csv
import sys
from dataclasses import dataclass
from typing import Dict, Any

@dataclass
class PerformanceResult:
    passed: bool
    violations: list
    summary: Dict[str, Any]

def load_thresholds(path: str) -> dict:
    with open(path) as f:
        return yaml.safe_load(f)

def check_k6_results(results_path: str, thresholds: dict) -> PerformanceResult:
    """Проверка результатов k6"""
    with open(results_path) as f:
        results = json.load(f)

    violations = []
    metrics = results.get('metrics', {})

    # Проверка p95 latency
    p95 = metrics.get('http_req_duration', {}).get('values', {}).get('p(95)', 0)
    if p95 > thresholds['latency']['p95_ms']:
        violations.append(f"P95 latency {p95:.2f}ms exceeds threshold {thresholds['latency']['p95_ms']}ms")

    # Проверка error rate
    error_rate = metrics.get('http_req_failed', {}).get('values', {}).get('rate', 0)
    if error_rate > thresholds['error_rate']['max_percent'] / 100:
        violations.append(f"Error rate {error_rate*100:.2f}% exceeds threshold {thresholds['error_rate']['max_percent']}%")

    # Проверка throughput
    rps = metrics.get('http_reqs', {}).get('values', {}).get('rate', 0)
    if rps < thresholds['throughput']['min_rps']:
        violations.append(f"Throughput {rps:.2f} RPS below threshold {thresholds['throughput']['min_rps']} RPS")

    return PerformanceResult(
        passed=len(violations) == 0,
        violations=violations,
        summary={
            'p95_latency_ms': p95,
            'error_rate_percent': error_rate * 100,
            'throughput_rps': rps
        }
    )

def check_locust_results(csv_path: str, thresholds: dict) -> PerformanceResult:
    """Проверка результатов Locust"""
    violations = []
    summary = {}

    with open(csv_path) as f:
        reader = csv.DictReader(f)
        for row in reader:
            if row['Name'] == 'Aggregated':
                p95 = float(row['95%'])
                error_rate = float(row['Failure Count']) / float(row['Request Count']) * 100
                rps = float(row['Requests/s'])

                summary = {
                    'p95_latency_ms': p95,
                    'error_rate_percent': error_rate,
                    'throughput_rps': rps
                }

                if p95 > thresholds['latency']['p95_ms']:
                    violations.append(f"P95 latency {p95:.2f}ms exceeds threshold")
                if error_rate > thresholds['error_rate']['max_percent']:
                    violations.append(f"Error rate {error_rate:.2f}% exceeds threshold")
                if rps < thresholds['throughput']['min_rps']:
                    violations.append(f"Throughput {rps:.2f} RPS below threshold")

    return PerformanceResult(
        passed=len(violations) == 0,
        violations=violations,
        summary=summary
    )

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument('--locust-csv')
    parser.add_argument('--k6-json')
    parser.add_argument('--thresholds', required=True)
    args = parser.parse_args()

    thresholds = load_thresholds(args.thresholds)
    all_passed = True

    if args.k6_json:
        result = check_k6_results(args.k6_json, thresholds)
        print(f"\nk6 Results:")
        print(f"  Summary: {result.summary}")
        if not result.passed:
            print(f"  Violations:")
            for v in result.violations:
                print(f"    - {v}")
            all_passed = False

    if args.locust_csv:
        result = check_locust_results(args.locust_csv, thresholds)
        print(f"\nLocust Results:")
        print(f"  Summary: {result.summary}")
        if not result.passed:
            print(f"  Violations:")
            for v in result.violations:
                print(f"    - {v}")
            all_passed = False

    if not all_passed:
        print("\n❌ Performance tests FAILED")
        sys.exit(1)

    print("\n✅ Performance tests PASSED")
    sys.exit(0)

if __name__ == '__main__':
    main()
```

### Конфигурация порогов

```yaml
# config/performance_thresholds.yaml
latency:
  p50_ms: 100
  p95_ms: 500
  p99_ms: 1000
  max_ms: 5000

error_rate:
  max_percent: 1.0

throughput:
  min_rps: 100

# Пороги по эндпоинтам
endpoints:
  /api/products:
    p95_ms: 200
    max_error_rate: 0.1

  /api/orders:
    p95_ms: 1000
    max_error_rate: 0.5

  /api/search:
    p95_ms: 500
    max_error_rate: 0.1
```

## Анализ результатов

### Генерация отчётов

```python
# report_generator.py
import json
from datetime import datetime
from jinja2 import Template
import matplotlib.pyplot as plt
import pandas as pd
from dataclasses import dataclass
from typing import List, Dict

@dataclass
class PerformanceReport:
    timestamp: datetime
    duration_seconds: int
    total_requests: int
    avg_response_time_ms: float
    p50_ms: float
    p95_ms: float
    p99_ms: float
    error_rate: float
    throughput_rps: float
    violations: List[str]

class ReportGenerator:
    """Генератор отчётов о производительности"""

    def __init__(self, results_dir: str):
        self.results_dir = results_dir

    def generate_html_report(self, data: PerformanceReport) -> str:
        """Генерация HTML отчёта"""
        template = Template("""
        <!DOCTYPE html>
        <html>
        <head>
            <title>Performance Test Report</title>
            <style>
                body { font-family: Arial, sans-serif; margin: 40px; }
                .metric { display: inline-block; margin: 10px; padding: 20px; background: #f5f5f5; }
                .pass { color: green; }
                .fail { color: red; }
                table { border-collapse: collapse; width: 100%; }
                th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
            </style>
        </head>
        <body>
            <h1>Performance Test Report</h1>
            <p>Generated: {{ timestamp }}</p>

            <h2>Summary</h2>
            <div class="metrics">
                <div class="metric">
                    <strong>Total Requests</strong><br>
                    {{ total_requests }}
                </div>
                <div class="metric">
                    <strong>Throughput</strong><br>
                    {{ "%.2f"|format(throughput_rps) }} req/s
                </div>
                <div class="metric">
                    <strong>Error Rate</strong><br>
                    <span class="{{ 'pass' if error_rate < 1 else 'fail' }}">
                        {{ "%.2f"|format(error_rate) }}%
                    </span>
                </div>
            </div>

            <h2>Latency</h2>
            <table>
                <tr><th>Metric</th><th>Value (ms)</th></tr>
                <tr><td>Average</td><td>{{ "%.2f"|format(avg_response_time) }}</td></tr>
                <tr><td>P50</td><td>{{ "%.2f"|format(p50) }}</td></tr>
                <tr><td>P95</td><td>{{ "%.2f"|format(p95) }}</td></tr>
                <tr><td>P99</td><td>{{ "%.2f"|format(p99) }}</td></tr>
            </table>

            {% if violations %}
            <h2 class="fail">Violations</h2>
            <ul>
                {% for v in violations %}
                <li>{{ v }}</li>
                {% endfor %}
            </ul>
            {% endif %}
        </body>
        </html>
        """)

        return template.render(
            timestamp=data.timestamp.isoformat(),
            total_requests=data.total_requests,
            throughput_rps=data.throughput_rps,
            error_rate=data.error_rate,
            avg_response_time=data.avg_response_time_ms,
            p50=data.p50_ms,
            p95=data.p95_ms,
            p99=data.p99_ms,
            violations=data.violations
        )

    def plot_latency_distribution(self, latencies: List[float], output_path: str):
        """График распределения latency"""
        plt.figure(figsize=(12, 6))

        # Гистограмма
        plt.subplot(1, 2, 1)
        plt.hist(latencies, bins=50, edgecolor='black')
        plt.xlabel('Response Time (ms)')
        plt.ylabel('Frequency')
        plt.title('Response Time Distribution')

        # Box plot
        plt.subplot(1, 2, 2)
        plt.boxplot(latencies)
        plt.ylabel('Response Time (ms)')
        plt.title('Response Time Box Plot')

        plt.tight_layout()
        plt.savefig(output_path)
        plt.close()

    def plot_throughput_over_time(self, timestamps: List[float], rps: List[float], output_path: str):
        """График throughput во времени"""
        plt.figure(figsize=(12, 4))
        plt.plot(timestamps, rps)
        plt.xlabel('Time (seconds)')
        plt.ylabel('Requests per Second')
        plt.title('Throughput Over Time')
        plt.grid(True)
        plt.savefig(output_path)
        plt.close()
```

## Best Practices

```python
"""
Best Practices для Performance Testing:

1. Подготовка:
   - Используйте данные, близкие к production
   - Тестируйте в изолированной среде
   - Прогревайте систему перед тестами
   - Документируйте конфигурацию тестов

2. Выполнение:
   - Начинайте с baseline тестов
   - Увеличивайте нагрузку постепенно
   - Мониторьте ресурсы во время тестов
   - Сохраняйте все результаты

3. Анализ:
   - Смотрите на percentiles, не средние
   - Анализируйте тренды, не только абсолютные значения
   - Корреляция с метриками инфраструктуры
   - Ищите точки насыщения

4. CI/CD интеграция:
   - Автоматизируйте тесты в pipeline
   - Устанавливайте реалистичные пороги
   - Сравнивайте с baseline
   - Блокируйте деплой при деградации

5. Типичные ошибки:
   - Тестирование только happy path
   - Игнорирование cold start
   - Недостаточная длительность тестов
   - Отсутствие корреляции с логами
"""

class PerformanceTestGuidelines:
    """Рекомендации по нагрузочному тестированию"""

    # Минимальная длительность тестов по типам
    MIN_DURATION = {
        "smoke": 60,           # 1 минута
        "load": 300,           # 5 минут
        "stress": 600,         # 10 минут
        "endurance": 3600,     # 1 час
    }

    # Рекомендуемые пороги
    RECOMMENDED_THRESHOLDS = {
        "p95_latency_ms": 500,
        "p99_latency_ms": 1000,
        "error_rate_percent": 1.0,
        "min_throughput_percent": 80,  # % от baseline
    }

    @staticmethod
    def calculate_virtual_users(
        target_rps: float,
        avg_response_time_ms: float,
        think_time_ms: float = 1000
    ) -> int:
        """
        Расчёт количества виртуальных пользователей

        Формула: VU = RPS * (response_time + think_time) / 1000
        """
        cycle_time_s = (avg_response_time_ms + think_time_ms) / 1000
        return int(target_rps * cycle_time_s)
```

## Заключение

Эффективное тестирование производительности требует:

1. **Правильного выбора инструментов** — Locust/k6 для HTTP, pytest-benchmark для микробенчмарков
2. **Реалистичных сценариев** — симуляция реального поведения пользователей
3. **Автоматизации** — интеграция в CI/CD pipeline
4. **Анализа трендов** — сравнение с baseline и историческими данными
5. **Мониторинга ресурсов** — корреляция с метриками инфраструктуры
6. **Документации** — сохранение результатов и конфигураций

Performance testing — это не разовое мероприятие, а непрерывный процесс обеспечения качества.
