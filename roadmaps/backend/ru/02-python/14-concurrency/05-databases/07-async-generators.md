# Асинхронные генераторы

[prev: ./06-transactions.md](./06-transactions.md) | [next: ../06-cpu-bound/readme.md](../06-cpu-bound/readme.md)

---

## Что такое асинхронные генераторы?

**Асинхронные генераторы** — это комбинация генераторов и асинхронного программирования. Они позволяют асинхронно производить значения по мере необходимости, что особенно полезно при работе с большими объёмами данных из базы данных.

### Синтаксис асинхронного генератора

```python
# Обычный генератор
def sync_generator():
    yield 1
    yield 2
    yield 3

# Асинхронный генератор
async def async_generator():
    yield 1
    await asyncio.sleep(0.1)
    yield 2
    await asyncio.sleep(0.1)
    yield 3
```

## Курсоры в asyncpg как асинхронные генераторы

asyncpg предоставляет метод `cursor()` для потоковой обработки больших результатов:

```python
import asyncio
import asyncpg


async def cursor_as_async_generator():
    conn = await asyncpg.connect('postgresql://postgres:password@localhost/mydb')

    try:
        # Курсор работает только внутри транзакции
        async with conn.transaction():
            # Курсор — это асинхронный итератор
            async for record in conn.cursor('SELECT * FROM users'):
                print(f"User: {record['username']}")
                # Обработка записи по одной

    finally:
        await conn.close()


asyncio.run(cursor_as_async_generator())
```

### Настройка предварительной загрузки (prefetch)

```python
import asyncio
import asyncpg


async def cursor_with_prefetch():
    conn = await asyncpg.connect('postgresql://postgres:password@localhost/mydb')

    try:
        async with conn.transaction():
            # prefetch указывает количество записей для предварительной загрузки
            # Оптимальное значение зависит от размера записей и доступной памяти
            async for record in conn.cursor(
                'SELECT * FROM large_table',
                prefetch=1000  # Загружать по 1000 записей за раз
            ):
                await process_record(record)

    finally:
        await conn.close()


async def process_record(record):
    """Обработка одной записи."""
    pass


asyncio.run(cursor_with_prefetch())
```

## Создание собственных асинхронных генераторов для БД

### Генератор для пагинации

```python
import asyncio
import asyncpg
from typing import AsyncGenerator, List


async def paginated_fetch(
    pool: asyncpg.Pool,
    query: str,
    page_size: int = 100
) -> AsyncGenerator[List[asyncpg.Record], None]:
    """
    Асинхронный генератор для постраничного получения данных.

    Yields:
        Списки записей по page_size штук
    """
    offset = 0

    while True:
        async with pool.acquire() as conn:
            paginated_query = f"{query} LIMIT {page_size} OFFSET {offset}"
            rows = await conn.fetch(paginated_query)

            if not rows:
                break

            yield rows
            offset += page_size


async def main():
    pool = await asyncpg.create_pool('postgresql://postgres:password@localhost/mydb')

    try:
        # Использование генератора пагинации
        page_num = 0
        async for page in paginated_fetch(pool, 'SELECT * FROM users ORDER BY id'):
            page_num += 1
            print(f"Страница {page_num}: {len(page)} записей")

            for user in page:
                print(f"  - {user['username']}")

    finally:
        await pool.close()


asyncio.run(main())
```

### Генератор для записей по одной

```python
import asyncio
import asyncpg
from typing import AsyncGenerator


async def fetch_records(
    pool: asyncpg.Pool,
    query: str,
    *args,
    batch_size: int = 500
) -> AsyncGenerator[asyncpg.Record, None]:
    """
    Асинхронный генератор для получения записей по одной.
    Внутри использует batch_size для эффективности.

    Yields:
        Одна запись за итерацию
    """
    offset = 0

    while True:
        async with pool.acquire() as conn:
            rows = await conn.fetch(
                f"{query} LIMIT {batch_size} OFFSET {offset}",
                *args
            )

            if not rows:
                break

            for row in rows:
                yield row

            offset += batch_size


async def main():
    pool = await asyncpg.create_pool('postgresql://postgres:password@localhost/mydb')

    try:
        count = 0
        async for user in fetch_records(pool, 'SELECT * FROM users ORDER BY id'):
            count += 1
            print(f"User #{count}: {user['username']}")

            if count >= 10:  # Прервать после 10 записей
                break

    finally:
        await pool.close()


asyncio.run(main())
```

## Потоковая обработка больших данных

### Экспорт данных в файл

```python
import asyncio
import asyncpg
import json
from typing import AsyncGenerator


async def stream_to_json_file(
    pool: asyncpg.Pool,
    query: str,
    filename: str
):
    """Потоковый экспорт данных в JSON файл."""
    import aiofiles

    async with aiofiles.open(filename, 'w') as f:
        await f.write('[\n')
        first = True

        async with pool.acquire() as conn:
            async with conn.transaction():
                async for record in conn.cursor(query, prefetch=1000):
                    if not first:
                        await f.write(',\n')
                    first = False

                    await f.write(json.dumps(dict(record), default=str, ensure_ascii=False))

        await f.write('\n]')

    print(f"Данные экспортированы в {filename}")


async def main():
    pool = await asyncpg.create_pool('postgresql://postgres:password@localhost/mydb')

    try:
        await stream_to_json_file(
            pool,
            'SELECT * FROM orders ORDER BY created_at',
            'orders_export.json'
        )

    finally:
        await pool.close()


asyncio.run(main())
```

### Обработка с ограничением параллелизма

```python
import asyncio
import asyncpg
from typing import AsyncGenerator, Callable, Any


async def process_with_concurrency(
    generator: AsyncGenerator,
    processor: Callable,
    max_concurrent: int = 10
):
    """
    Обработка записей из генератора с ограничением параллелизма.

    Args:
        generator: Асинхронный генератор записей
        processor: Асинхронная функция обработки
        max_concurrent: Максимальное количество параллельных задач
    """
    semaphore = asyncio.Semaphore(max_concurrent)
    tasks = []

    async def bounded_process(item):
        async with semaphore:
            return await processor(item)

    async for item in generator:
        task = asyncio.create_task(bounded_process(item))
        tasks.append(task)

        # Периодически очищаем завершённые задачи
        if len(tasks) >= max_concurrent * 2:
            done, tasks = await asyncio.wait(
                tasks,
                return_when=asyncio.FIRST_COMPLETED
            )
            tasks = list(tasks)

    # Дожидаемся оставшихся задач
    if tasks:
        await asyncio.gather(*tasks)


async def process_user(user: asyncpg.Record):
    """Обработка одного пользователя."""
    print(f"Processing: {user['username']}")
    await asyncio.sleep(0.1)  # Симуляция работы


async def main():
    pool = await asyncpg.create_pool('postgresql://postgres:password@localhost/mydb')

    try:
        async with pool.acquire() as conn:
            async with conn.transaction():
                generator = conn.cursor('SELECT * FROM users', prefetch=100)
                await process_with_concurrency(generator, process_user, max_concurrent=5)

    finally:
        await pool.close()


asyncio.run(main())
```

## Асинхронные генераторы для ETL-процессов

### Extract-Transform-Load пайплайн

```python
import asyncio
import asyncpg
from typing import AsyncGenerator, Dict, Any
from dataclasses import dataclass
from datetime import datetime


@dataclass
class User:
    id: int
    username: str
    email: str
    created_at: datetime


# EXTRACT: Чтение данных из источника
async def extract_users(
    pool: asyncpg.Pool
) -> AsyncGenerator[asyncpg.Record, None]:
    """Извлечение пользователей из базы данных."""
    async with pool.acquire() as conn:
        async with conn.transaction():
            async for record in conn.cursor(
                'SELECT * FROM users ORDER BY id',
                prefetch=500
            ):
                yield record


# TRANSFORM: Преобразование данных
async def transform_user(
    record: asyncpg.Record
) -> Dict[str, Any]:
    """Преобразование записи в нужный формат."""
    return {
        'user_id': record['id'],
        'full_name': record['username'].title(),
        'email_domain': record['email'].split('@')[1] if '@' in record['email'] else '',
        'registration_year': record['created_at'].year if record['created_at'] else None,
        'processed_at': datetime.now().isoformat()
    }


async def transform_pipeline(
    source: AsyncGenerator[asyncpg.Record, None]
) -> AsyncGenerator[Dict[str, Any], None]:
    """Пайплайн трансформации."""
    async for record in source:
        transformed = await transform_user(record)
        yield transformed


# LOAD: Загрузка данных в целевую базу
async def load_users(
    pool: asyncpg.Pool,
    data_stream: AsyncGenerator[Dict[str, Any], None],
    batch_size: int = 100
):
    """Загрузка данных в целевую таблицу."""
    batch = []

    async for item in data_stream:
        batch.append(item)

        if len(batch) >= batch_size:
            await _insert_batch(pool, batch)
            batch = []

    # Загружаем остаток
    if batch:
        await _insert_batch(pool, batch)


async def _insert_batch(pool: asyncpg.Pool, batch: list):
    """Вставка пакета данных."""
    async with pool.acquire() as conn:
        await conn.executemany('''
            INSERT INTO users_processed (user_id, full_name, email_domain, registration_year, processed_at)
            VALUES ($1, $2, $3, $4, $5)
            ON CONFLICT (user_id) DO UPDATE SET
                full_name = EXCLUDED.full_name,
                processed_at = EXCLUDED.processed_at
        ''', [
            (item['user_id'], item['full_name'], item['email_domain'],
             item['registration_year'], item['processed_at'])
            for item in batch
        ])
    print(f"Loaded batch of {len(batch)} records")


async def run_etl():
    """Запуск полного ETL процесса."""
    source_pool = await asyncpg.create_pool(
        'postgresql://postgres:password@localhost/source_db'
    )
    target_pool = await asyncpg.create_pool(
        'postgresql://postgres:password@localhost/target_db'
    )

    try:
        # Создание пайплайна
        extract = extract_users(source_pool)
        transform = transform_pipeline(extract)

        # Загрузка
        await load_users(target_pool, transform)

        print("ETL процесс завершён успешно")

    finally:
        await source_pool.close()
        await target_pool.close()


asyncio.run(run_etl())
```

## Работа с LISTEN/NOTIFY через генераторы

```python
import asyncio
import asyncpg
from typing import AsyncGenerator


async def listen_notifications(
    dsn: str,
    channel: str
) -> AsyncGenerator[str, None]:
    """
    Асинхронный генератор для получения уведомлений PostgreSQL.

    Yields:
        Payload уведомлений из канала
    """
    conn = await asyncpg.connect(dsn)

    try:
        # Очередь для уведомлений
        queue = asyncio.Queue()

        # Callback для получения уведомлений
        def notification_handler(connection, pid, channel, payload):
            queue.put_nowait(payload)

        await conn.add_listener(channel, notification_handler)
        print(f"Listening on channel: {channel}")

        while True:
            payload = await queue.get()
            yield payload

    finally:
        await conn.remove_listener(channel, notification_handler)
        await conn.close()


async def send_notification(dsn: str, channel: str, message: str):
    """Отправка уведомления."""
    conn = await asyncpg.connect(dsn)
    try:
        await conn.execute(f"NOTIFY {channel}, $1", message)
    finally:
        await conn.close()


async def main():
    dsn = 'postgresql://postgres:password@localhost/mydb'
    channel = 'my_channel'

    # Запуск слушателя в фоне
    async def listener():
        async for notification in listen_notifications(dsn, channel):
            print(f"Received: {notification}")
            if notification == 'stop':
                break

    listener_task = asyncio.create_task(listener())

    # Отправка тестовых уведомлений
    await asyncio.sleep(1)
    await send_notification(dsn, channel, 'Hello!')
    await asyncio.sleep(0.5)
    await send_notification(dsn, channel, 'World!')
    await asyncio.sleep(0.5)
    await send_notification(dsn, channel, 'stop')

    await listener_task


asyncio.run(main())
```

## Агрегация данных через генераторы

```python
import asyncio
import asyncpg
from typing import AsyncGenerator, Dict, Any
from collections import defaultdict


async def aggregate_by_chunks(
    pool: asyncpg.Pool,
    query: str,
    group_key: str,
    agg_field: str,
    chunk_size: int = 1000
) -> AsyncGenerator[Dict[str, Any], None]:
    """
    Агрегация данных чанками для экономии памяти.

    Yields:
        Промежуточные результаты агрегации
    """
    aggregates = defaultdict(lambda: {'count': 0, 'sum': 0, 'min': None, 'max': None})
    processed = 0

    async with pool.acquire() as conn:
        async with conn.transaction():
            async for record in conn.cursor(query, prefetch=chunk_size):
                key = record[group_key]
                value = record[agg_field]

                if value is not None:
                    aggregates[key]['count'] += 1
                    aggregates[key]['sum'] += value

                    if aggregates[key]['min'] is None or value < aggregates[key]['min']:
                        aggregates[key]['min'] = value
                    if aggregates[key]['max'] is None or value > aggregates[key]['max']:
                        aggregates[key]['max'] = value

                processed += 1

                # Yield промежуточные результаты каждые chunk_size записей
                if processed % chunk_size == 0:
                    yield {
                        'processed': processed,
                        'groups': len(aggregates),
                        'snapshot': dict(aggregates)
                    }

    # Финальный результат
    yield {
        'processed': processed,
        'groups': len(aggregates),
        'final': True,
        'results': {
            key: {
                **agg,
                'avg': agg['sum'] / agg['count'] if agg['count'] > 0 else 0
            }
            for key, agg in aggregates.items()
        }
    }


async def main():
    pool = await asyncpg.create_pool('postgresql://postgres:password@localhost/mydb')

    try:
        async for result in aggregate_by_chunks(
            pool,
            'SELECT category_id, price FROM products',
            group_key='category_id',
            agg_field='price'
        ):
            if result.get('final'):
                print("\nФинальные результаты:")
                for category, stats in result['results'].items():
                    print(f"  Категория {category}: "
                          f"count={stats['count']}, "
                          f"avg={stats['avg']:.2f}, "
                          f"min={stats['min']}, "
                          f"max={stats['max']}")
            else:
                print(f"Обработано: {result['processed']}, групп: {result['groups']}")

    finally:
        await pool.close()


asyncio.run(main())
```

## Лучшие практики

1. **Используйте курсоры внутри транзакций** — это требование asyncpg
2. **Настраивайте prefetch** в зависимости от размера записей и доступной памяти
3. **Ограничивайте параллелизм** при обработке для предотвращения перегрузки
4. **Обрабатывайте ошибки** внутри генераторов корректно
5. **Освобождайте ресурсы** — закрывайте соединения в finally блоках

```python
# Пример корректной обработки ошибок в генераторе
async def safe_generator(pool: asyncpg.Pool) -> AsyncGenerator[asyncpg.Record, None]:
    async with pool.acquire() as conn:
        try:
            async with conn.transaction():
                async for record in conn.cursor('SELECT * FROM users'):
                    try:
                        yield record
                    except GeneratorExit:
                        # Генератор был закрыт извне (break, return в цикле for)
                        print("Generator closed")
                        return
        except asyncpg.PostgresError as e:
            print(f"Database error: {e}")
            raise
```

---

[prev: ./06-transactions.md](./06-transactions.md) | [next: ../06-cpu-bound/readme.md](../06-cpu-bound/readme.md)
