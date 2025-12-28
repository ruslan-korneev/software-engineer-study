# Load Balancers (Балансировщики нагрузки)

[prev: 09-cdn](./09-cdn.md) | [next: 11-horizontal-vs-vertical-scaling](./11-horizontal-vs-vertical-scaling.md)

---

## Что такое Load Balancer и зачем он нужен

**Load Balancer (балансировщик нагрузки)** — это компонент инфраструктуры, который распределяет входящий сетевой трафик между несколькими серверами (backend servers), обеспечивая:

1. **Высокую доступность** — если один сервер падает, трафик перенаправляется на другие
2. **Масштабируемость** — можно добавлять серверы для обработки растущей нагрузки
3. **Производительность** — равномерное распределение нагрузки предотвращает перегрузку отдельных серверов
4. **Гибкость** — можно обновлять серверы по очереди без downtime (rolling updates)

```
                    ┌─────────────────┐
                    │    Клиенты      │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │  Load Balancer  │
                    └────────┬────────┘
                             │
           ┌─────────────────┼─────────────────┐
           │                 │                 │
           ▼                 ▼                 ▼
    ┌──────────┐      ┌──────────┐      ┌──────────┐
    │ Server 1 │      │ Server 2 │      │ Server 3 │
    └──────────┘      └──────────┘      └──────────┘
```

### Основные задачи Load Balancer

| Задача | Описание |
|--------|----------|
| Распределение нагрузки | Равномерное распределение запросов между серверами |
| Health checking | Проверка работоспособности серверов |
| Session persistence | Сохранение сессии пользователя на одном сервере |
| SSL termination | Расшифровка HTTPS на балансировщике |
| Защита от атак | Базовая защита от DDoS |
| Сжатие и кэширование | Оптимизация трафика (для L7) |

---

## Уровни балансировки: L4 vs L7

### L4 — Transport Layer (TCP/UDP)

L4 балансировщик работает на транспортном уровне модели OSI и оперирует:
- IP-адресами источника и назначения
- TCP/UDP портами
- Не анализирует содержимое пакетов (payload)

```
┌────────────────────────────────────────────────┐
│                   L4 Load Balancer             │
├────────────────────────────────────────────────┤
│  Видит:                                        │
│  • Source IP: 192.168.1.100                    │
│  • Dest IP: 10.0.0.1                           │
│  • Source Port: 54321                          │
│  • Dest Port: 443                              │
│  • Protocol: TCP                               │
│                                                │
│  НЕ видит:                                     │
│  • HTTP headers                                │
│  • URL path                                    │
│  • Cookies                                     │
│  • Request body                                │
└────────────────────────────────────────────────┘
```

**Преимущества L4:**
- Очень высокая производительность (миллионы соединений в секунду)
- Низкая латентность (не нужно разбирать пакеты)
- Подходит для любых TCP/UDP протоколов (не только HTTP)
- Простота настройки

**Недостатки L4:**
- Нет понимания содержимого запросов
- Нельзя маршрутизировать по URL, headers, cookies
- Ограниченные возможности для sticky sessions

### L7 — Application Layer (HTTP/HTTPS)

L7 балансировщик работает на прикладном уровне и понимает HTTP/HTTPS протокол:

```
┌────────────────────────────────────────────────┐
│                   L7 Load Balancer             │
├────────────────────────────────────────────────┤
│  Видит всё, что видит L4, плюс:               │
│  • HTTP Method: GET, POST, PUT...              │
│  • URL: /api/users/123                         │
│  • Headers: Host, Cookie, User-Agent...        │
│  • Request Body (JSON, form data...)           │
│  • SSL/TLS certificate info                    │
│                                                │
│  Может делать:                                 │
│  • Content-based routing                       │
│  • URL rewriting                               │
│  • Header modification                         │
│  • Caching                                     │
│  • Compression                                 │
└────────────────────────────────────────────────┘
```

**Преимущества L7:**
- Умная маршрутизация по содержимому запроса
- SSL termination
- Кэширование статического контента
- Модификация headers
- Детальные health checks (HTTP status codes)
- A/B тестирование, canary deployments

**Недостатки L7:**
- Выше latency (нужно разобрать HTTP)
- Меньшая производительность по сравнению с L4
- Сложнее настройка

### Когда использовать какой

| Сценарий | Рекомендация |
|----------|--------------|
| Простое распределение нагрузки TCP | L4 |
| Высокопроизводительные non-HTTP сервисы | L4 |
| gRPC, WebSocket без инспекции | L4 |
| Микросервисы с маршрутизацией по URL | L7 |
| SSL termination | L7 |
| Sticky sessions по cookies | L7 |
| A/B тестирование | L7 |
| API Gateway функции | L7 |
| Смешанная нагрузка (API + static) | L7 |

### Гибридный подход

На практике часто используют оба уровня:

```
                    ┌─────────────────┐
                    │    Интернет     │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │   L4 (NLB)      │  ← Высокая производительность,
                    │   TCP/UDP       │    SSL passthrough
                    └────────┬────────┘
                             │
           ┌─────────────────┴─────────────────┐
           ▼                                   ▼
    ┌──────────────┐                    ┌──────────────┐
    │   L7 (ALB)   │                    │   L7 (ALB)   │
    │   Cluster 1  │                    │   Cluster 2  │
    └──────┬───────┘                    └──────┬───────┘
           │                                   │
     ┌─────┴─────┐                       ┌─────┴─────┐
     ▼           ▼                       ▼           ▼
  ┌─────┐     ┌─────┐                 ┌─────┐     ┌─────┐
  │ App │     │ App │                 │ App │     │ App │
  └─────┘     └─────┘                 └─────┘     └─────┘
```

---

## Алгоритмы балансировки

### 1. Round Robin (циклический)

Самый простой алгоритм — запросы распределяются по очереди:

```
Запрос 1 → Server A
Запрос 2 → Server B
Запрос 3 → Server C
Запрос 4 → Server A  (цикл начинается заново)
Запрос 5 → Server B
...
```

**Преимущества:**
- Простота реализации
- Равномерное распределение при одинаковых серверах
- Предсказуемость

**Недостатки:**
- Не учитывает текущую нагрузку серверов
- Не учитывает разную мощность серверов
- Один "тяжёлый" запрос может перегрузить сервер

```nginx
# Nginx — Round Robin по умолчанию
upstream backend {
    server backend1.example.com;
    server backend2.example.com;
    server backend3.example.com;
}
```

### 2. Weighted Round Robin (взвешенный циклический)

Модификация Round Robin с учётом веса (мощности) серверов:

```
Веса: Server A = 5, Server B = 3, Server C = 2

Распределение на 10 запросов:
Server A: 5 запросов (50%)
Server B: 3 запроса (30%)
Server C: 2 запроса (20%)
```

**Когда использовать:**
- Серверы имеют разную производительность
- Постепенный вывод сервера (снижая вес до 0)
- Canary deployments (новая версия с малым весом)

```nginx
# Nginx — Weighted Round Robin
upstream backend {
    server backend1.example.com weight=5;
    server backend2.example.com weight=3;
    server backend3.example.com weight=2;
}
```

```haproxy
# HAProxy
backend servers
    balance roundrobin
    server srv1 10.0.0.1:80 weight 5
    server srv2 10.0.0.2:80 weight 3
    server srv3 10.0.0.3:80 weight 2
```

### 3. Least Connections (наименьшее число соединений)

Запрос отправляется на сервер с минимальным количеством активных соединений:

```
Текущее состояние:
Server A: 10 активных соединений
Server B: 3 активных соединения  ← сюда пойдёт новый запрос
Server C: 7 активных соединений

Новый запрос → Server B
```

**Преимущества:**
- Учитывает реальную нагрузку
- Хорошо работает при разных временах обработки запросов
- Адаптируется к неравномерной нагрузке

**Недостатки:**
- Дополнительные накладные расходы на отслеживание соединений
- Может не учитывать "тяжесть" запросов

```nginx
# Nginx
upstream backend {
    least_conn;
    server backend1.example.com;
    server backend2.example.com;
    server backend3.example.com;
}
```

```haproxy
# HAProxy
backend servers
    balance leastconn
    server srv1 10.0.0.1:80
    server srv2 10.0.0.2:80
    server srv3 10.0.0.3:80
```

### 4. Weighted Least Connections

Комбинация Least Connections и весов:

```
Формула: score = active_connections / weight

Server A: 10 соединений, weight=5 → score = 2.0
Server B: 3 соединения, weight=3  → score = 1.0  ← выбираем
Server C: 4 соединения, weight=2  → score = 2.0
```

```nginx
# Nginx
upstream backend {
    least_conn;
    server backend1.example.com weight=5;
    server backend2.example.com weight=3;
    server backend3.example.com weight=2;
}
```

### 5. IP Hash (хеширование по IP)

Запросы от одного клиента всегда идут на один сервер:

```
hash(client_ip) % number_of_servers = server_index

Client 192.168.1.1 → hash → Server A
Client 192.168.1.2 → hash → Server C
Client 192.168.1.3 → hash → Server A
Client 192.168.1.1 → hash → Server A (тот же клиент — тот же сервер)
```

**Преимущества:**
- Простая реализация sticky sessions
- Не требует хранения состояния
- Кэширование на сервере эффективнее

**Недостатки:**
- При добавлении/удалении серверов перераспределяются многие клиенты
- Клиенты за NAT получают один IP → неравномерная нагрузка
- Если сервер падает, клиенты теряют сессию

```nginx
# Nginx
upstream backend {
    ip_hash;
    server backend1.example.com;
    server backend2.example.com;
    server backend3.example.com;
}
```

### 6. Consistent Hashing (консистентное хеширование)

Улучшенная версия IP Hash — минимизирует перераспределение при изменении числа серверов:

```
          0
          │
    ┌─────┴─────┐
    │           │
  255          64        ← Виртуальное кольцо 0-255
    │           │
    └─────┬─────┘
         128

Серверы размещаются на кольце по хешу:
Server A → позиция 50
Server B → позиция 120
Server C → позиция 200

Запрос хешируется → ищется ближайший сервер по часовой стрелке
hash(request) = 100 → Server B (ближайший после 100)
```

**При удалении Server B:**
- Только клиенты Server B перераспределятся
- Клиенты A и C останутся на своих серверах

```nginx
# Nginx (коммерческий модуль)
upstream backend {
    hash $request_uri consistent;
    server backend1.example.com;
    server backend2.example.com;
    server backend3.example.com;
}
```

### 7. Least Response Time (наименьшее время отклика)

Запрос отправляется на сервер с минимальным временем ответа:

```
Среднее время ответа:
Server A: 45ms
Server B: 23ms  ← выбираем
Server C: 67ms

Новый запрос → Server B
```

**Преимущества:**
- Оптимизирует пользовательский опыт
- Автоматически учитывает производительность серверов
- Адаптируется к изменениям в нагрузке

**Недостатки:**
- Требует постоянного мониторинга latency
- Может вызвать "стадное поведение" (все идут на быстрый сервер)
- Сложнее реализация

```nginx
# Nginx Plus (коммерческий)
upstream backend {
    least_time header;  # или last_byte
    server backend1.example.com;
    server backend2.example.com;
}
```

```haproxy
# HAProxy — использует leastconn с server-state
backend servers
    balance leastconn
    option httpchk GET /health
    server srv1 10.0.0.1:80 check observe layer7
    server srv2 10.0.0.2:80 check observe layer7
```

### 8. Random (случайный выбор)

Сервер выбирается случайным образом:

```python
import random
servers = ['A', 'B', 'C']
selected = random.choice(servers)
```

**Вариация — Random with Two Choices:**
1. Выбираем случайно 2 сервера
2. Отправляем запрос на менее загруженный

Этот подход даёт хорошее распределение с минимальными накладными расходами.

### Сравнительная таблица алгоритмов

| Алгоритм | Сложность | Учёт нагрузки | Sticky | Когда использовать |
|----------|-----------|---------------|--------|-------------------|
| Round Robin | Низкая | Нет | Нет | Однородные запросы и серверы |
| Weighted RR | Низкая | Частично | Нет | Разные мощности серверов |
| Least Connections | Средняя | Да | Нет | Долгие запросы, разная нагрузка |
| IP Hash | Низкая | Нет | Да | Простые sticky sessions |
| Consistent Hash | Средняя | Нет | Да | Кэширование, минимизация перераспределения |
| Least Response Time | Высокая | Да | Нет | Критична latency |
| Random | Низкая | Нет | Нет | Большое число серверов |

---

## Health Checks (проверки здоровья)

Health checks — механизм проверки работоспособности backend-серверов.

### Типы Health Checks

#### 1. Passive (пассивные)

Анализируют реальный трафик — если сервер возвращает ошибки, он помечается как нездоровый:

```
Клиент → LB → Server (500 Error)
                 │
                 └─ LB отмечает: "Server нездоров"
```

```nginx
# Nginx — пассивные проверки
upstream backend {
    server backend1.example.com max_fails=3 fail_timeout=30s;
    server backend2.example.com max_fails=3 fail_timeout=30s;
}
# max_fails=3 — после 3 ошибок сервер отключается
# fail_timeout=30s — на 30 секунд
```

#### 2. Active (активные)

Балансировщик сам отправляет проверочные запросы:

```
LB ─────────────────────────────────────────────────────►
   │                                                    │
   │  GET /health  ─────────────► Server 1 ─► 200 OK   │
   │  GET /health  ─────────────► Server 2 ─► 200 OK   │
   │  GET /health  ─────────────► Server 3 ─► 503      │
   │                                          │        │
   │                                          └─ Исключить из пула
   │
   └─ Проверки каждые N секунд
```

```nginx
# Nginx Plus (коммерческий)
upstream backend {
    zone backend 64k;

    server backend1.example.com;
    server backend2.example.com;

    health_check interval=5s fails=3 passes=2;
    # interval=5s — проверка каждые 5 секунд
    # fails=3 — после 3 неудач считать нездоровым
    # passes=2 — после 2 успехов считать здоровым
}
```

```haproxy
# HAProxy — активные проверки
backend servers
    option httpchk GET /health HTTP/1.1\r\nHost:\ example.com

    server srv1 10.0.0.1:80 check inter 5s fall 3 rise 2
    server srv2 10.0.0.2:80 check inter 5s fall 3 rise 2

    # inter 5s — интервал проверки
    # fall 3 — 3 неудачи = нездоров
    # rise 2 — 2 успеха = здоров
```

### L4 vs L7 Health Checks

**L4 (TCP) проверки:**
```
LB: Попытка установить TCP соединение на порт 80
    ├─ Успех (SYN-ACK) → Сервер здоров
    └─ Неудача (timeout, RST) → Сервер нездоров
```

```haproxy
# HAProxy — TCP check
backend servers
    option tcp-check
    server srv1 10.0.0.1:80 check
```

**L7 (HTTP) проверки:**
```
LB: GET /health HTTP/1.1
    ├─ 200 OK → Здоров
    ├─ 503 Service Unavailable → Нездоров
    └─ Timeout → Нездоров
```

### Endpoint для Health Check

Рекомендуется создавать специальный endpoint:

```python
# FastAPI
from fastapi import FastAPI, status
from fastapi.responses import JSONResponse

app = FastAPI()

@app.get("/health")
async def health_check():
    # Базовая проверка
    return {"status": "healthy"}

@app.get("/health/ready")
async def readiness_check():
    """Проверка готовности обрабатывать запросы"""
    checks = {
        "database": check_database(),
        "cache": check_redis(),
        "external_api": check_external_service()
    }

    all_healthy = all(checks.values())
    status_code = status.HTTP_200_OK if all_healthy else status.HTTP_503_SERVICE_UNAVAILABLE

    return JSONResponse(
        content={"status": "ready" if all_healthy else "not_ready", "checks": checks},
        status_code=status_code
    )

@app.get("/health/live")
async def liveness_check():
    """Проверка что приложение живо (для K8s liveness probe)"""
    return {"status": "alive"}
```

### Паттерн: Graceful Shutdown

При выводе сервера из эксплуатации:

```
1. Сервер получает SIGTERM
2. Health endpoint возвращает 503 (но сервер ещё работает)
3. LB исключает сервер из пула (перестаёт слать новые запросы)
4. Сервер дорабатывает текущие запросы
5. Сервер завершает работу
```

```python
import signal
import asyncio

shutdown_event = asyncio.Event()

def handle_shutdown(signum, frame):
    shutdown_event.set()

signal.signal(signal.SIGTERM, handle_shutdown)

@app.get("/health")
async def health():
    if shutdown_event.is_set():
        return JSONResponse(
            content={"status": "shutting_down"},
            status_code=503
        )
    return {"status": "healthy"}
```

---

## Session Persistence (Sticky Sessions)

**Sticky sessions** — механизм, при котором все запросы от одного клиента направляются на один и тот же backend-сервер.

### Зачем нужны Sticky Sessions

```
Без sticky sessions:
Запрос 1 (login) → Server A → Сессия создана на A
Запрос 2 (profile) → Server B → Сессия не найдена! 401 Unauthorized

Со sticky sessions:
Запрос 1 (login) → Server A → Сессия создана на A
Запрос 2 (profile) → Server A → Сессия найдена, OK
```

### Способы реализации

#### 1. Cookie-based (рекомендуется для L7)

LB устанавливает специальную cookie:

```
Первый запрос:
Client → LB → Server A
             │
             └─ LB добавляет: Set-Cookie: SERVERID=srv-a

Следующие запросы:
Client ──Cookie: SERVERID=srv-a──► LB ──► Server A
```

```nginx
# Nginx Plus
upstream backend {
    server backend1.example.com;
    server backend2.example.com;

    sticky cookie srv_id expires=1h domain=.example.com path=/;
}
```

```haproxy
# HAProxy
backend servers
    balance roundrobin
    cookie SERVERID insert indirect nocache

    server srv1 10.0.0.1:80 cookie s1
    server srv2 10.0.0.2:80 cookie s2
```

#### 2. Source IP (для L4)

Используется хеш IP-адреса клиента:

```nginx
# Nginx
upstream backend {
    ip_hash;
    server backend1.example.com;
    server backend2.example.com;
}
```

**Проблемы:**
- Клиенты за NAT получат один IP
- При смене IP (мобильные) — потеря сессии
- При падении сервера — потеря всех сессий этого сервера

#### 3. Application-controlled

Приложение само указывает, на какой сервер направить:

```
Client → LB → App → Response с Header: X-Sticky-Server: srv-a
              │
              └─ LB запоминает: этот клиент → srv-a
```

### Альтернатива: Shared Session Storage

Вместо sticky sessions — общее хранилище сессий:

```
                    ┌───────────────┐
                    │     Redis     │
                    │  (sessions)   │
                    └───────┬───────┘
                            │
           ┌────────────────┼────────────────┐
           │                │                │
    ┌──────┴──────┐  ┌──────┴──────┐  ┌──────┴──────┐
    │  Server A   │  │  Server B   │  │  Server C   │
    │ (stateless) │  │ (stateless) │  │ (stateless) │
    └─────────────┘  └─────────────┘  └─────────────┘
```

**Преимущества:**
- Серверы stateless — легко масштабировать
- При падении сервера сессии не теряются
- Равномерное распределение нагрузки

```python
# FastAPI + Redis sessions
from fastapi import FastAPI, Request
from fastapi_sessions import SessionMiddleware
import redis

app = FastAPI()
redis_client = redis.Redis(host='redis', port=6379)

app.add_middleware(
    SessionMiddleware,
    session_backend=RedisBackend(redis_client),
    secret_key="super-secret-key"
)
```

### Когда использовать Sticky Sessions

| Сценарий | Рекомендация |
|----------|--------------|
| Legacy приложения с in-memory сессиями | Sticky sessions |
| Файловые загрузки (multipart) | Sticky sessions |
| WebSocket соединения | Sticky sessions |
| Stateless API с JWT | Не нужны |
| Современные приложения | Shared storage (Redis) |

---

## SSL/TLS Termination

**SSL termination** — расшифровка HTTPS трафика на балансировщике.

### Варианты обработки SSL

#### 1. SSL Termination (расшифровка на LB)

```
Client ──HTTPS──► LB ──HTTP──► Backend
         │        │
         │        └─ SSL сертификат здесь
         │           Расшифровывает трафик
         │
         └─ Зашифрованный трафик
```

**Преимущества:**
- Один сертификат на LB (проще управлять)
- Backend не тратит CPU на шифрование
- LB может инспектировать трафик (L7 routing)
- Проще отладка (можно смотреть трафик)

**Недостатки:**
- Трафик между LB и backend не зашифрован
- LB — точка, где данные в открытом виде

```nginx
# Nginx — SSL termination
server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate /etc/ssl/certs/example.com.crt;
    ssl_certificate_key /etc/ssl/private/example.com.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;

    location / {
        proxy_pass http://backend;  # HTTP к backend
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

#### 2. SSL Passthrough (сквозная передача)

```
Client ──HTTPS──► LB ──HTTPS──► Backend
                  │            │
                  │            └─ SSL сертификат здесь
                  │
                  └─ Просто пересылает зашифрованный трафик
```

**Преимущества:**
- End-to-end шифрование
- LB не видит содержимое (безопасность)
- Backend контролирует SSL

**Недостатки:**
- LB работает только на L4
- Нельзя использовать L7 фичи (routing по URL и т.д.)
- Нужны сертификаты на каждом backend

```haproxy
# HAProxy — SSL passthrough
frontend https_front
    bind *:443
    mode tcp
    default_backend https_back

backend https_back
    mode tcp
    balance roundrobin
    server srv1 10.0.0.1:443
    server srv2 10.0.0.2:443
```

#### 3. SSL Re-encryption (повторное шифрование)

```
Client ──HTTPS──► LB ──HTTPS──► Backend
         │        │      │
         │        │      └─ Внутренний сертификат
         │        │         (может быть self-signed)
         │        │
         │        └─ Публичный сертификат
         │           Расшифровывает, инспектирует, шифрует снова
```

**Преимущества:**
- End-to-end шифрование
- LB может инспектировать трафик (L7 routing)
- Разные сертификаты (публичный и внутренний)

**Недостатки:**
- Двойное шифрование — больше CPU
- Сложнее управление сертификатами

```nginx
# Nginx — SSL re-encryption
upstream backend {
    server backend1.example.com:443;
    server backend2.example.com:443;
}

server {
    listen 443 ssl;

    ssl_certificate /etc/ssl/public.crt;
    ssl_certificate_key /etc/ssl/public.key;

    location / {
        proxy_pass https://backend;
        proxy_ssl_verify on;
        proxy_ssl_trusted_certificate /etc/ssl/internal-ca.crt;
    }
}
```

### Рекомендации по SSL

1. **Используйте TLS 1.2+** — отключите TLS 1.0, 1.1
2. **Modern cipher suites** — ECDHE для forward secrecy
3. **HSTS** — принудительный HTTPS
4. **OCSP Stapling** — быстрая проверка сертификата

```nginx
# Nginx — оптимизированная SSL конфигурация
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
ssl_prefer_server_ciphers off;

# Session resumption для производительности
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 1d;
ssl_session_tickets off;

# HSTS
add_header Strict-Transport-Security "max-age=31536000" always;

# OCSP Stapling
ssl_stapling on;
ssl_stapling_verify on;
resolver 8.8.8.8 8.8.4.4 valid=300s;
```

---

## Типы Load Balancers

### Hardware vs Software

| Характеристика | Hardware LB | Software LB |
|---------------|-------------|-------------|
| Производительность | Очень высокая (ASIC чипы) | Высокая (зависит от CPU) |
| Стоимость | $$$$ (десятки-сотни тысяч $) | Бесплатно или недорого |
| Гибкость | Ограниченная | Высокая |
| Масштабирование | Покупка нового оборудования | Добавление серверов |
| Поддержка | Вендор | Сообщество или коммерческая |
| Примеры | F5 BIG-IP, Citrix ADC, A10 | Nginx, HAProxy, Envoy |

**Hardware LB используют:**
- Крупные enterprise с legacy инфраструктурой
- Экстремальные требования к производительности
- Специальные compliance требования

**Software LB — стандарт в современных системах:**
- Cloud-native архитектуры
- Kubernetes
- Микросервисы

### Популярные Software Load Balancers

#### HAProxy

**High Availability Proxy** — один из самых популярных и производительных.

**Характеристики:**
- L4 и L7 балансировка
- Очень высокая производительность
- Богатая статистика и мониторинг
- Гибкие ACL для маршрутизации
- Бесплатный и open-source

```haproxy
# /etc/haproxy/haproxy.cfg

global
    maxconn 50000
    log /dev/log local0
    stats socket /run/haproxy/admin.sock mode 660 level admin

defaults
    mode http
    timeout connect 5s
    timeout client 50s
    timeout server 50s
    option httplog
    option dontlognull

frontend http_front
    bind *:80
    bind *:443 ssl crt /etc/ssl/certs/example.pem

    # ACL для маршрутизации
    acl is_api path_beg /api
    acl is_static path_beg /static

    use_backend api_servers if is_api
    use_backend static_servers if is_static
    default_backend web_servers

backend web_servers
    balance roundrobin
    option httpchk GET /health
    server web1 10.0.0.1:8080 check
    server web2 10.0.0.2:8080 check

backend api_servers
    balance leastconn
    option httpchk GET /api/health
    server api1 10.0.1.1:8080 check
    server api2 10.0.1.2:8080 check

backend static_servers
    balance roundrobin
    server static1 10.0.2.1:80 check

# Статистика
listen stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 10s
    stats admin if LOCALHOST
```

#### Nginx

**Веб-сервер с функциями балансировки.**

**Характеристики:**
- L7 балансировка (L4 в Plus версии)
- Кэширование и сжатие
- Статический контент
- Reverse proxy
- Широко распространён

```nginx
# /etc/nginx/nginx.conf

upstream api_backend {
    least_conn;
    server api1.example.com:8080 weight=5;
    server api2.example.com:8080 weight=3;
    server api3.example.com:8080 backup;

    keepalive 32;  # Connection pooling
}

upstream websocket_backend {
    ip_hash;  # Sticky для WebSocket
    server ws1.example.com:8080;
    server ws2.example.com:8080;
}

server {
    listen 80;
    listen 443 ssl;
    server_name example.com;

    ssl_certificate /etc/ssl/certs/example.crt;
    ssl_certificate_key /etc/ssl/private/example.key;

    # API
    location /api/ {
        proxy_pass http://api_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeouts
        proxy_connect_timeout 5s;
        proxy_read_timeout 60s;
    }

    # WebSocket
    location /ws/ {
        proxy_pass http://websocket_backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    # Статика с кэшированием
    location /static/ {
        alias /var/www/static/;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
}
```

#### Envoy Proxy

**Современный L7 proxy для микросервисов.**

**Характеристики:**
- Создан для cloud-native (Lyft)
- L4 и L7 балансировка
- gRPC, HTTP/2 из коробки
- Observability (metrics, tracing, logging)
- Dynamic configuration (xDS API)
- Основа для Istio service mesh

```yaml
# envoy.yaml
static_resources:
  listeners:
    - name: listener_0
      address:
        socket_address:
          address: 0.0.0.0
          port_value: 80
      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                stat_prefix: ingress_http
                route_config:
                  name: local_route
                  virtual_hosts:
                    - name: backend
                      domains: ["*"]
                      routes:
                        - match:
                            prefix: "/api"
                          route:
                            cluster: api_cluster
                        - match:
                            prefix: "/"
                          route:
                            cluster: web_cluster
                http_filters:
                  - name: envoy.filters.http.router
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router

  clusters:
    - name: api_cluster
      connect_timeout: 5s
      type: STRICT_DNS
      lb_policy: LEAST_REQUEST
      health_checks:
        - timeout: 5s
          interval: 10s
          unhealthy_threshold: 3
          healthy_threshold: 2
          http_health_check:
            path: "/health"
      load_assignment:
        cluster_name: api_cluster
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: api1.example.com
                      port_value: 8080
              - endpoint:
                  address:
                    socket_address:
                      address: api2.example.com
                      port_value: 8080

    - name: web_cluster
      connect_timeout: 5s
      type: STRICT_DNS
      lb_policy: ROUND_ROBIN
      load_assignment:
        cluster_name: web_cluster
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: web1.example.com
                      port_value: 8080
```

### AWS Load Balancers

#### ELB — Elastic Load Balancer (Classic, устаревший)

L4 и ограниченный L7. Используйте ALB или NLB вместо него.

#### ALB — Application Load Balancer

**L7 балансировщик для HTTP/HTTPS.**

**Возможности:**
- Path-based routing (`/api/*` → API target group)
- Host-based routing (`api.example.com` → API)
- WebSocket, HTTP/2
- Lambda интеграция
- Аутентификация (OIDC, Cognito)

```
                    ┌─────────────────────────────────────┐
                    │              ALB                    │
                    │                                     │
                    │  Rules:                             │
                    │  1. path=/api/* → API Target Group  │
                    │  2. path=/static/* → S3 bucket      │
                    │  3. default → Web Target Group      │
                    └─────────────────────────────────────┘
                             │         │         │
                    ┌────────┘         │         └────────┐
                    ▼                  ▼                  ▼
            ┌───────────┐      ┌───────────┐      ┌───────────┐
            │ API TG    │      │ S3 Bucket │      │ Web TG    │
            │ (EC2/ECS) │      │ (static)  │      │ (EC2/ECS) │
            └───────────┘      └───────────┘      └───────────┘
```

#### NLB — Network Load Balancer

**L4 балансировщик для TCP/UDP.**

**Возможности:**
- Миллионы запросов в секунду
- Ультра-низкая латентность (~100 микросекунд)
- Static IP / Elastic IP
- Preserve source IP
- TLS termination

**Когда использовать:**
- Высокопроизводительные TCP сервисы
- gRPC
- Игровые серверы
- IoT
- Non-HTTP протоколы

#### GLB — Gateway Load Balancer

**L3 балансировщик для виртуальных appliances.**

Используется для:
- Файрволов
- IDS/IPS
- Deep packet inspection

### Сравнение облачных LB

| Характеристика | AWS ALB | AWS NLB | GCP HTTP(S) LB | Azure App Gateway |
|---------------|---------|---------|----------------|-------------------|
| Уровень | L7 | L4 | L7 | L7 |
| Протоколы | HTTP, HTTPS, WS | TCP, UDP, TLS | HTTP, HTTPS | HTTP, HTTPS, WS |
| SSL termination | Да | Да | Да | Да |
| Static IP | Нет | Да | Да (Anycast) | Да |
| Path routing | Да | Нет | Да | Да |
| WebSocket | Да | Да | Да | Да |
| Auto-scaling | Да | Да | Да | Да |

---

## High Availability для Load Balancers

Если LB — единственная точка входа, то он становится **Single Point of Failure (SPOF)**.

### Active-Passive (Failover)

```
                    ┌─────────────────┐
                    │   Floating IP   │
                    │   (Virtual IP)  │
                    └────────┬────────┘
                             │
           ┌─────────────────┴─────────────────┐
           │                                   │
    ┌──────┴──────┐                     ┌──────┴──────┐
    │   LB Active │ ◄── heartbeat ──►   │  LB Standby │
    │   (Master)  │                     │   (Backup)  │
    └──────┬──────┘                     └─────────────┘
           │
     Весь трафик
           │
    ┌──────┴──────┐
    │   Backends  │
    └─────────────┘

При падении Active:
- Standby обнаруживает отсутствие heartbeat
- Standby забирает Floating IP себе
- Становится новым Active
```

**Реализация с Keepalived (VRRP):**

```bash
# На Master LB (/etc/keepalived/keepalived.conf)
vrrp_script check_haproxy {
    script "/usr/bin/killall -0 haproxy"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 101
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass secret123
    }

    virtual_ipaddress {
        192.168.1.100/24  # Floating IP
    }

    track_script {
        check_haproxy
    }
}
```

```bash
# На Backup LB
vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 51
    priority 100  # Ниже чем у Master
    # ... остальное такое же
}
```

### Active-Active

```
                    ┌─────────────────────────┐
                    │          DNS            │
                    │   example.com           │
                    │   A: 1.1.1.1 (LB1)      │
                    │   A: 2.2.2.2 (LB2)      │
                    └────────────┬────────────┘
                                 │
                    ┌────────────┴────────────┐
                    │                         │
             ┌──────┴──────┐           ┌──────┴──────┐
             │    LB 1     │           │    LB 2     │
             │   Active    │           │   Active    │
             └──────┬──────┘           └──────┬──────┘
                    │                         │
                    └────────────┬────────────┘
                                 │
                          ┌──────┴──────┐
                          │   Backends  │
                          └─────────────┘
```

**Преимущества:**
- Оба LB обрабатывают трафик
- Лучшая утилизация ресурсов
- Масштабирование добавлением LB

**Реализация:**
- DNS Round Robin
- Anycast (для глобальной балансировки)
- L4 балансировщик перед L7 (двухуровневая схема)

### Глобальная балансировка (GSLB)

```
                              ┌─────────────────┐
                              │   Global DNS    │
                              │    (Route 53,   │
                              │   Cloudflare)   │
                              └────────┬────────┘
                                       │
            Геолокация / Latency-based routing
                                       │
          ┌────────────────────────────┼────────────────────────────┐
          │                            │                            │
          ▼                            ▼                            ▼
    ┌───────────┐                ┌───────────┐                ┌───────────┐
    │  US East  │                │  EU West  │                │ Asia Pac  │
    │   Region  │                │   Region  │                │   Region  │
    └─────┬─────┘                └─────┬─────┘                └─────┬─────┘
          │                            │                            │
     ┌────┴────┐                  ┌────┴────┐                  ┌────┴────┐
     │   LB    │                  │   LB    │                  │   LB    │
     └────┬────┘                  └────┬────┘                  └────┬────┘
          │                            │                            │
     ┌────┴────┐                  ┌────┴────┐                  ┌────┴────┐
     │ Servers │                  │ Servers │                  │ Servers │
     └─────────┘                  └─────────┘                  └─────────┘
```

**AWS Route 53 пример:**
- Latency-based routing — направляет в ближайший регион
- Geolocation routing — по географии клиента
- Failover routing — резервный регион при падении
- Weighted routing — распределение по весам

---

## Примеры конфигурации и Best Practices

### Production-ready HAProxy конфигурация

```haproxy
#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    maxconn 100000
    log 127.0.0.1 local0 info
    chroot /var/lib/haproxy
    pidfile /var/run/haproxy.pid
    user haproxy
    group haproxy
    daemon

    # SSL/TLS
    ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256
    ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

    # Stats socket для управления
    stats socket /var/run/haproxy.sock mode 600 level admin
    stats timeout 30s

#---------------------------------------------------------------------
# Defaults
#---------------------------------------------------------------------
defaults
    mode http
    log global
    option httplog
    option dontlognull
    option http-server-close
    option forwardfor except 127.0.0.0/8
    option redispatch

    retries 3
    timeout http-request 10s
    timeout queue 1m
    timeout connect 10s
    timeout client 1m
    timeout server 1m
    timeout http-keep-alive 10s
    timeout check 10s

    maxconn 50000

    # Сжатие
    compression algo gzip
    compression type text/html text/plain text/css application/json

#---------------------------------------------------------------------
# Frontend: HTTP → HTTPS redirect
#---------------------------------------------------------------------
frontend http_front
    bind *:80

    # Security headers
    http-response set-header X-Frame-Options DENY
    http-response set-header X-Content-Type-Options nosniff
    http-response set-header X-XSS-Protection "1; mode=block"

    # Редирект на HTTPS
    redirect scheme https code 301 if !{ ssl_fc }

#---------------------------------------------------------------------
# Frontend: HTTPS
#---------------------------------------------------------------------
frontend https_front
    bind *:443 ssl crt /etc/ssl/private/combined.pem alpn h2,http/1.1

    # HSTS
    http-response set-header Strict-Transport-Security "max-age=31536000; includeSubDomains"

    # Rate limiting (защита от DDoS)
    stick-table type ip size 100k expire 30s store http_req_rate(10s)
    http-request track-sc0 src
    http-request deny deny_status 429 if { sc_http_req_rate(0) gt 100 }

    # ACL для маршрутизации
    acl is_api path_beg /api/
    acl is_websocket hdr(Upgrade) -i websocket
    acl is_health path /health

    # Маршрутизация
    use_backend health_check if is_health
    use_backend websocket_servers if is_websocket
    use_backend api_servers if is_api
    default_backend web_servers

#---------------------------------------------------------------------
# Backend: Web servers
#---------------------------------------------------------------------
backend web_servers
    balance roundrobin
    option httpchk GET /health HTTP/1.1\r\nHost:\ example.com

    # Connection pooling
    http-reuse always

    # Headers
    http-request set-header X-Forwarded-Proto https
    http-request set-header X-Real-IP %[src]

    server web1 10.0.0.1:8080 check inter 5s fall 3 rise 2 weight 100
    server web2 10.0.0.2:8080 check inter 5s fall 3 rise 2 weight 100
    server web3 10.0.0.3:8080 check inter 5s fall 3 rise 2 weight 100 backup

#---------------------------------------------------------------------
# Backend: API servers
#---------------------------------------------------------------------
backend api_servers
    balance leastconn
    option httpchk GET /api/health HTTP/1.1\r\nHost:\ example.com

    # Sticky sessions по cookie
    cookie SERVERID insert indirect nocache

    # Timeout для долгих запросов
    timeout server 5m

    server api1 10.0.1.1:8080 check cookie s1 inter 5s fall 3 rise 2
    server api2 10.0.1.2:8080 check cookie s2 inter 5s fall 3 rise 2

#---------------------------------------------------------------------
# Backend: WebSocket servers
#---------------------------------------------------------------------
backend websocket_servers
    balance source

    # Долгие таймауты для WebSocket
    timeout tunnel 1h

    server ws1 10.0.2.1:8080 check
    server ws2 10.0.2.2:8080 check

#---------------------------------------------------------------------
# Backend: Health check (для LB health)
#---------------------------------------------------------------------
backend health_check
    http-request return status 200 content-type text/plain string "OK"

#---------------------------------------------------------------------
# Statistics
#---------------------------------------------------------------------
listen stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 5s
    stats auth admin:secure_password
    stats admin if TRUE
```

### Production-ready Nginx конфигурация

```nginx
# /etc/nginx/nginx.conf

user nginx;
worker_processes auto;
worker_rlimit_nofile 65535;

error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 4096;
    use epoll;
    multi_accept on;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Logging
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for" '
                    'rt=$request_time uct=$upstream_connect_time '
                    'uht=$upstream_header_time urt=$upstream_response_time';

    access_log /var/log/nginx/access.log main buffer=16k flush=5s;

    # Performance
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    keepalive_requests 100;

    # Buffers
    client_body_buffer_size 10K;
    client_header_buffer_size 1k;
    client_max_body_size 100m;
    large_client_header_buffers 4 32k;

    # Compression
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css application/json application/javascript
               text/xml application/xml application/xml+rss text/javascript;

    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
    limit_conn_zone $binary_remote_addr zone=conn_limit:10m;

    # Upstreams
    upstream api_backend {
        least_conn;
        keepalive 32;

        server api1.internal:8080 weight=5 max_fails=3 fail_timeout=30s;
        server api2.internal:8080 weight=5 max_fails=3 fail_timeout=30s;
        server api3.internal:8080 weight=5 max_fails=3 fail_timeout=30s backup;
    }

    upstream web_backend {
        least_conn;
        keepalive 32;

        server web1.internal:8080 max_fails=3 fail_timeout=30s;
        server web2.internal:8080 max_fails=3 fail_timeout=30s;
    }

    upstream websocket_backend {
        ip_hash;

        server ws1.internal:8080;
        server ws2.internal:8080;
    }

    # SSL settings
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;
    ssl_stapling on;
    ssl_stapling_verify on;

    # HTTP → HTTPS redirect
    server {
        listen 80 default_server;
        listen [::]:80 default_server;
        server_name _;
        return 301 https://$host$request_uri;
    }

    # Main server
    server {
        listen 443 ssl http2;
        listen [::]:443 ssl http2;
        server_name example.com;

        ssl_certificate /etc/ssl/certs/example.com.crt;
        ssl_certificate_key /etc/ssl/private/example.com.key;

        # Security headers
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header X-XSS-Protection "1; mode=block" always;
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

        # Health check
        location /health {
            access_log off;
            return 200 "OK\n";
            add_header Content-Type text/plain;
        }

        # API
        location /api/ {
            limit_req zone=api_limit burst=20 nodelay;
            limit_conn conn_limit 10;

            proxy_pass http://api_backend;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            proxy_connect_timeout 5s;
            proxy_send_timeout 60s;
            proxy_read_timeout 60s;

            # Retry на других серверах при ошибке
            proxy_next_upstream error timeout http_500 http_502 http_503;
            proxy_next_upstream_tries 2;
        }

        # WebSocket
        location /ws/ {
            proxy_pass http://websocket_backend;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;

            proxy_read_timeout 86400s;
            proxy_send_timeout 86400s;
        }

        # Static files
        location /static/ {
            alias /var/www/static/;
            expires 30d;
            add_header Cache-Control "public, immutable";
            access_log off;
        }

        # Web application
        location / {
            proxy_pass http://web_backend;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            proxy_connect_timeout 5s;
            proxy_read_timeout 60s;
        }
    }
}
```

### Best Practices

#### 1. Connection Pooling

Держите соединения между LB и backend открытыми:

```nginx
upstream backend {
    keepalive 32;  # Пул из 32 keep-alive соединений
    server backend1:8080;
}

location / {
    proxy_pass http://backend;
    proxy_http_version 1.1;
    proxy_set_header Connection "";  # Для keep-alive
}
```

#### 2. Graceful degradation

```haproxy
backend api_servers
    # Основные серверы
    server api1 10.0.0.1:8080 check
    server api2 10.0.0.2:8080 check

    # Backup сервер — используется только если все основные упали
    server api_backup 10.0.0.10:8080 check backup

    # Sorry сервер — статичная заглушка
    server sorry_server 10.0.0.100:80 check backup
```

#### 3. Timeouts

```
Client ─────► LB ─────► Backend
       │           │
       │           └─ proxy_read_timeout (ожидание ответа)
       │              proxy_connect_timeout (установка соединения)
       │
       └─ client_body_timeout (получение тела запроса)
          send_timeout (отправка ответа клиенту)
```

Правило: `client_timeout > server_timeout`

#### 4. Мониторинг и алерты

**Ключевые метрики:**
- Request rate (RPS)
- Error rate (4xx, 5xx)
- Latency (p50, p95, p99)
- Active connections
- Backend health status
- Queue length

```haproxy
# HAProxy экспортирует метрики на /stats
# Или через Prometheus exporter
listen stats
    bind *:8404
    stats enable
    stats uri /stats
```

#### 5. Логирование для отладки

```nginx
log_format detailed '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent" '
                    'upstream_addr=$upstream_addr '
                    'upstream_status=$upstream_status '
                    'request_time=$request_time '
                    'upstream_response_time=$upstream_response_time';
```

#### 6. Security

```nginx
# Rate limiting
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

# Блокировка по GeoIP
# geoip_country /usr/share/GeoIP/GeoIP.dat;
# if ($geoip_country_code = "XX") { return 403; }

# Базовая защита от DDoS
client_body_timeout 10s;
client_header_timeout 10s;

# Максимум соединений с одного IP
limit_conn_zone $binary_remote_addr zone=addr:10m;
limit_conn addr 100;
```

---

## Резюме

| Аспект | Рекомендации |
|--------|--------------|
| **Уровень** | L7 для HTTP/микросервисов, L4 для производительности |
| **Алгоритм** | Least Connections для неравномерной нагрузки, Round Robin для простых случаев |
| **Health Checks** | Active + Passive, HTTP endpoint `/health` |
| **Sessions** | Shared storage (Redis) > Sticky sessions |
| **SSL** | Termination на LB, TLS 1.2+ |
| **HA** | Active-Passive с VRRP или Active-Active с DNS |
| **Инструменты** | HAProxy, Nginx, облачные ALB/NLB |

### Чеклист для production

- [ ] Настроен health check для всех backend
- [ ] SSL termination с современными cipher suites
- [ ] Connection pooling между LB и backend
- [ ] Timeouts настроены правильно
- [ ] Rate limiting для защиты от DDoS
- [ ] Мониторинг и алерты настроены
- [ ] HA для самого LB (VRRP или мульти-AZ)
- [ ] Логирование с upstream метриками
- [ ] Graceful shutdown в приложениях
- [ ] Backup серверы для degradation
