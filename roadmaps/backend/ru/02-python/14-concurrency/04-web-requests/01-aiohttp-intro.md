# Введение в aiohttp

[prev: ./readme.md](./readme.md) | [next: ./02-context-managers.md](./02-context-managers.md)

---

## Что такое aiohttp?

**aiohttp** - это асинхронная HTTP-библиотека для Python, построенная на основе asyncio. Она предоставляет как клиентскую, так и серверную функциональность для работы с HTTP-протоколом.

В отличие от синхронной библиотеки `requests`, aiohttp позволяет выполнять множество HTTP-запросов конкурентно, не блокируя выполнение программы. Это особенно полезно при работе с API, веб-скрапинге и других сценариях, где нужно делать много сетевых запросов.

## Установка

```bash
pip install aiohttp
```

Для дополнительных возможностей (ускорение, DNS-кеширование):

```bash
pip install aiohttp[speedups]
```

## Основные компоненты

### ClientSession

`ClientSession` - это основной класс для выполнения HTTP-запросов. Он управляет пулом соединений, cookies и другими параметрами.

```python
import aiohttp
import asyncio

async def main():
    async with aiohttp.ClientSession() as session:
        async with session.get('https://api.github.com') as response:
            print(f"Статус: {response.status}")
            data = await response.json()
            print(data)

asyncio.run(main())
```

### Почему важно использовать одну сессию?

Создание `ClientSession` - это относительно дорогая операция. Сессия:
- Управляет пулом TCP-соединений
- Переиспользует соединения (keep-alive)
- Хранит cookies между запросами
- Кеширует DNS-записи

```python
# ПЛОХО: создание сессии на каждый запрос
async def bad_example():
    for url in urls:
        async with aiohttp.ClientSession() as session:  # Каждый раз новая сессия!
            async with session.get(url) as response:
                pass

# ХОРОШО: одна сессия для всех запросов
async def good_example():
    async with aiohttp.ClientSession() as session:
        for url in urls:
            async with session.get(url) as response:
                pass
```

## HTTP-методы

aiohttp поддерживает все стандартные HTTP-методы:

```python
import aiohttp
import asyncio

async def http_methods_example():
    async with aiohttp.ClientSession() as session:
        # GET запрос
        async with session.get('https://httpbin.org/get') as resp:
            print(await resp.json())

        # POST запрос
        async with session.post('https://httpbin.org/post', json={'key': 'value'}) as resp:
            print(await resp.json())

        # PUT запрос
        async with session.put('https://httpbin.org/put', data={'key': 'value'}) as resp:
            print(await resp.json())

        # DELETE запрос
        async with session.delete('https://httpbin.org/delete') as resp:
            print(await resp.json())

        # PATCH запрос
        async with session.patch('https://httpbin.org/patch', json={'key': 'new_value'}) as resp:
            print(await resp.json())

asyncio.run(http_methods_example())
```

## Работа с параметрами запроса

### Query-параметры

```python
async def query_params_example():
    async with aiohttp.ClientSession() as session:
        # Способ 1: через словарь
        params = {'key1': 'value1', 'key2': 'value2'}
        async with session.get('https://httpbin.org/get', params=params) as resp:
            print(await resp.json())

        # Способ 2: множественные значения для одного ключа
        params = [('key', 'value1'), ('key', 'value2')]
        async with session.get('https://httpbin.org/get', params=params) as resp:
            print(await resp.json())
```

### Заголовки

```python
async def headers_example():
    headers = {
        'User-Agent': 'MyApp/1.0',
        'Authorization': 'Bearer token123',
        'Content-Type': 'application/json'
    }

    async with aiohttp.ClientSession(headers=headers) as session:
        # Заголовки из сессии применяются ко всем запросам
        async with session.get('https://httpbin.org/headers') as resp:
            print(await resp.json())

        # Можно переопределить для конкретного запроса
        async with session.get(
            'https://httpbin.org/headers',
            headers={'X-Custom-Header': 'custom_value'}
        ) as resp:
            print(await resp.json())
```

## Работа с ответом

### Чтение данных

```python
async def response_handling():
    async with aiohttp.ClientSession() as session:
        async with session.get('https://api.github.com') as response:
            # Статус ответа
            print(f"Status: {response.status}")
            print(f"Reason: {response.reason}")

            # Заголовки ответа
            print(f"Content-Type: {response.headers['Content-Type']}")

            # Чтение тела ответа
            text = await response.text()  # Как строка
            json_data = await response.json()  # Как JSON
            binary = await response.read()  # Как байты

            # URL (может отличаться из-за редиректов)
            print(f"Final URL: {response.url}")
```

### Потоковое чтение больших файлов

```python
async def download_file(url: str, filename: str):
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            with open(filename, 'wb') as f:
                # Читаем по частям, не загружая весь файл в память
                async for chunk in response.content.iter_chunked(8192):
                    f.write(chunk)
```

## Таймауты

Важно устанавливать таймауты для защиты от зависших запросов:

```python
import aiohttp
from aiohttp import ClientTimeout

async def timeout_example():
    # Таймаут для всей сессии
    timeout = ClientTimeout(
        total=30,      # Общий таймаут на весь запрос
        connect=10,    # Таймаут на установку соединения
        sock_read=10,  # Таймаут на чтение данных
        sock_connect=10  # Таймаут на подключение сокета
    )

    async with aiohttp.ClientSession(timeout=timeout) as session:
        try:
            async with session.get('https://httpbin.org/delay/5') as resp:
                print(await resp.text())
        except asyncio.TimeoutError:
            print("Запрос превысил таймаут!")
```

## Обработка ошибок

```python
import aiohttp
from aiohttp import ClientError, ClientResponseError

async def error_handling():
    async with aiohttp.ClientSession() as session:
        try:
            async with session.get('https://httpbin.org/status/404') as response:
                # Проверка успешности запроса
                response.raise_for_status()
                data = await response.json()
        except ClientResponseError as e:
            print(f"HTTP ошибка: {e.status} - {e.message}")
        except ClientError as e:
            print(f"Ошибка клиента: {e}")
        except asyncio.TimeoutError:
            print("Превышено время ожидания")
```

## Сравнение с requests

| Характеристика | requests | aiohttp |
|---------------|----------|---------|
| Тип | Синхронный | Асинхронный |
| Конкурентность | Нет (нужны threads) | Да (через asyncio) |
| Производительность | Ниже для множества запросов | Выше для множества запросов |
| Простота использования | Проще | Требует понимания async/await |
| Память | Выше при многих потоках | Ниже |

### Пример: синхронный vs асинхронный подход

```python
import time
import requests
import aiohttp
import asyncio

urls = [f'https://httpbin.org/delay/1' for _ in range(5)]

# Синхронный подход с requests
def sync_requests():
    start = time.time()
    for url in urls:
        requests.get(url)
    print(f"Синхронно: {time.time() - start:.2f} сек")  # ~5 секунд

# Асинхронный подход с aiohttp
async def async_requests():
    start = time.time()
    async with aiohttp.ClientSession() as session:
        tasks = [session.get(url) for url in urls]
        responses = await asyncio.gather(*tasks)
        for resp in responses:
            await resp.read()
    print(f"Асинхронно: {time.time() - start:.2f} сек")  # ~1 секунда

sync_requests()
asyncio.run(async_requests())
```

## Лучшие практики

1. **Используйте одну сессию** для всех запросов в рамках одной задачи
2. **Всегда устанавливайте таймауты** для защиты от зависаний
3. **Используйте контекстные менеджеры** (`async with`) для правильного освобождения ресурсов
4. **Обрабатывайте ошибки** - сеть ненадежна
5. **Ограничивайте количество одновременных запросов** с помощью семафоров (см. следующие темы)

## Частые ошибки

```python
# ОШИБКА: Не используется await для чтения ответа
async with session.get(url) as response:
    data = response.json()  # Забыли await!

# ПРАВИЛЬНО:
async with session.get(url) as response:
    data = await response.json()

# ОШИБКА: Чтение ответа после закрытия контекста
async with session.get(url) as response:
    pass
data = await response.json()  # Ошибка! response уже закрыт

# ПРАВИЛЬНО:
async with session.get(url) as response:
    data = await response.json()
# Используем data после закрытия контекста
```

---

[prev: ./readme.md](./readme.md) | [next: ./02-context-managers.md](./02-context-managers.md)
