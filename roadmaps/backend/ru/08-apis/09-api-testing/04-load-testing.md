# Load Testing (Нагрузочное тестирование)

[prev: 03-functional-testing](./03-functional-testing.md) | [next: 05-mocking-apis](./05-mocking-apis.md)

---

## Введение

**Нагрузочное тестирование (Load Testing)** - это тип тестирования производительности, при котором проверяется поведение системы под ожидаемой и повышенной нагрузкой. Цель - определить, как API работает при различном количестве одновременных пользователей и запросов.

## Виды тестирования производительности

### 1. Load Testing (Нагрузочное)
Тестирование при ожидаемой нагрузке. Проверяет, справляется ли система с обычным количеством пользователей.

### 2. Stress Testing (Стресс-тестирование)
Тестирование за пределами нормальной нагрузки. Определяет точку отказа системы.

### 3. Spike Testing (Пиковое)
Тестирование резких скачков нагрузки. Проверяет, как система справляется с внезапным увеличением трафика.

### 4. Soak Testing (Длительное)
Тестирование под нагрузкой в течение длительного времени. Выявляет утечки памяти и деградацию производительности.

### 5. Scalability Testing (Масштабируемость)
Тестирование способности системы масштабироваться при увеличении нагрузки.

## Ключевые метрики

| Метрика | Описание | Типичные значения |
|---------|----------|-------------------|
| **Response Time** | Время ответа | < 200ms (хорошо), < 1s (приемлемо) |
| **Throughput** | Запросов в секунду (RPS) | Зависит от системы |
| **Error Rate** | Процент ошибок | < 1% (хорошо), < 5% (приемлемо) |
| **Concurrent Users** | Одновременные пользователи | Зависит от требований |
| **Latency Percentiles** | P50, P95, P99 | P99 < 1s |

## Инструменты для нагрузочного тестирования

### Locust (Python)

Мощный и гибкий инструмент для нагрузочного тестирования на Python.

```bash
pip install locust
```

### Базовый тест

```python
# locustfile.py
from locust import HttpUser, task, between

class APIUser(HttpUser):
    """Симуляция пользователя API."""

    # Время ожидания между запросами (1-5 секунд)
    wait_time = between(1, 5)

    def on_start(self):
        """Выполняется при старте каждого пользователя."""
        # Аутентификация
        response = self.client.post("/auth/login", json={
            "email": "test@example.com",
            "password": "password123"
        })
        self.token = response.json()["access_token"]
        self.client.headers["Authorization"] = f"Bearer {self.token}"

    @task(3)
    def get_users(self):
        """Получение списка пользователей (вес 3)."""
        self.client.get("/api/users")

    @task(2)
    def get_user_profile(self):
        """Получение профиля (вес 2)."""
        self.client.get("/api/users/me")

    @task(1)
    def create_post(self):
        """Создание поста (вес 1)."""
        self.client.post("/api/posts", json={
            "title": "Load Test Post",
            "content": "This is a test post"
        })
```

```bash
# Запуск Locust
locust -f locustfile.py --host=http://localhost:8000

# Запуск без UI (headless)
locust -f locustfile.py --host=http://localhost:8000 \
    --headless -u 100 -r 10 -t 5m

# -u: количество пользователей
# -r: скорость создания пользователей в секунду
# -t: длительность теста
```

### Продвинутый сценарий

```python
# locustfile_advanced.py
from locust import HttpUser, task, between, events, TaskSet
import random
import logging

class UserBehavior(TaskSet):
    """Поведение пользователя в системе."""

    def on_start(self):
        """Инициализация пользователя."""
        self.user_id = None
        self.post_ids = []

    @task(10)
    def browse_posts(self):
        """Просмотр постов - частая операция."""
        with self.client.get(
            "/api/posts",
            params={"page": random.randint(1, 10), "size": 20},
            catch_response=True
        ) as response:
            if response.status_code == 200:
                data = response.json()
                if len(data.get("items", [])) > 0:
                    response.success()
                else:
                    response.failure("Empty response")
            else:
                response.failure(f"Status: {response.status_code}")

    @task(5)
    def view_post_details(self):
        """Просмотр деталей поста."""
        post_id = random.randint(1, 1000)
        with self.client.get(
            f"/api/posts/{post_id}",
            name="/api/posts/[id]",  # Группировка в статистике
            catch_response=True
        ) as response:
            if response.status_code == 200:
                response.success()
            elif response.status_code == 404:
                response.success()  # 404 допустим
            else:
                response.failure(f"Unexpected: {response.status_code}")

    @task(2)
    def search_posts(self):
        """Поиск постов."""
        queries = ["python", "api", "testing", "backend"]
        self.client.get(
            "/api/posts/search",
            params={"q": random.choice(queries)},
            name="/api/posts/search"
        )

    @task(1)
    def create_and_delete_post(self):
        """Создание и удаление поста."""
        # Создаем пост
        response = self.client.post("/api/posts", json={
            "title": f"Load Test {random.randint(1, 10000)}",
            "content": "Test content for load testing"
        })

        if response.status_code == 201:
            post_id = response.json()["id"]
            # Удаляем пост
            self.client.delete(
                f"/api/posts/{post_id}",
                name="/api/posts/[id]"
            )


class WebsiteUser(HttpUser):
    """Пользователь веб-сайта."""

    tasks = [UserBehavior]
    wait_time = between(1, 3)

    def on_start(self):
        """Аутентификация при старте."""
        response = self.client.post("/auth/login", json={
            "email": "loadtest@example.com",
            "password": "testpass123"
        })
        if response.status_code == 200:
            token = response.json()["access_token"]
            self.client.headers["Authorization"] = f"Bearer {token}"


# Обработчики событий
@events.request.add_listener
def on_request(request_type, name, response_time, response_length, **kwargs):
    """Логирование медленных запросов."""
    if response_time > 1000:  # > 1 секунды
        logging.warning(f"Slow request: {name} - {response_time}ms")

@events.test_stop.add_listener
def on_test_stop(environment, **kwargs):
    """Действия после завершения теста."""
    stats = environment.stats
    print(f"\nTotal requests: {stats.total.num_requests}")
    print(f"Failures: {stats.total.num_failures}")
    print(f"Average response time: {stats.total.avg_response_time}ms")
```

### k6 (JavaScript)

Современный инструмент для нагрузочного тестирования от Grafana Labs.

```bash
# Установка
brew install k6  # macOS
# или
docker pull grafana/k6
```

```javascript
// load_test.js
import http from 'k6/http';
import { check, sleep, group } from 'k6';
import { Rate, Trend } from 'k6/metrics';

// Кастомные метрики
const errorRate = new Rate('errors');
const loginDuration = new Trend('login_duration');

// Конфигурация теста
export const options = {
    // Стадии нагрузки
    stages: [
        { duration: '1m', target: 50 },   // Разогрев до 50 пользователей
        { duration: '3m', target: 50 },   // Держим 50 пользователей
        { duration: '1m', target: 100 },  // Увеличиваем до 100
        { duration: '3m', target: 100 },  // Держим 100 пользователей
        { duration: '1m', target: 0 },    // Плавное снижение
    ],
    // Пороговые значения
    thresholds: {
        http_req_duration: ['p(95)<500', 'p(99)<1000'],
        errors: ['rate<0.01'],  // < 1% ошибок
        login_duration: ['p(95)<300'],
    },
};

const BASE_URL = 'http://localhost:8000/api';
let authToken = '';

export function setup() {
    // Подготовка: получение токена
    const loginRes = http.post(`${BASE_URL}/auth/login`, JSON.stringify({
        email: 'loadtest@example.com',
        password: 'testpass123'
    }), {
        headers: { 'Content-Type': 'application/json' }
    });

    return { token: loginRes.json('access_token') };
}

export default function(data) {
    const headers = {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${data.token}`
    };

    group('Browse API', () => {
        // Получение списка пользователей
        const usersRes = http.get(`${BASE_URL}/users`, { headers });
        check(usersRes, {
            'users status is 200': (r) => r.status === 200,
            'users response has items': (r) => r.json('items') !== undefined,
        });
        errorRate.add(usersRes.status !== 200);

        sleep(1);

        // Получение списка постов
        const postsRes = http.get(`${BASE_URL}/posts?page=1&size=10`, { headers });
        check(postsRes, {
            'posts status is 200': (r) => r.status === 200,
        });
        errorRate.add(postsRes.status !== 200);
    });

    group('User Actions', () => {
        // Создание поста
        const createRes = http.post(`${BASE_URL}/posts`, JSON.stringify({
            title: `Load Test Post ${Date.now()}`,
            content: 'Test content'
        }), { headers });

        check(createRes, {
            'create post status is 201': (r) => r.status === 201,
        });

        if (createRes.status === 201) {
            const postId = createRes.json('id');

            // Чтение поста
            http.get(`${BASE_URL}/posts/${postId}`, { headers });

            // Удаление поста
            http.del(`${BASE_URL}/posts/${postId}`, null, { headers });
        }
    });

    sleep(Math.random() * 2 + 1);  // 1-3 секунды
}

export function teardown(data) {
    // Очистка после теста
    console.log('Test completed');
}
```

```bash
# Запуск теста
k6 run load_test.js

# С выводом в InfluxDB
k6 run --out influxdb=http://localhost:8086/k6 load_test.js

# С HTML-отчетом
k6 run --out json=results.json load_test.js
```

### Сценарии нагрузки k6

```javascript
// scenarios.js
export const options = {
    scenarios: {
        // Постоянная нагрузка
        constant_load: {
            executor: 'constant-vus',
            vus: 50,
            duration: '5m',
        },

        // Нарастающая нагрузка
        ramping_load: {
            executor: 'ramping-vus',
            startVUs: 0,
            stages: [
                { duration: '2m', target: 100 },
                { duration: '5m', target: 100 },
                { duration: '2m', target: 0 },
            ],
        },

        // Заданное количество запросов
        constant_arrival: {
            executor: 'constant-arrival-rate',
            rate: 100,  // 100 RPS
            timeUnit: '1s',
            duration: '5m',
            preAllocatedVUs: 50,
        },

        // Пиковая нагрузка
        spike: {
            executor: 'ramping-arrival-rate',
            startRate: 10,
            timeUnit: '1s',
            stages: [
                { duration: '1m', target: 10 },
                { duration: '10s', target: 500 },  // Резкий скачок
                { duration: '1m', target: 500 },
                { duration: '10s', target: 10 },   // Резкое падение
            ],
            preAllocatedVUs: 200,
        },
    },
};
```

## Apache JMeter

Классический инструмент для нагрузочного тестирования.

### Конфигурация через CLI

```bash
# Запуск теста
jmeter -n -t test_plan.jmx -l results.jtl -e -o report/

# -n: non-GUI mode
# -t: test plan file
# -l: results file
# -e: generate HTML report
# -o: report output folder
```

### JMeter Test Plan (XML)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jmeterTestPlan version="1.2">
  <hashTree>
    <TestPlan guiclass="TestPlanGui" testclass="TestPlan" testname="API Load Test">
      <elementProp name="TestPlan.user_defined_variables" elementType="Arguments"/>
      <boolProp name="TestPlan.functional_mode">false</boolProp>
    </TestPlan>
    <hashTree>
      <ThreadGroup guiclass="ThreadGroupGui" testclass="ThreadGroup" testname="Users">
        <intProp name="ThreadGroup.num_threads">100</intProp>
        <intProp name="ThreadGroup.ramp_time">60</intProp>
        <longProp name="ThreadGroup.duration">300</longProp>
        <boolProp name="ThreadGroup.scheduler">true</boolProp>
      </ThreadGroup>
      <hashTree>
        <HTTPSamplerProxy guiclass="HttpTestSampleGui" testclass="HTTPSamplerProxy" testname="Get Users">
          <stringProp name="HTTPSampler.domain">localhost</stringProp>
          <stringProp name="HTTPSampler.port">8000</stringProp>
          <stringProp name="HTTPSampler.path">/api/users</stringProp>
          <stringProp name="HTTPSampler.method">GET</stringProp>
        </HTTPSamplerProxy>
      </hashTree>
    </hashTree>
  </hashTree>
</jmeterTestPlan>
```

## Wrk - простой HTTP бенчмарк

```bash
# Установка
brew install wrk  # macOS

# Базовый тест
wrk -t12 -c400 -d30s http://localhost:8000/api/users

# -t: количество потоков
# -c: количество соединений
# -d: длительность

# С Lua скриптом
wrk -t4 -c100 -d30s -s script.lua http://localhost:8000/api/users
```

```lua
-- script.lua
wrk.method = "POST"
wrk.body   = '{"email": "test@example.com", "name": "Test"}'
wrk.headers["Content-Type"] = "application/json"
wrk.headers["Authorization"] = "Bearer token123"

response = function(status, headers, body)
    if status ~= 201 then
        print("Error: " .. status)
    end
end
```

## Python: собственные скрипты

```python
# benchmark.py
import asyncio
import aiohttp
import time
from dataclasses import dataclass
from typing import List
import statistics

@dataclass
class BenchmarkResult:
    total_requests: int
    successful_requests: int
    failed_requests: int
    total_time: float
    avg_response_time: float
    min_response_time: float
    max_response_time: float
    p50: float
    p95: float
    p99: float
    requests_per_second: float

async def make_request(
    session: aiohttp.ClientSession,
    url: str,
    method: str = "GET",
    **kwargs
) -> tuple[bool, float]:
    """Выполнение одного запроса."""
    start = time.perf_counter()
    try:
        async with session.request(method, url, **kwargs) as response:
            await response.read()
            elapsed = time.perf_counter() - start
            return response.status < 400, elapsed
    except Exception:
        elapsed = time.perf_counter() - start
        return False, elapsed

async def run_benchmark(
    url: str,
    num_requests: int,
    concurrency: int,
    method: str = "GET",
    headers: dict = None,
    json_data: dict = None
) -> BenchmarkResult:
    """Запуск бенчмарка."""
    response_times: List[float] = []
    successful = 0
    failed = 0

    semaphore = asyncio.Semaphore(concurrency)

    async def limited_request():
        nonlocal successful, failed
        async with semaphore:
            success, elapsed = await make_request(
                session, url, method,
                headers=headers, json=json_data
            )
            response_times.append(elapsed * 1000)  # В миллисекундах
            if success:
                successful += 1
            else:
                failed += 1

    connector = aiohttp.TCPConnector(limit=concurrency)
    async with aiohttp.ClientSession(connector=connector) as session:
        start_time = time.perf_counter()

        tasks = [limited_request() for _ in range(num_requests)]
        await asyncio.gather(*tasks)

        total_time = time.perf_counter() - start_time

    sorted_times = sorted(response_times)
    return BenchmarkResult(
        total_requests=num_requests,
        successful_requests=successful,
        failed_requests=failed,
        total_time=total_time,
        avg_response_time=statistics.mean(response_times),
        min_response_time=min(response_times),
        max_response_time=max(response_times),
        p50=sorted_times[int(len(sorted_times) * 0.50)],
        p95=sorted_times[int(len(sorted_times) * 0.95)],
        p99=sorted_times[int(len(sorted_times) * 0.99)],
        requests_per_second=num_requests / total_time
    )

async def main():
    result = await run_benchmark(
        url="http://localhost:8000/api/users",
        num_requests=1000,
        concurrency=50,
        headers={"Authorization": "Bearer token123"}
    )

    print(f"""
Benchmark Results:
==================
Total Requests:     {result.total_requests}
Successful:         {result.successful_requests}
Failed:             {result.failed_requests}
Total Time:         {result.total_time:.2f}s
RPS:                {result.requests_per_second:.2f}

Response Times (ms):
  Average:          {result.avg_response_time:.2f}
  Min:              {result.min_response_time:.2f}
  Max:              {result.max_response_time:.2f}
  P50:              {result.p50:.2f}
  P95:              {result.p95:.2f}
  P99:              {result.p99:.2f}
    """)

if __name__ == "__main__":
    asyncio.run(main())
```

## Best Practices

### 1. Реалистичные сценарии

```python
# Используйте реальное распределение нагрузки
class RealisticUser(HttpUser):
    wait_time = between(1, 5)

    @task(70)  # 70% запросов
    def read_operations(self):
        self.client.get("/api/posts")

    @task(20)  # 20% запросов
    def search(self):
        self.client.get("/api/search?q=test")

    @task(10)  # 10% запросов
    def write_operations(self):
        self.client.post("/api/posts", json={...})
```

### 2. Прогрев системы

```javascript
// k6: прогрев перед основным тестом
export const options = {
    stages: [
        { duration: '2m', target: 10 },  // Прогрев
        { duration: '5m', target: 100 }, // Основной тест
        { duration: '2m', target: 0 },   // Охлаждение
    ],
};
```

### 3. Мониторинг ресурсов

```bash
# Мониторинг во время теста
htop                          # CPU и память
docker stats                  # Docker контейнеры
iostat -x 1                   # Disk I/O
netstat -s                    # Сетевая статистика
```

### 4. Изоляция тестового окружения

```yaml
# docker-compose.test.yml
version: '3.8'
services:
  api:
    build: .
    environment:
      - DATABASE_URL=postgresql://test:test@db:5432/loadtest
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 2G

  db:
    image: postgres:15
    environment:
      - POSTGRES_DB=loadtest
```

### 5. Анализ результатов

```python
# Сравнение результатов тестов
def compare_results(baseline: BenchmarkResult, current: BenchmarkResult):
    """Сравнение результатов с базовым уровнем."""
    rps_change = (current.requests_per_second - baseline.requests_per_second) / baseline.requests_per_second * 100
    p95_change = (current.p95 - baseline.p95) / baseline.p95 * 100

    print(f"RPS change: {rps_change:+.2f}%")
    print(f"P95 change: {p95_change:+.2f}%")

    if rps_change < -10:
        print("WARNING: Significant performance degradation!")
    if p95_change > 20:
        print("WARNING: Response times increased significantly!")
```

## Интеграция с CI/CD

```yaml
# .github/workflows/load-test.yml
name: Load Test

on:
  push:
    branches: [main]
  schedule:
    - cron: '0 2 * * *'  # Ежедневно в 2:00

jobs:
  load-test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: test

    steps:
      - uses: actions/checkout@v3

      - name: Start API
        run: |
          docker-compose up -d
          sleep 10

      - name: Install k6
        run: |
          sudo apt-get update
          sudo apt-get install -y k6

      - name: Run load test
        run: |
          k6 run --out json=results.json load_test.js

      - name: Check thresholds
        run: |
          python scripts/check_thresholds.py results.json

      - name: Upload results
        uses: actions/upload-artifact@v3
        with:
          name: load-test-results
          path: results.json
```

## Инструменты мониторинга

| Инструмент | Описание |
|------------|----------|
| **Grafana** | Визуализация метрик |
| **Prometheus** | Сбор метрик |
| **InfluxDB** | Time-series база данных |
| **Datadog** | APM и мониторинг |
| **New Relic** | Мониторинг производительности |

## Заключение

Нагрузочное тестирование - критически важная часть обеспечения качества API:

- **Выявляет узкие места** до продакшена
- **Определяет пределы** системы
- **Помогает планировать** масштабирование
- **Предотвращает** проблемы под нагрузкой

Ключевые рекомендации:
- Тестируйте реалистичные сценарии
- Устанавливайте четкие пороговые значения
- Автоматизируйте тесты в CI/CD
- Мониторьте ресурсы во время тестов
- Сравнивайте результаты с базовыми показателями

---

[prev: 03-functional-testing](./03-functional-testing.md) | [next: 05-mocking-apis](./05-mocking-apis.md)
