# HTTP Versions (Версии HTTP)

## Обзор эволюции HTTP

| Версия | Год | Ключевые особенности |
|--------|-----|---------------------|
| HTTP/0.9 | 1991 | Только GET, без заголовков |
| HTTP/1.0 | 1996 | Заголовки, методы, статус коды |
| HTTP/1.1 | 1997 | Keep-alive, chunked, Host |
| HTTP/2 | 2015 | Мультиплексирование, сжатие заголовков |
| HTTP/3 | 2022 | QUIC вместо TCP, 0-RTT |

## HTTP/0.9 (1991)

Первая версия HTTP. Крайне простой протокол.

### Особенности
- Только метод GET
- Нет заголовков
- Только HTML-документы
- Соединение закрывается после ответа

```http
# Запрос
GET /page.html

# Ответ (только HTML, без статуса и заголовков)
<html>...</html>
```

## HTTP/1.0 (1996)

Значительное расширение функциональности.

### Нововведения
- Версия протокола в запросе
- Заголовки запроса и ответа
- Статус коды
- Content-Type (не только HTML)
- Методы POST, HEAD
- Авторизация

### Ограничения
- Одно соединение = один запрос (новое TCP-соединение для каждого запроса)
- Нет обязательного Host заголовка

```http
# Запрос
GET /page.html HTTP/1.0
User-Agent: Mozilla/1.0

# Ответ
HTTP/1.0 200 OK
Content-Type: text/html
Content-Length: 1234

<html>...</html>
```

## HTTP/1.1 (1997)

Самая долгоживущая версия. До сих пор широко используется.

### Ключевые улучшения

#### 1. Persistent Connections (Keep-Alive)

По умолчанию соединение остается открытым для повторных запросов.

```http
GET /page1.html HTTP/1.1
Host: example.com
Connection: keep-alive

HTTP/1.1 200 OK
Connection: keep-alive

# Тот же сокет
GET /page2.html HTTP/1.1
Host: example.com
```

#### 2. Обязательный Host заголовок

Позволяет виртуальный хостинг (несколько сайтов на одном IP).

```http
GET /page.html HTTP/1.1
Host: example.com  # Обязательный
```

#### 3. Chunked Transfer Encoding

Отправка ответа частями без знания полного размера.

```http
HTTP/1.1 200 OK
Transfer-Encoding: chunked

7\r\n
Mozilla\r\n
9\r\n
Developer\r\n
0\r\n
\r\n
```

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
import asyncio

app = FastAPI()

async def generate_data():
    for i in range(10):
        yield f"data chunk {i}\n"
        await asyncio.sleep(0.5)

@app.get("/stream")
def stream_data():
    return StreamingResponse(
        generate_data(),
        media_type="text/plain"
    )
```

#### 4. Pipelining

Отправка нескольких запросов без ожидания ответов (редко используется).

```
Клиент                    Сервер
   |                         |
   |-- GET /1 -------------->|
   |-- GET /2 -------------->|
   |-- GET /3 -------------->|
   |                         |
   |<-- Response 1 ----------|
   |<-- Response 2 ----------|
   |<-- Response 3 ----------|
```

#### 5. Кэширование

Улучшенные механизмы: Cache-Control, ETag, If-None-Match.

```http
# Запрос с условием
GET /resource HTTP/1.1
If-None-Match: "abc123"

# Ответ (ресурс не изменился)
HTTP/1.1 304 Not Modified
ETag: "abc123"
```

#### 6. Дополнительные методы

PUT, DELETE, OPTIONS, TRACE, CONNECT.

### Ограничения HTTP/1.1

1. **Head-of-Line Blocking**: Ответы должны приходить в порядке запросов
2. **Избыточные заголовки**: Повторяются с каждым запросом
3. **Ограниченный параллелизм**: Браузеры открывают 6-8 соединений на домен

## HTTP/2 (2015)

Бинарный протокол с множеством оптимизаций.

### Ключевые особенности

#### 1. Бинарный протокол

Вместо текстового формата — бинарные фреймы.

```
# HTTP/1.1 (текстовый)
GET /resource HTTP/1.1
Host: example.com

# HTTP/2 (бинарный)
+-----------------------------------------------+
|                 Length (24)                   |
+---------------+---------------+---------------+
|   Type (8)    |   Flags (8)   |
+-+-------------+---------------+-------------------------------+
|R|                 Stream Identifier (31)                      |
+=+=============================================================+
|                   Frame Payload (0...)                        |
+---------------------------------------------------------------+
```

#### 2. Мультиплексирование

Несколько запросов и ответов одновременно через одно TCP-соединение.

```
           Одно TCP соединение
+----------------------------------------------+
| Stream 1: GET /style.css                     |
| Stream 3: GET /script.js                     |
| Stream 5: GET /image.png                     |
| Stream 1: Response (CSS данные)              |
| Stream 3: Response (JS данные)               |
| Stream 5: Response (Image данные)            |
+----------------------------------------------+
```

Нет Head-of-Line Blocking на уровне HTTP (но есть на уровне TCP).

#### 3. Сжатие заголовков (HPACK)

Заголовки сжимаются и индексируются.

```
# HTTP/1.1 - повторение заголовков
GET /1 HTTP/1.1
Host: example.com
User-Agent: Mozilla/5.0
Accept: */*

GET /2 HTTP/1.1
Host: example.com          # Повторяется
User-Agent: Mozilla/5.0    # Повторяется
Accept: */*                # Повторяется

# HTTP/2 - индексирование
# Первый запрос: полные заголовки
# Второй запрос: только индексы (62, 63, 64)
```

#### 4. Server Push

Сервер может отправлять ресурсы до их запроса клиентом.

```python
# Концептуально (зависит от веб-сервера)
# Клиент запрашивает /page.html
# Сервер также отправляет /style.css и /script.js

# В nginx
location / {
    http2_push /style.css;
    http2_push /script.js;
}
```

> **Примечание**: Server Push оказался менее полезным, чем ожидалось, и в HTTP/3 его нет.

#### 5. Приоритизация потоков

Клиент указывает приоритеты для ресурсов.

```
Stream 1 (CSS):    вес 256, зависит от root
Stream 3 (JS):     вес 128, зависит от Stream 1
Stream 5 (Image):  вес 32,  зависит от root
```

### Использование HTTP/2

```python
# uvicorn с HTTP/2
# uvicorn main:app --host 0.0.0.0 --port 443 --ssl-keyfile key.pem --ssl-certfile cert.pem --http h2

# hypercorn с HTTP/2
# hypercorn main:app --bind 0.0.0.0:443 --certfile cert.pem --keyfile key.pem

# Клиент httpx с HTTP/2
import httpx

async with httpx.AsyncClient(http2=True) as client:
    response = await client.get("https://example.com")
    print(response.http_version)  # "HTTP/2"
```

### HTTP/2 требует HTTPS

Технически HTTP/2 может работать без шифрования (h2c), но браузеры требуют HTTPS.

## HTTP/3 (2022)

Использует QUIC вместо TCP.

### Почему QUIC?

TCP имеет проблему Head-of-Line Blocking: потеря одного пакета блокирует все потоки. QUIC решает это, работая поверх UDP.

```
HTTP/1.1 over TCP:
[HTTP] -> [TCP] -> [IP]

HTTP/2 over TCP:
[HTTP/2] -> [TCP] -> [IP]

HTTP/3 over QUIC:
[HTTP/3] -> [QUIC] -> [UDP] -> [IP]
```

### Ключевые особенности

#### 1. Независимые потоки

Потеря пакета в одном потоке не влияет на другие.

```
# TCP (HTTP/2): потеря пакета блокирует все
Stream 1: [пакет 1] [пакет 2] [ПОТЕРЯН] [пакет 4]
Stream 2: [блокировано...]
Stream 3: [блокировано...]

# QUIC (HTTP/3): потоки независимы
Stream 1: [пакет 1] [пакет 2] [ждет пакет 3...]
Stream 2: [пакет 1] [пакет 2] [пакет 3] ✓
Stream 3: [пакет 1] [пакет 2] [пакет 3] ✓
```

#### 2. 0-RTT соединение

Повторное соединение без полного handshake.

```
# TCP + TLS (HTTP/2): 2-3 RTT
Клиент -> SYN -> Сервер
Клиент <- SYN-ACK <- Сервер
Клиент -> ACK -> Сервер
... TLS handshake ...

# QUIC (HTTP/3): 0-1 RTT для повторных соединений
Клиент -> Запрос + ранее сохраненные ключи -> Сервер
Клиент <- Ответ <- Сервер
```

#### 3. Connection Migration

Соединение сохраняется при смене IP (например, Wi-Fi -> 4G).

```
# TCP: соединение теряется при смене IP
Wi-Fi: 192.168.1.100 -> connection established
4G:    10.0.0.50     -> new connection required

# QUIC: Connection ID сохраняет соединение
Wi-Fi: 192.168.1.100 -> Connection ID: abc123
4G:    10.0.0.50     -> Connection ID: abc123 (то же соединение)
```

#### 4. Встроенное шифрование

TLS 1.3 интегрирован в QUIC. Нет незашифрованного варианта.

### Использование HTTP/3

```python
# httpx пока не поддерживает HTTP/3 нативно
# Используйте aioquic или специализированные клиенты

# Проверка поддержки HTTP/3 сервером
import subprocess
result = subprocess.run(
    ["curl", "--http3", "-I", "https://cloudflare.com"],
    capture_output=True
)
print(result.stdout.decode())
```

### Alt-Svc заголовок

Сервер сообщает о поддержке HTTP/3:

```http
HTTP/2 200 OK
Alt-Svc: h3=":443"; ma=86400
```

## Сравнение версий

| Аспект | HTTP/1.1 | HTTP/2 | HTTP/3 |
|--------|----------|--------|--------|
| Транспорт | TCP | TCP | QUIC/UDP |
| Формат | Текстовый | Бинарный | Бинарный |
| Мультиплексирование | Нет | Да | Да |
| HOL Blocking | Да | TCP уровень | Нет |
| Сжатие заголовков | Нет | HPACK | QPACK |
| Server Push | Нет | Да | Удален |
| 0-RTT | Нет | Нет | Да |
| Connection Migration | Нет | Нет | Да |
| Шифрование | Опционально | Практически обязательно | Обязательно |

## Настройка серверов

### Nginx

```nginx
# HTTP/2
server {
    listen 443 ssl http2;
    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;
}

# HTTP/3 (nginx 1.25+)
server {
    listen 443 ssl;
    listen 443 quic reuseport;
    http2 on;
    http3 on;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    # Сообщаем клиенту о HTTP/3
    add_header Alt-Svc 'h3=":443"; ma=86400';
}
```

### Caddy (автоматическая поддержка HTTP/3)

```caddyfile
example.com {
    # HTTP/3 включен по умолчанию
    reverse_proxy localhost:8000
}
```

### Python веб-серверы

```python
# uvicorn с HTTP/2
# pip install uvicorn[standard] httptools

import uvicorn

if __name__ == "__main__":
    uvicorn.run(
        "main:app",
        host="0.0.0.0",
        port=443,
        ssl_keyfile="key.pem",
        ssl_certfile="cert.pem",
        http="h2"  # HTTP/2
    )
```

```python
# hypercorn с HTTP/2 и HTTP/3
# pip install hypercorn[h3]

# hypercorn main:app --quic-bind 0.0.0.0:443 --bind 0.0.0.0:443 \
#   --certfile cert.pem --keyfile key.pem
```

## Определение версии

```python
import httpx

# Синхронный клиент (HTTP/1.1)
response = httpx.get("https://example.com")
print(response.http_version)  # "HTTP/1.1"

# Асинхронный с HTTP/2
async with httpx.AsyncClient(http2=True) as client:
    response = await client.get("https://example.com")
    print(response.http_version)  # "HTTP/2" если сервер поддерживает
```

```bash
# curl
curl --http1.1 -I https://example.com
curl --http2 -I https://example.com
curl --http3 -I https://example.com  # Требует сборку с HTTP/3

# Просмотр протокола
curl -sI https://cloudflare.com | grep -i "http/"
```

## Best Practices

### 1. Используйте HTTP/2 или HTTP/3 когда возможно

```python
# Включите HTTP/2 на production сервере
# Это бесплатная оптимизация производительности
```

### 2. Для API достаточно HTTP/1.1

```python
# Большинство API клиентов отлично работают с HTTP/1.1
# HTTP/2 больше важен для веб-страниц с множеством ресурсов
```

### 3. Всегда используйте HTTPS

```python
# HTTP/2 и HTTP/3 практически требуют HTTPS
# HTTP/1.1 тоже должен использовать HTTPS
```

### 4. Настройте Alt-Svc для HTTP/3

```python
from fastapi import FastAPI

app = FastAPI()

@app.middleware("http")
async def add_alt_svc(request, call_next):
    response = await call_next(request)
    response.headers["Alt-Svc"] = 'h3=":443"; ma=86400'
    return response
```

## Типичные ошибки

1. **Использование устаревших клиентов**
   ```python
   # Старые версии requests не поддерживают HTTP/2
   # Используйте httpx для HTTP/2
   ```

2. **Забытые сертификаты для HTTP/2**
   ```python
   # HTTP/2 требует HTTPS (в браузерах)
   # Используйте Let's Encrypt или self-signed для разработки
   ```

3. **Игнорирование HTTP/3 firewall**
   ```python
   # QUIC использует UDP порт 443
   # Убедитесь, что firewall пропускает UDP трафик
   ```

4. **Неоптимизированный HTTP/1.1**
   ```python
   # Используйте keep-alive
   # Используйте gzip сжатие
   # Используйте кэширование
   ```
