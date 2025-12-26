# HTTP (HyperText Transfer Protocol)

## Что такое HTTP

**HTTP (HyperText Transfer Protocol)** — это протокол прикладного уровня для передачи данных в интернете. Изначально создан для передачи HTML-документов, но сегодня используется для передачи любых данных: изображений, видео, JSON, XML и т.д.

HTTP работает по модели **клиент-сервер**:
1. **Клиент** (браузер, мобильное приложение, curl) отправляет запрос
2. **Сервер** обрабатывает запрос и отправляет ответ
3. Соединение закрывается (в HTTP/1.0) или остается открытым (keep-alive в HTTP/1.1+)

## Основные характеристики HTTP

### 1. Stateless (Без сохранения состояния)

HTTP не хранит информацию о предыдущих запросах. Каждый запрос независим.

```python
# Каждый запрос должен содержать всю необходимую информацию
# Сервер не помнит предыдущие запросы

# Запрос 1: Аутентификация
POST /login
{"username": "john", "password": "secret"}
# Ответ: token = "abc123"

# Запрос 2: Получение данных (нужно передать токен заново)
GET /profile
Authorization: Bearer abc123
```

### 2. Текстовый протокол

HTTP использует текстовый формат для заголовков, что упрощает отладку.

```http
GET /api/users HTTP/1.1
Host: api.example.com
Accept: application/json
```

### 3. Request-Response модель

Каждый запрос получает ровно один ответ.

```
Клиент                    Сервер
   |                         |
   |------- Запрос --------->|
   |                         |
   |<------ Ответ -----------|
   |                         |
```

## Структура HTTP-запроса

```http
POST /api/users HTTP/1.1          <- Стартовая строка
Host: api.example.com             <- Заголовки
Content-Type: application/json    <-
Authorization: Bearer token123    <-
                                  <- Пустая строка
{                                 <- Тело запроса
    "name": "John",
    "email": "john@example.com"
}
```

### Компоненты запроса:

1. **Стартовая строка (Request Line)**
   - Метод (GET, POST, PUT, DELETE и т.д.)
   - URI (путь к ресурсу)
   - Версия протокола (HTTP/1.1)

2. **Заголовки (Headers)**
   - Метаинформация о запросе
   - Формат: `Имя: Значение`

3. **Пустая строка**
   - Разделитель между заголовками и телом

4. **Тело запроса (Body)**
   - Опционально
   - Данные, отправляемые на сервер

## Структура HTTP-ответа

```http
HTTP/1.1 200 OK                      <- Строка статуса
Content-Type: application/json       <- Заголовки
Content-Length: 85                   <-
Date: Mon, 15 Jan 2024 10:30:00 GMT  <-
                                     <- Пустая строка
{                                    <- Тело ответа
    "id": 123,
    "name": "John",
    "email": "john@example.com"
}
```

### Компоненты ответа:

1. **Строка статуса (Status Line)**
   - Версия протокола
   - Код статуса (200, 404, 500 и т.д.)
   - Текстовое описание статуса

2. **Заголовки ответа**
   - Метаинформация об ответе

3. **Тело ответа**
   - Данные, возвращаемые сервером

## HTTP vs HTTPS

**HTTPS (HTTP Secure)** — это HTTP с шифрованием через TLS/SSL.

```
HTTP:  http://example.com  (порт 80)
HTTPS: https://example.com (порт 443)
```

### Как работает HTTPS:

```
Клиент                           Сервер
   |                                |
   |------ ClientHello ------------>|
   |<----- ServerHello + Сертификат-|
   |                                |
   |-- Проверка сертификата        |
   |-- Генерация ключа сессии      |
   |                                |
   |<==== Зашифрованный HTTP =====>|
```

```python
import requests

# HTTP (небезопасно)
response = requests.get("http://api.example.com/data")

# HTTPS (безопасно)
response = requests.get("https://api.example.com/data")

# Проверка SSL сертификата (по умолчанию включена)
response = requests.get("https://api.example.com/data", verify=True)

# Отключить проверку (ТОЛЬКО для разработки!)
response = requests.get("https://api.example.com/data", verify=False)
```

## Порты по умолчанию

| Протокол | Порт |
|----------|------|
| HTTP     | 80   |
| HTTPS    | 443  |

```python
# Эквивалентные URL
"http://example.com"       == "http://example.com:80"
"https://example.com"      == "https://example.com:443"
"http://localhost:8000"    # Нестандартный порт указывается явно
```

## Работа с HTTP в Python

### Использование requests

```python
import requests

# GET запрос
response = requests.get("https://api.github.com/users/octocat")
print(response.status_code)  # 200
print(response.headers)      # {'content-type': 'application/json', ...}
print(response.json())       # {'login': 'octocat', ...}

# POST запрос с JSON
response = requests.post(
    "https://api.example.com/users",
    json={"name": "John", "email": "john@example.com"},
    headers={"Authorization": "Bearer token123"}
)

# Таймаут
response = requests.get(
    "https://api.example.com/slow-endpoint",
    timeout=5  # 5 секунд
)

# Сессии (сохранение cookies между запросами)
session = requests.Session()
session.headers.update({"Authorization": "Bearer token"})

response1 = session.get("https://api.example.com/resource1")
response2 = session.get("https://api.example.com/resource2")
```

### Использование httpx (асинхронный)

```python
import httpx
import asyncio

async def fetch_data():
    async with httpx.AsyncClient() as client:
        response = await client.get("https://api.github.com/users/octocat")
        return response.json()

# Синхронный вариант
with httpx.Client() as client:
    response = client.get("https://api.github.com/users/octocat")
```

### Низкоуровневая работа с http.client

```python
import http.client
import json

# Создание соединения
conn = http.client.HTTPSConnection("api.github.com")

# Отправка запроса
conn.request(
    "GET",
    "/users/octocat",
    headers={"User-Agent": "Python/3.x"}
)

# Получение ответа
response = conn.getresponse()
print(response.status)  # 200
data = json.loads(response.read().decode())

conn.close()
```

## TCP/IP и HTTP

HTTP работает поверх TCP/IP:

```
[Приложение]  - HTTP
[Транспорт]   - TCP
[Сеть]        - IP
[Канальный]   - Ethernet/Wi-Fi
```

### Установка TCP соединения (Three-Way Handshake):

```
Клиент                    Сервер
   |                         |
   |-------- SYN ----------->|
   |<------- SYN-ACK --------|
   |-------- ACK ----------->|
   |                         |
   |<====== HTTP ==========>|
```

## Keep-Alive соединения

В HTTP/1.1 по умолчанию используются постоянные соединения:

```http
GET /page1 HTTP/1.1
Host: example.com
Connection: keep-alive

# Ответ
HTTP/1.1 200 OK
Connection: keep-alive
Keep-Alive: timeout=5, max=100

# То же соединение используется для следующего запроса
GET /page2 HTTP/1.1
Host: example.com
```

```python
import requests

# Сессия автоматически использует keep-alive
session = requests.Session()

# Эти запросы используют одно TCP-соединение
for i in range(10):
    response = session.get(f"https://api.example.com/resource/{i}")
```

## Прокси и HTTP

```python
import requests

# HTTP прокси
proxies = {
    "http": "http://proxy.example.com:8080",
    "https": "http://proxy.example.com:8080",
}

response = requests.get(
    "https://api.example.com/data",
    proxies=proxies
)

# SOCKS прокси (требует pip install requests[socks])
proxies = {
    "http": "socks5://127.0.0.1:9050",
    "https": "socks5://127.0.0.1:9050",
}
```

## Best Practices

### 1. Всегда используйте HTTPS

```python
# Плохо
requests.get("http://api.example.com/sensitive-data")

# Хорошо
requests.get("https://api.example.com/sensitive-data")
```

### 2. Устанавливайте таймауты

```python
# Плохо (может зависнуть навсегда)
requests.get("https://api.example.com/slow")

# Хорошо
requests.get("https://api.example.com/slow", timeout=(3.05, 27))
# (connect timeout, read timeout)
```

### 3. Обрабатывайте ошибки

```python
import requests
from requests.exceptions import RequestException, Timeout, HTTPError

try:
    response = requests.get("https://api.example.com/data", timeout=5)
    response.raise_for_status()  # Вызывает исключение при 4xx/5xx
    data = response.json()
except Timeout:
    print("Запрос превысил таймаут")
except HTTPError as e:
    print(f"HTTP ошибка: {e.response.status_code}")
except RequestException as e:
    print(f"Ошибка запроса: {e}")
```

### 4. Используйте сессии для множественных запросов

```python
# Плохо (новое соединение каждый раз)
for url in urls:
    requests.get(url)

# Хорошо (переиспользование соединения)
with requests.Session() as session:
    for url in urls:
        session.get(url)
```

## Типичные ошибки

1. **Игнорирование HTTPS** — передача чувствительных данных без шифрования
2. **Отсутствие таймаутов** — программа может зависнуть
3. **Игнорирование кодов ответа** — не проверяется успешность запроса
4. **Создание нового соединения для каждого запроса** — неэффективно
5. **Хранение секретов в URL** — они попадают в логи

```python
# Плохо
requests.get("https://api.example.com/data?api_key=secret123")

# Хорошо
requests.get(
    "https://api.example.com/data",
    headers={"Authorization": "Bearer secret123"}
)
```

## Инструменты для работы с HTTP

### curl (командная строка)

```bash
# GET запрос
curl https://api.github.com/users/octocat

# POST с JSON
curl -X POST https://api.example.com/users \
  -H "Content-Type: application/json" \
  -d '{"name": "John"}'

# Просмотр заголовков
curl -I https://example.com

# Подробный вывод
curl -v https://example.com
```

### httpie (удобная альтернатива curl)

```bash
# GET
http https://api.github.com/users/octocat

# POST с JSON
http POST https://api.example.com/users name=John email=john@example.com

# Заголовки
http https://api.example.com Authorization:"Bearer token"
```
