# Batch Processing

## Введение

**Batch Processing (Пакетная обработка)** — это паттерн обработки данных, при котором данные накапливаются и обрабатываются группами (пакетами) вместо обработки каждого элемента по отдельности. Этот подход особенно эффективен для больших объёмов данных, где индивидуальная обработка была бы слишком затратной.

## Архитектурная диаграмма

```
┌────────────────────────────────────────────────────────────────────────────┐
│                         Batch Processing Architecture                       │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Data Sources                    Batch Processing                Outputs  │
│   ┌──────────┐                                                              │
│   │   API    │──┐                                                           │
│   └──────────┘  │                                                           │
│                 │    ┌─────────────────────────────────────┐               │
│   ┌──────────┐  │    │                                     │    ┌────────┐ │
│   │   DB     │──┼───>│  ┌─────────┐  ┌─────────┐         │───>│   DB   │ │
│   └──────────┘  │    │  │ Extract │─>│Transform│─>        │    └────────┘ │
│                 │    │  └─────────┘  └─────────┘          │               │
│   ┌──────────┐  │    │                    │               │    ┌────────┐ │
│   │  Files   │──┤    │              ┌─────▼─────┐         │───>│  File  │ │
│   └──────────┘  │    │              │   Load    │         │    └────────┘ │
│                 │    │              └───────────┘         │               │
│   ┌──────────┐  │    │                                     │    ┌────────┐ │
│   │  Queue   │──┘    │      ETL Pipeline                   │───>│  API   │ │
│   └──────────┘       └─────────────────────────────────────┘    └────────┘ │
│                                                                             │
│   Накопление          Обработка пакетами               Результаты          │
│   данных              (scheduled/triggered)                                │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

## Real-time vs Batch Processing

```
┌────────────────────────────────────────────────────────────────────────┐
│                    Real-time vs Batch Processing                        │
├────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   REAL-TIME:                                                            │
│   ─────────                                                             │
│   Event ──> Process ──> Result    (мгновенно)                          │
│   Event ──> Process ──> Result    (мгновенно)                          │
│   Event ──> Process ──> Result    (мгновенно)                          │
│                                                                         │
│   BATCH:                                                                │
│   ──────                                                                │
│   Event ──┐                                                             │
│   Event ──┼──> [Batch] ──> Process ──> Results  (по расписанию)        │
│   Event ──┘                                                             │
│                                                                         │
│   MICRO-BATCH:                                                          │
│   ───────────                                                           │
│   Events ──> [Mini Batch] ──> Process ──> Results  (каждые N секунд)   │
│                                                                         │
└────────────────────────────────────────────────────────────────────────┘
```

### Сравнительная таблица

| Критерий | Real-time | Batch | Micro-batch |
|----------|-----------|-------|-------------|
| Latency | Миллисекунды | Часы/Дни | Секунды/Минуты |
| Throughput | Низкий | Очень высокий | Высокий |
| Complexity | Высокая | Низкая | Средняя |
| Resource Usage | Постоянный | Пиковый | Умеренный |
| Use Cases | Alerts, Fraud | Reports, ETL | Analytics |

---

## Паттерны Batch Processing

### 1. ETL (Extract, Transform, Load)

```python
from dataclasses import dataclass
from typing import Iterator, List, Any
import pandas as pd
from datetime import datetime

@dataclass
class BatchJob:
    job_id: str
    source: str
    destination: str
    batch_size: int
    started_at: datetime = None
    completed_at: datetime = None
    records_processed: int = 0
    status: str = 'pending'

class ETLPipeline:
    """
    ETL Pipeline для пакетной обработки данных.
    """

    def __init__(self, batch_size: int = 1000):
        self.batch_size = batch_size

    # EXTRACT
    async def extract(self, source: str) -> Iterator[dict]:
        """Извлекает данные из источника пакетами."""
        offset = 0

        while True:
            # Читаем пакет данных
            batch = await self.fetch_batch(source, offset, self.batch_size)

            if not batch:
                break

            for record in batch:
                yield record

            offset += self.batch_size

    async def fetch_batch(self, source: str, offset: int, limit: int) -> List[dict]:
        """Получает пакет данных из источника."""
        # Пример: чтение из базы данных
        query = f"""
            SELECT * FROM {source}
            ORDER BY id
            OFFSET {offset} LIMIT {limit}
        """
        return await self.db.fetch_all(query)

    # TRANSFORM
    def transform(self, records: Iterator[dict]) -> Iterator[dict]:
        """Трансформирует данные."""
        for record in records:
            transformed = self.apply_transformations(record)
            if transformed:  # Фильтрация
                yield transformed

    def apply_transformations(self, record: dict) -> dict:
        """Применяет трансформации к записи."""
        # Нормализация
        record['email'] = record.get('email', '').lower().strip()

        # Вычисляемые поля
        record['full_name'] = f"{record.get('first_name', '')} {record.get('last_name', '')}"

        # Валидация
        if not self.validate(record):
            return None

        return record

    def validate(self, record: dict) -> bool:
        """Валидирует запись."""
        required_fields = ['id', 'email']
        return all(record.get(field) for field in required_fields)

    # LOAD
    async def load(self, records: Iterator[dict], destination: str):
        """Загружает данные в целевую систему пакетами."""
        batch = []

        for record in records:
            batch.append(record)

            if len(batch) >= self.batch_size:
                await self.write_batch(batch, destination)
                batch = []

        # Оставшиеся записи
        if batch:
            await self.write_batch(batch, destination)

    async def write_batch(self, batch: List[dict], destination: str):
        """Записывает пакет в целевую систему."""
        # Bulk insert для эффективности
        query = f"""
            INSERT INTO {destination} (id, email, full_name)
            VALUES ($1, $2, $3)
            ON CONFLICT (id) DO UPDATE SET
                email = EXCLUDED.email,
                full_name = EXCLUDED.full_name
        """
        await self.db.executemany(query, [
            (r['id'], r['email'], r['full_name']) for r in batch
        ])

    # RUN PIPELINE
    async def run(self, source: str, destination: str) -> BatchJob:
        """Запускает ETL pipeline."""
        job = BatchJob(
            job_id=str(uuid.uuid4()),
            source=source,
            destination=destination,
            batch_size=self.batch_size,
            started_at=datetime.utcnow(),
            status='running'
        )

        try:
            # Extract -> Transform -> Load
            raw_data = self.extract(source)
            transformed_data = self.transform(raw_data)
            await self.load(transformed_data, destination)

            job.status = 'completed'
            job.completed_at = datetime.utcnow()

        except Exception as e:
            job.status = 'failed'
            job.error = str(e)

        return job

# Использование
pipeline = ETLPipeline(batch_size=1000)
job = await pipeline.run('users_raw', 'users_processed')
print(f'Job {job.job_id}: {job.status}')
```

### 2. Batch API Requests

```python
import asyncio
import httpx
from typing import List, Dict
from dataclasses import dataclass

@dataclass
class BatchRequest:
    id: str
    method: str
    url: str
    body: dict = None

@dataclass
class BatchResponse:
    id: str
    status_code: int
    body: dict

class BatchAPIClient:
    """
    Клиент для пакетной отправки API запросов.
    """

    def __init__(self, base_url: str, batch_size: int = 50, concurrency: int = 10):
        self.base_url = base_url
        self.batch_size = batch_size
        self.concurrency = concurrency
        self.semaphore = asyncio.Semaphore(concurrency)

    async def execute_batch(self, requests: List[BatchRequest]) -> List[BatchResponse]:
        """Выполняет пакет запросов с контролем параллелизма."""
        async with httpx.AsyncClient(base_url=self.base_url) as client:
            tasks = [
                self._execute_with_semaphore(client, req)
                for req in requests
            ]
            return await asyncio.gather(*tasks, return_exceptions=True)

    async def _execute_with_semaphore(
        self,
        client: httpx.AsyncClient,
        request: BatchRequest
    ) -> BatchResponse:
        """Выполняет запрос с ограничением параллелизма."""
        async with self.semaphore:
            try:
                response = await client.request(
                    method=request.method,
                    url=request.url,
                    json=request.body,
                    timeout=30.0
                )
                return BatchResponse(
                    id=request.id,
                    status_code=response.status_code,
                    body=response.json()
                )
            except Exception as e:
                return BatchResponse(
                    id=request.id,
                    status_code=500,
                    body={'error': str(e)}
                )

    async def process_in_batches(
        self,
        all_requests: List[BatchRequest]
    ) -> List[BatchResponse]:
        """Обрабатывает все запросы пакетами."""
        all_responses = []

        for i in range(0, len(all_requests), self.batch_size):
            batch = all_requests[i:i + self.batch_size]
            responses = await self.execute_batch(batch)
            all_responses.extend(responses)

            # Пауза между пакетами для rate limiting
            if i + self.batch_size < len(all_requests):
                await asyncio.sleep(1)

        return all_responses

# Использование
async def sync_users_to_external_service(users: List[dict]):
    client = BatchAPIClient(
        base_url='https://api.external.com',
        batch_size=50,
        concurrency=10
    )

    requests = [
        BatchRequest(
            id=str(user['id']),
            method='PUT',
            url=f'/users/{user["id"]}',
            body=user
        )
        for user in users
    ]

    responses = await client.process_in_batches(requests)

    successful = sum(1 for r in responses if r.status_code == 200)
    print(f'Synced {successful}/{len(users)} users')
```

### 3. Database Batch Operations

```python
from typing import List, AsyncIterator
import asyncpg

class DatabaseBatchProcessor:
    """
    Пакетная обработка операций с базой данных.
    """

    def __init__(self, dsn: str, batch_size: int = 1000):
        self.dsn = dsn
        self.batch_size = batch_size
        self.pool = None

    async def connect(self):
        self.pool = await asyncpg.create_pool(self.dsn, min_size=5, max_size=20)

    async def close(self):
        await self.pool.close()

    # Batch Insert
    async def batch_insert(self, table: str, records: List[dict]):
        """Пакетная вставка записей."""
        if not records:
            return

        columns = records[0].keys()
        placeholders = ', '.join(f'${i+1}' for i in range(len(columns)))
        query = f"""
            INSERT INTO {table} ({', '.join(columns)})
            VALUES ({placeholders})
        """

        async with self.pool.acquire() as conn:
            # Используем транзакцию для атомарности
            async with conn.transaction():
                await conn.executemany(
                    query,
                    [tuple(r[col] for col in columns) for r in records]
                )

    # Batch Update
    async def batch_update(
        self,
        table: str,
        records: List[dict],
        key_column: str = 'id'
    ):
        """Пакетное обновление записей."""
        if not records:
            return

        # Создаём временную таблицу с обновлениями
        columns = records[0].keys()

        async with self.pool.acquire() as conn:
            async with conn.transaction():
                # Создаём временную таблицу
                temp_table = f'temp_{table}_{uuid.uuid4().hex[:8]}'
                await conn.execute(f"""
                    CREATE TEMP TABLE {temp_table} (
                        LIKE {table} INCLUDING ALL
                    )
                """)

                # Вставляем данные во временную таблицу
                placeholders = ', '.join(f'${i+1}' for i in range(len(columns)))
                await conn.executemany(
                    f"INSERT INTO {temp_table} ({', '.join(columns)}) VALUES ({placeholders})",
                    [tuple(r[col] for col in columns) for r in records]
                )

                # Обновляем основную таблицу
                set_clause = ', '.join(
                    f'{col} = {temp_table}.{col}'
                    for col in columns if col != key_column
                )
                await conn.execute(f"""
                    UPDATE {table}
                    SET {set_clause}
                    FROM {temp_table}
                    WHERE {table}.{key_column} = {temp_table}.{key_column}
                """)

    # Batch Delete
    async def batch_delete(self, table: str, ids: List[int]):
        """Пакетное удаление записей."""
        if not ids:
            return

        async with self.pool.acquire() as conn:
            # Удаляем пакетами для предотвращения блокировок
            for i in range(0, len(ids), self.batch_size):
                batch_ids = ids[i:i + self.batch_size]
                await conn.execute(
                    f"DELETE FROM {table} WHERE id = ANY($1)",
                    batch_ids
                )

    # Streaming read with batches
    async def read_in_batches(
        self,
        query: str,
        params: tuple = None
    ) -> AsyncIterator[List[dict]]:
        """Читает данные пакетами с использованием курсора."""
        async with self.pool.acquire() as conn:
            async with conn.transaction():
                # Используем server-side cursor для больших данных
                async for batch in conn.cursor(query, *params or []).fetch(self.batch_size):
                    yield [dict(record) for record in batch]

# Использование
async def migrate_data():
    processor = DatabaseBatchProcessor('postgresql://localhost/db', batch_size=1000)
    await processor.connect()

    try:
        # Читаем и обрабатываем пакетами
        async for batch in processor.read_in_batches(
            "SELECT * FROM old_users WHERE migrated = false"
        ):
            # Трансформируем
            transformed = [transform_user(u) for u in batch]

            # Записываем
            await processor.batch_insert('new_users', transformed)

            # Помечаем как мигрированные
            await processor.batch_update(
                'old_users',
                [{'id': u['id'], 'migrated': True} for u in batch]
            )

    finally:
        await processor.close()
```

### 4. File-based Batch Processing

```python
import csv
import json
from pathlib import Path
from typing import Iterator, List
import gzip

class FileBatchProcessor:
    """
    Пакетная обработка файлов.
    """

    def __init__(self, batch_size: int = 10000):
        self.batch_size = batch_size

    def read_csv_in_batches(self, file_path: str) -> Iterator[List[dict]]:
        """Читает CSV файл пакетами."""
        batch = []

        # Поддержка gzip
        opener = gzip.open if file_path.endswith('.gz') else open

        with opener(file_path, 'rt', encoding='utf-8') as f:
            reader = csv.DictReader(f)

            for row in reader:
                batch.append(row)

                if len(batch) >= self.batch_size:
                    yield batch
                    batch = []

            if batch:
                yield batch

    def read_jsonl_in_batches(self, file_path: str) -> Iterator[List[dict]]:
        """Читает JSON Lines файл пакетами."""
        batch = []

        opener = gzip.open if file_path.endswith('.gz') else open

        with opener(file_path, 'rt', encoding='utf-8') as f:
            for line in f:
                if line.strip():
                    batch.append(json.loads(line))

                    if len(batch) >= self.batch_size:
                        yield batch
                        batch = []

            if batch:
                yield batch

    def write_csv_batches(
        self,
        file_path: str,
        batches: Iterator[List[dict]],
        fieldnames: List[str] = None
    ):
        """Записывает пакеты в CSV файл."""
        first_batch = True

        opener = gzip.open if file_path.endswith('.gz') else open

        with opener(file_path, 'wt', encoding='utf-8', newline='') as f:
            writer = None

            for batch in batches:
                if first_batch:
                    fieldnames = fieldnames or list(batch[0].keys())
                    writer = csv.DictWriter(f, fieldnames=fieldnames)
                    writer.writeheader()
                    first_batch = False

                writer.writerows(batch)

    def process_large_file(
        self,
        input_path: str,
        output_path: str,
        transform_func
    ):
        """Обрабатывает большой файл пакетами."""
        def process_batches():
            for batch in self.read_csv_in_batches(input_path):
                processed = [transform_func(record) for record in batch]
                yield [r for r in processed if r is not None]

        self.write_csv_batches(output_path, process_batches())

# Использование
def transform_record(record: dict) -> dict:
    """Трансформация записи."""
    return {
        'id': record['id'],
        'email': record['email'].lower(),
        'created_date': record['created_at'][:10]
    }

processor = FileBatchProcessor(batch_size=10000)
processor.process_large_file(
    'input_data.csv.gz',
    'output_data.csv.gz',
    transform_record
)
```

---

## Scheduled Batch Jobs

### Использование Celery Beat

```python
from celery import Celery
from celery.schedules import crontab
from datetime import timedelta

app = Celery('batch_tasks', broker='redis://localhost:6379/0')

# Конфигурация расписания
app.conf.beat_schedule = {
    # Каждый час
    'hourly-sync': {
        'task': 'tasks.sync_users',
        'schedule': crontab(minute=0),
    },
    # Каждую ночь в 2:00
    'nightly-report': {
        'task': 'tasks.generate_daily_report',
        'schedule': crontab(hour=2, minute=0),
    },
    # Каждые 5 минут
    'frequent-check': {
        'task': 'tasks.check_pending_orders',
        'schedule': timedelta(minutes=5),
    },
    # Каждую неделю в воскресенье
    'weekly-cleanup': {
        'task': 'tasks.cleanup_old_data',
        'schedule': crontab(hour=3, minute=0, day_of_week='sunday'),
    },
}

# Задачи
@app.task(bind=True, max_retries=3)
def sync_users(self):
    """Синхронизация пользователей."""
    try:
        pipeline = ETLPipeline(batch_size=1000)
        job = pipeline.run('external_users', 'users')
        return {'job_id': job.job_id, 'status': job.status}
    except Exception as e:
        self.retry(exc=e, countdown=60 * (self.request.retries + 1))

@app.task
def generate_daily_report():
    """Генерация ежедневного отчёта."""
    report = ReportGenerator().generate_daily()
    ReportStorage().save(report)
    NotificationService().send_report(report)

@app.task
def cleanup_old_data():
    """Очистка старых данных."""
    cutoff_date = datetime.utcnow() - timedelta(days=90)
    deleted = DataCleaner().delete_before(cutoff_date)
    return {'deleted_records': deleted}
```

### APScheduler для Python приложений

```python
from apscheduler.schedulers.asyncio import AsyncIOScheduler
from apscheduler.triggers.cron import CronTrigger
from apscheduler.triggers.interval import IntervalTrigger
import asyncio

class BatchScheduler:
    def __init__(self):
        self.scheduler = AsyncIOScheduler()

    def add_jobs(self):
        # Каждые 5 минут
        self.scheduler.add_job(
            self.process_pending_items,
            IntervalTrigger(minutes=5),
            id='process_pending',
            replace_existing=True
        )

        # Каждый день в 3:00
        self.scheduler.add_job(
            self.daily_aggregation,
            CronTrigger(hour=3, minute=0),
            id='daily_aggregation',
            replace_existing=True
        )

        # В начале каждого месяца
        self.scheduler.add_job(
            self.monthly_report,
            CronTrigger(day=1, hour=0, minute=0),
            id='monthly_report',
            replace_existing=True
        )

    async def process_pending_items(self):
        """Обрабатывает накопившиеся элементы."""
        processor = BatchProcessor()
        await processor.process_all_pending()

    async def daily_aggregation(self):
        """Ежедневная агрегация данных."""
        aggregator = DataAggregator()
        await aggregator.aggregate_yesterday()

    async def monthly_report(self):
        """Ежемесячный отчёт."""
        reporter = MonthlyReporter()
        await reporter.generate_and_send()

    def start(self):
        self.add_jobs()
        self.scheduler.start()

# Использование
scheduler = BatchScheduler()
scheduler.start()
asyncio.get_event_loop().run_forever()
```

---

## Batch Processing в API

### Batch Endpoint

```python
from fastapi import FastAPI, HTTPException, BackgroundTasks
from pydantic import BaseModel
from typing import List, Optional
import uuid

app = FastAPI()

class BatchItem(BaseModel):
    id: str
    method: str  # create, update, delete
    data: dict

class BatchRequest(BaseModel):
    items: List[BatchItem]

class BatchItemResult(BaseModel):
    id: str
    success: bool
    error: Optional[str] = None
    data: Optional[dict] = None

class BatchResponse(BaseModel):
    batch_id: str
    results: List[BatchItemResult]
    successful: int
    failed: int

@app.post('/api/v1/users/batch', response_model=BatchResponse)
async def batch_users(request: BatchRequest):
    """
    Batch endpoint для операций с пользователями.

    Пример запроса:
    {
        "items": [
            {"id": "1", "method": "create", "data": {"name": "John", "email": "john@example.com"}},
            {"id": "2", "method": "update", "data": {"id": "123", "name": "Jane"}},
            {"id": "3", "method": "delete", "data": {"id": "456"}}
        ]
    }
    """
    if len(request.items) > 100:
        raise HTTPException(400, 'Maximum 100 items per batch')

    results = []

    for item in request.items:
        try:
            if item.method == 'create':
                result = await create_user(item.data)
            elif item.method == 'update':
                result = await update_user(item.data)
            elif item.method == 'delete':
                result = await delete_user(item.data['id'])
            else:
                raise ValueError(f'Unknown method: {item.method}')

            results.append(BatchItemResult(
                id=item.id,
                success=True,
                data=result
            ))

        except Exception as e:
            results.append(BatchItemResult(
                id=item.id,
                success=False,
                error=str(e)
            ))

    successful = sum(1 for r in results if r.success)

    return BatchResponse(
        batch_id=str(uuid.uuid4()),
        results=results,
        successful=successful,
        failed=len(results) - successful
    )

# Async batch processing with job tracking
class BatchJob(BaseModel):
    job_id: str
    status: str
    progress: int
    total: int
    created_at: datetime
    completed_at: Optional[datetime] = None
    results_url: Optional[str] = None

batch_jobs = {}  # In production: Redis/DB

@app.post('/api/v1/batch/async')
async def create_async_batch(
    request: BatchRequest,
    background_tasks: BackgroundTasks
):
    """Создаёт асинхронную batch job."""
    job_id = str(uuid.uuid4())

    job = BatchJob(
        job_id=job_id,
        status='pending',
        progress=0,
        total=len(request.items),
        created_at=datetime.utcnow()
    )

    batch_jobs[job_id] = job

    # Запускаем обработку в фоне
    background_tasks.add_task(
        process_batch_async,
        job_id,
        request.items
    )

    return {
        'job_id': job_id,
        'status_url': f'/api/v1/batch/{job_id}/status'
    }

@app.get('/api/v1/batch/{job_id}/status')
async def get_batch_status(job_id: str):
    """Получает статус batch job."""
    if job_id not in batch_jobs:
        raise HTTPException(404, 'Job not found')
    return batch_jobs[job_id]

async def process_batch_async(job_id: str, items: List[BatchItem]):
    """Асинхронная обработка batch job."""
    job = batch_jobs[job_id]
    job.status = 'processing'

    results = []

    for i, item in enumerate(items):
        try:
            result = await process_item(item)
            results.append({'id': item.id, 'success': True, 'data': result})
        except Exception as e:
            results.append({'id': item.id, 'success': False, 'error': str(e)})

        job.progress = i + 1

    # Сохраняем результаты
    results_key = f'batch_results:{job_id}'
    await redis.setex(results_key, 86400, json.dumps(results))

    job.status = 'completed'
    job.completed_at = datetime.utcnow()
    job.results_url = f'/api/v1/batch/{job_id}/results'
```

### Pagination для больших результатов

```python
from fastapi import FastAPI, Query
from typing import List, Optional

app = FastAPI()

class PaginatedResponse(BaseModel):
    items: List[dict]
    total: int
    page: int
    page_size: int
    has_next: bool
    has_prev: bool
    next_cursor: Optional[str] = None

@app.get('/api/v1/users')
async def list_users(
    page: int = Query(1, ge=1),
    page_size: int = Query(100, ge=1, le=1000),
    cursor: Optional[str] = None
):
    """
    Пагинированный список пользователей.
    Поддерживает offset и cursor-based пагинацию.
    """
    if cursor:
        # Cursor-based пагинация (эффективнее для больших данных)
        users, next_cursor = await get_users_after_cursor(cursor, page_size)
        return {
            'items': users,
            'next_cursor': next_cursor,
            'has_next': next_cursor is not None
        }
    else:
        # Offset-based пагинация
        offset = (page - 1) * page_size
        users = await get_users_paginated(offset, page_size)
        total = await get_users_count()

        return PaginatedResponse(
            items=users,
            total=total,
            page=page,
            page_size=page_size,
            has_next=offset + page_size < total,
            has_prev=page > 1
        )

# Bulk export endpoint
@app.get('/api/v1/users/export')
async def export_users(format: str = 'csv'):
    """
    Экспорт всех пользователей.
    Возвращает streaming response для больших данных.
    """
    async def generate():
        if format == 'csv':
            yield 'id,name,email\n'

        async for batch in get_users_in_batches(batch_size=1000):
            for user in batch:
                if format == 'csv':
                    yield f"{user['id']},{user['name']},{user['email']}\n"
                else:
                    yield json.dumps(user) + '\n'

    return StreamingResponse(
        generate(),
        media_type='text/csv' if format == 'csv' else 'application/x-ndjson',
        headers={'Content-Disposition': f'attachment; filename=users.{format}'}
    )
```

---

## Best Practices

### 1. Checkpointing для долгих задач

```python
class CheckpointedBatchProcessor:
    """
    Batch processor с checkpoint'ами для возобновления после сбоев.
    """

    def __init__(self, job_id: str, redis_client):
        self.job_id = job_id
        self.redis = redis_client
        self.checkpoint_key = f'batch:checkpoint:{job_id}'

    async def get_checkpoint(self) -> dict:
        """Получает последний checkpoint."""
        data = await self.redis.get(self.checkpoint_key)
        return json.loads(data) if data else {'offset': 0, 'processed': 0}

    async def save_checkpoint(self, offset: int, processed: int):
        """Сохраняет checkpoint."""
        await self.redis.setex(
            self.checkpoint_key,
            86400,  # 24 часа
            json.dumps({'offset': offset, 'processed': processed})
        )

    async def process_with_checkpoints(self, items: List, process_func):
        """Обрабатывает элементы с checkpoint'ами."""
        checkpoint = await self.get_checkpoint()
        start_offset = checkpoint['offset']
        processed_count = checkpoint['processed']

        for i, item in enumerate(items[start_offset:], start=start_offset):
            try:
                await process_func(item)
                processed_count += 1

                # Сохраняем checkpoint каждые 100 элементов
                if processed_count % 100 == 0:
                    await self.save_checkpoint(i + 1, processed_count)

            except Exception as e:
                # Сохраняем checkpoint перед ошибкой
                await self.save_checkpoint(i, processed_count)
                raise

        # Очищаем checkpoint после успешного завершения
        await self.redis.delete(self.checkpoint_key)
        return processed_count
```

### 2. Rate Limiting и Backpressure

```python
import asyncio
from contextlib import asynccontextmanager

class RateLimitedBatchProcessor:
    """
    Batch processor с rate limiting.
    """

    def __init__(self, rate_limit: int = 100, time_window: int = 1):
        """
        Args:
            rate_limit: Максимум операций за time_window
            time_window: Окно времени в секундах
        """
        self.rate_limit = rate_limit
        self.time_window = time_window
        self.semaphore = asyncio.Semaphore(rate_limit)
        self.tokens = rate_limit
        self.last_refill = asyncio.get_event_loop().time()

    async def acquire(self):
        """Получает токен для операции."""
        while True:
            now = asyncio.get_event_loop().time()
            time_passed = now - self.last_refill

            # Пополняем токены
            if time_passed >= self.time_window:
                self.tokens = self.rate_limit
                self.last_refill = now

            if self.tokens > 0:
                self.tokens -= 1
                return

            # Ждём до следующего пополнения
            await asyncio.sleep(self.time_window - time_passed)

    async def process_with_rate_limit(self, items: List, process_func):
        """Обрабатывает элементы с rate limiting."""
        results = []

        for item in items:
            await self.acquire()
            result = await process_func(item)
            results.append(result)

        return results
```

### 3. Мониторинг и метрики

```python
from prometheus_client import Counter, Histogram, Gauge
import time

# Метрики
BATCH_JOBS_TOTAL = Counter(
    'batch_jobs_total',
    'Total batch jobs',
    ['job_type', 'status']
)

BATCH_DURATION = Histogram(
    'batch_job_duration_seconds',
    'Batch job duration',
    ['job_type']
)

BATCH_ITEMS_PROCESSED = Counter(
    'batch_items_processed_total',
    'Total items processed',
    ['job_type', 'status']
)

BATCH_QUEUE_SIZE = Gauge(
    'batch_queue_size',
    'Current batch queue size',
    ['job_type']
)

class MonitoredBatchProcessor:
    def __init__(self, job_type: str):
        self.job_type = job_type

    async def process(self, items: List, process_func):
        start_time = time.time()
        successful = 0
        failed = 0

        try:
            for item in items:
                try:
                    await process_func(item)
                    successful += 1
                    BATCH_ITEMS_PROCESSED.labels(
                        job_type=self.job_type,
                        status='success'
                    ).inc()
                except Exception:
                    failed += 1
                    BATCH_ITEMS_PROCESSED.labels(
                        job_type=self.job_type,
                        status='failed'
                    ).inc()

            status = 'completed' if failed == 0 else 'partial'
            BATCH_JOBS_TOTAL.labels(
                job_type=self.job_type,
                status=status
            ).inc()

        except Exception:
            BATCH_JOBS_TOTAL.labels(
                job_type=self.job_type,
                status='failed'
            ).inc()
            raise

        finally:
            duration = time.time() - start_time
            BATCH_DURATION.labels(job_type=self.job_type).observe(duration)

        return {'successful': successful, 'failed': failed, 'duration': duration}
```

---

## Типичные ошибки

### 1. Отсутствие идемпотентности

```python
# ❌ Плохо: повторный запуск создаст дубликаты
async def process_batch(items):
    for item in items:
        await db.insert(item)

# ✅ Хорошо: идемпотентная обработка
async def process_batch(items):
    for item in items:
        await db.execute("""
            INSERT INTO items (id, data)
            VALUES ($1, $2)
            ON CONFLICT (id) DO UPDATE SET data = $2
        """, item['id'], item['data'])
```

### 2. Отсутствие checkpoint'ов

```python
# ❌ Плохо: при сбое начинаем сначала
async def process_million_records():
    records = await get_all_records()  # 1 млн записей
    for record in records:
        await process(record)  # Если упадёт на 900000 — начинаем сначала

# ✅ Хорошо: с checkpoint'ами
async def process_million_records():
    checkpoint = await get_checkpoint()

    async for batch in get_records_from(checkpoint.offset):
        for record in batch:
            await process(record)

        await save_checkpoint(batch[-1].id)
```

### 3. Блокировка всей системы

```python
# ❌ Плохо: блокирующая обработка
def process_large_batch():
    items = get_all_items()  # Загружает всё в память
    for item in items:
        process(item)  # Блокирует на часы

# ✅ Хорошо: streaming + background
async def process_large_batch_async():
    job_id = create_job()

    async for batch in stream_items(batch_size=1000):
        await process_batch_async(batch)
        await update_job_progress(job_id)

    await complete_job(job_id)
```

---

## Сравнение инструментов

| Инструмент | Use Case | Особенности |
|------------|----------|-------------|
| Celery | Task queues | Популярный, много бэкендов |
| Apache Airflow | Complex DAGs | Визуализация, scheduling |
| Luigi | Data pipelines | Dependency management |
| Prefect | Modern workflows | Cloud-native, Python |
| dbt | Data transformation | SQL-based, analytics |
| Apache Spark | Big Data | Distributed, ML support |

---

## Заключение

Batch Processing — мощный паттерн для обработки больших объёмов данных. Ключевые принципы:

1. **Используйте checkpoint'ы** — для возобновления после сбоев
2. **Делайте обработку идемпотентной** — повторные запуски безопасны
3. **Мониторьте выполнение** — метрики, алерты, логи
4. **Планируйте ресурсы** — batch jobs потребляют много ресурсов
5. **Тестируйте на реальных объёмах** — поведение может отличаться

Выбирайте между real-time и batch исходя из требований к latency и объёму данных.
