# Concurrency (Asyncio)

[prev: 13-testing](../13-testing.md) | [next: 15-frameworks](../15-frameworks/01-fastapi.md)

---

По книге "Python Concurrency with Asyncio" — Matthew Fowler

## Содержание

### [01. Первое знакомство с asyncio](./01-introduction/readme.md)
Что такое asyncio, конкурентность vs параллелизм, GIL, однопоточная конкурентность, цикл событий.

### [02. Основы asyncio](./02-basics/readme.md)
Сопрограммы, задачи, futures, тайм-ауты, отладка.

### [03. Первое приложение asyncio](./03-first-app/readme.md)
Сокеты, selectors, эхо-сервер, корректная остановка.

### [04. Конкурентные веб-запросы](./04-web-requests/readme.md)
aiohttp, gather, as_completed, wait.

### [05. Неблокирующие драйверы баз данных](./05-databases/readme.md)
asyncpg, пулы подключений, транзакции, асинхронные генераторы.

### [06. Счетные задачи (CPU-bound)](./06-cpu-bound/readme.md)
multiprocessing, пулы процессов, MapReduce, разделяемые данные.

### [07. Решение проблем блокирования с потоками](./07-threading/readme.md)
threading, блокировки, взаимоблокировки, циклы событий в потоках.

### [08. Потоки данных](./08-streams/readme.md)
Транспорты, протоколы, StreamReader/Writer, серверы, чат.

### [09. Веб-приложения](./09-web-apps/readme.md)
REST API, ASGI vs WSGI, Starlette, Django async.

### [10. Микросервисы](./10-microservices/readme.md)
Backend-for-frontend, реализация сервисов.

### [11. Синхронизация](./11-synchronization/readme.md)
Race conditions, блокировки, семафоры, события, условия.

### [12. Асинхронные очереди](./12-queues/readme.md)
Queue, PriorityQueue, LifoQueue.

### [13. Управление подпроцессами](./13-subprocesses/readme.md)
Создание подпроцессов, взаимодействие.

### [14. Продвинутое использование asyncio](./14-advanced/readme.md)
Awaitable API, контекстные переменные, кастомные циклы событий.

---

[prev: 13-testing](../13-testing.md) | [next: 15-frameworks](../15-frameworks/01-fastapi.md)
