# Асинхронные контекстные менеджеры

[prev: ./01-aiohttp-intro.md](./01-aiohttp-intro.md) | [next: ./03-concurrent-tasks.md](./03-concurrent-tasks.md)

---

## Что такое контекстный менеджер?

Контекстный менеджер - это объект, который определяет контекст выполнения для блока кода. Он гарантирует, что определенные действия будут выполнены при входе в контекст и выходе из него, независимо от того, произошла ли ошибка.

В синхронном Python мы используем `with`:

```python
with open('file.txt', 'r') as f:
    content = f.read()
# Файл автоматически закрыт здесь
```

## Асинхронные контекстные менеджеры

В асинхронном коде используется `async with`. Это позволяет выполнять асинхронные операции при входе и выходе из контекста.

### Протокол асинхронного контекстного менеджера

Асинхронный контекстный менеджер реализует два магических метода:
- `__aenter__()` - асинхронный вход в контекст
- `__aexit__()` - асинхронный выход из контекста

```python
class AsyncContextManager:
    async def __aenter__(self):
        print("Вход в контекст")
        await asyncio.sleep(0.1)  # Асинхронная инициализация
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        print("Выход из контекста")
        await asyncio.sleep(0.1)  # Асинхронная очистка
        return False  # Не подавляем исключения

async def main():
    async with AsyncContextManager() as manager:
        print("Внутри контекста")
```

## Контекстные менеджеры в aiohttp

### ClientSession как контекстный менеджер

`ClientSession` - это асинхронный контекстный менеджер. При выходе из контекста он закрывает все соединения и освобождает ресурсы.

```python
import aiohttp
import asyncio

async def main():
    # ClientSession автоматически закроется при выходе из блока
    async with aiohttp.ClientSession() as session:
        async with session.get('https://api.github.com') as response:
            data = await response.json()
            print(data)
    # Здесь сессия уже закрыта

asyncio.run(main())
```

### Ручное управление сессией

Иногда нужно управлять сессией вручную:

```python
async def manual_session_management():
    session = aiohttp.ClientSession()
    try:
        async with session.get('https://api.github.com') as response:
            data = await response.json()
    finally:
        await session.close()  # Обязательно закрываем!
```

**Важно**: Если не закрыть сессию, будет утечка ресурсов и предупреждение:
```
Unclosed client session
client_session: <aiohttp.client.ClientSession object at 0x...>
```

### Response как контекстный менеджер

Каждый ответ `Response` также является контекстным менеджером:

```python
async def response_context():
    async with aiohttp.ClientSession() as session:
        # Response - контекстный менеджер
        async with session.get('https://httpbin.org/get') as response:
            # Тело ответа доступно только внутри этого блока
            data = await response.json()
            print(data)
        # Здесь response уже освобожден

        # ОШИБКА: нельзя читать response после выхода из контекста
        # print(await response.text())  # Вызовет ошибку!
```

## Зачем нужен контекстный менеджер для Response?

Response держит соединение открытым, пока вы читаете данные. Использование контекстного менеджера гарантирует:

1. **Освобождение соединения** - соединение возвращается в пул
2. **Правильное управление памятью** - буферы освобождаются
3. **Корректную обработку ошибок** - ресурсы очищаются даже при ошибках

```python
async def proper_resource_handling():
    async with aiohttp.ClientSession() as session:
        urls = ['https://httpbin.org/get' for _ in range(100)]

        for url in urls:
            async with session.get(url) as response:
                # Соединение используется
                data = await response.json()
            # Соединение возвращено в пул
            # Можно переиспользовать для следующего запроса
```

## Вложенные контекстные менеджеры

Часто используется вложение контекстных менеджеров:

```python
async def nested_contexts():
    # Уровень 1: Сессия
    async with aiohttp.ClientSession() as session:
        # Уровень 2: Первый запрос
        async with session.get('https://api.github.com/users/octocat') as resp1:
            user = await resp1.json()

        # Уровень 2: Второй запрос
        async with session.get('https://api.github.com/users/torvalds') as resp2:
            user2 = await resp2.json()

        print(user['login'], user2['login'])
```

## Создание собственного асинхронного контекстного менеджера

### Способ 1: Класс с __aenter__ и __aexit__

```python
import aiohttp
import asyncio

class ApiClient:
    def __init__(self, base_url: str):
        self.base_url = base_url
        self.session = None

    async def __aenter__(self):
        self.session = aiohttp.ClientSession()
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        if self.session:
            await self.session.close()
        return False

    async def get(self, endpoint: str):
        async with self.session.get(f"{self.base_url}{endpoint}") as response:
            return await response.json()

async def main():
    async with ApiClient('https://api.github.com') as client:
        user = await client.get('/users/octocat')
        print(user['name'])

asyncio.run(main())
```

### Способ 2: Декоратор @asynccontextmanager

```python
from contextlib import asynccontextmanager
import aiohttp

@asynccontextmanager
async def create_session():
    session = aiohttp.ClientSession()
    try:
        yield session
    finally:
        await session.close()

async def main():
    async with create_session() as session:
        async with session.get('https://api.github.com') as response:
            data = await response.json()
            print(data)
```

## Практический пример: HTTP-клиент с повторными попытками

```python
import aiohttp
import asyncio
from contextlib import asynccontextmanager
from typing import Optional

class RetryableClient:
    def __init__(self, max_retries: int = 3, timeout: float = 30.0):
        self.max_retries = max_retries
        self.timeout = aiohttp.ClientTimeout(total=timeout)
        self.session: Optional[aiohttp.ClientSession] = None

    async def __aenter__(self):
        self.session = aiohttp.ClientSession(timeout=self.timeout)
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        if self.session:
            await self.session.close()

    async def get(self, url: str):
        last_exception = None

        for attempt in range(1, self.max_retries + 1):
            try:
                async with self.session.get(url) as response:
                    response.raise_for_status()
                    return await response.json()
            except aiohttp.ClientError as e:
                last_exception = e
                print(f"Попытка {attempt}/{self.max_retries} не удалась: {e}")
                if attempt < self.max_retries:
                    await asyncio.sleep(2 ** attempt)  # Экспоненциальная задержка

        raise last_exception

async def main():
    async with RetryableClient(max_retries=3) as client:
        try:
            data = await client.get('https://api.github.com/users/octocat')
            print(data)
        except aiohttp.ClientError as e:
            print(f"Все попытки исчерпаны: {e}")

asyncio.run(main())
```

## Контекстные менеджеры для ограничения ресурсов

### Семафор как контекстный менеджер

```python
import aiohttp
import asyncio

async def limited_requests():
    semaphore = asyncio.Semaphore(5)  # Максимум 5 одновременных запросов

    async def fetch_with_limit(session, url):
        async with semaphore:  # Семафор - контекстный менеджер
            async with session.get(url) as response:
                return await response.json()

    async with aiohttp.ClientSession() as session:
        urls = [f'https://httpbin.org/delay/1' for _ in range(20)]
        tasks = [fetch_with_limit(session, url) for url in urls]
        results = await asyncio.gather(*tasks)
        print(f"Получено {len(results)} результатов")

asyncio.run(limited_requests())
```

## Комбинирование контекстных менеджеров

Python 3.10+ позволяет группировать контекстные менеджеры:

```python
# Python 3.10+
async def combined_managers():
    async with (
        aiohttp.ClientSession() as session,
        asyncio.timeout(30)  # Таймаут для всего блока
    ):
        async with session.get('https://api.github.com') as response:
            return await response.json()
```

Для более ранних версий:

```python
# Python 3.7+
async def combined_managers_old():
    async with aiohttp.ClientSession() as session:
        try:
            async with asyncio.timeout(30):
                async with session.get('https://api.github.com') as response:
                    return await response.json()
        except asyncio.TimeoutError:
            print("Превышен таймаут!")
```

## Лучшие практики

1. **Всегда используйте `async with`** для сессий и ответов
2. **Не храните ссылки на response** вне контекста
3. **Обрабатывайте исключения** в `__aexit__` аккуратно
4. **Используйте @asynccontextmanager** для простых случаев
5. **Комбинируйте с семафорами** для контроля нагрузки

## Частые ошибки

```python
# ОШИБКА: Забыли async with для сессии
async def wrong():
    session = aiohttp.ClientSession()
    response = await session.get('https://api.github.com')
    # Утечка ресурсов! session и response не закрыты

# ПРАВИЛЬНО:
async def correct():
    async with aiohttp.ClientSession() as session:
        async with session.get('https://api.github.com') as response:
            data = await response.json()

# ОШИБКА: Чтение response после выхода из контекста
async def wrong2():
    async with aiohttp.ClientSession() as session:
        response = await session.get('https://api.github.com')
    # response вне контекста!
    data = await response.json()  # Ошибка!

# ПРАВИЛЬНО:
async def correct2():
    async with aiohttp.ClientSession() as session:
        async with session.get('https://api.github.com') as response:
            data = await response.json()
    # Используем data, а не response
    return data
```

---

[prev: ./01-aiohttp-intro.md](./01-aiohttp-intro.md) | [next: ./03-concurrent-tasks.md](./03-concurrent-tasks.md)
