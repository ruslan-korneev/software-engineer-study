# Очереди в PostgreSQL (Queues)

## Введение

Очереди сообщений в PostgreSQL позволяют реализовать асинхронную обработку задач без внешних систем (Redis, RabbitMQ). PostgreSQL предоставляет встроенные механизмы (LISTEN/NOTIFY) и расширения (pgq, pg_boss) для реализации очередей.

## Зачем использовать очереди в PostgreSQL

### Преимущества

1. **Транзакционная целостность** - сообщения в рамках той же транзакции, что и данные
2. **Нет дополнительной инфраструктуры** - всё в одной базе данных
3. **ACID гарантии** - надежная доставка сообщений
4. **Простота развертывания** - не нужны отдельные сервисы

### Когда использовать

- Небольшие и средние нагрузки (до тысяч сообщений в секунду)
- Критична транзакционная целостность
- Нужна простота операций
- Не требуется горизонтальное масштабирование очереди

## LISTEN/NOTIFY - встроенный механизм

### Базовое использование

```sql
-- Сессия 1: Подписка на канал
LISTEN my_channel;

-- Сессия 2: Отправка уведомления
NOTIFY my_channel, 'Hello, World!';

-- Сессия 1 получит:
-- Asynchronous notification "my_channel" with payload "Hello, World!" received from server process with PID 12345.
```

### Использование с функцией pg_notify

```sql
-- pg_notify позволяет использовать переменные
SELECT pg_notify('my_channel', 'Message from function');

-- Использование в транзакции
BEGIN;
INSERT INTO orders (customer_id, amount) VALUES (1, 100.00);
SELECT pg_notify('new_order', '{"order_id": 123, "amount": 100}');
COMMIT;
-- Уведомление отправится только после COMMIT
```

### Python клиент для LISTEN/NOTIFY

```python
import psycopg2
import select
import json

def listen_for_notifications():
    conn = psycopg2.connect("dbname=mydb user=postgres")
    conn.set_isolation_level(psycopg2.extensions.ISOLATION_LEVEL_AUTOCOMMIT)

    cursor = conn.cursor()
    cursor.execute("LISTEN new_order;")

    print("Waiting for notifications...")

    while True:
        # Ожидание уведомлений
        if select.select([conn], [], [], 5) == ([], [], []):
            print("Timeout, still waiting...")
        else:
            conn.poll()
            while conn.notifies:
                notify = conn.notifies.pop(0)
                print(f"Got notification: {notify.channel}")
                print(f"Payload: {notify.payload}")

                # Обработка сообщения
                data = json.loads(notify.payload)
                process_order(data)

def process_order(data):
    print(f"Processing order: {data}")

if __name__ == "__main__":
    listen_for_notifications()
```

### Асинхронный клиент (asyncpg)

```python
import asyncio
import asyncpg
import json

async def listener():
    conn = await asyncpg.connect('postgresql://postgres@localhost/mydb')

    async def notification_handler(connection, pid, channel, payload):
        print(f"Received: {payload}")
        data = json.loads(payload)
        await process_order_async(data)

    await conn.add_listener('new_order', notification_handler)

    print("Listening for notifications...")
    # Держим соединение открытым
    while True:
        await asyncio.sleep(1)

async def process_order_async(data):
    print(f"Async processing: {data}")

asyncio.run(listener())
```

## Реализация очереди на таблице

### Простая очередь задач

```sql
-- Таблица очереди
CREATE TABLE task_queue (
    id BIGSERIAL PRIMARY KEY,
    task_type VARCHAR(50) NOT NULL,
    payload JSONB NOT NULL,
    status VARCHAR(20) DEFAULT 'pending',
    priority INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    error_message TEXT,
    retry_count INTEGER DEFAULT 0,
    max_retries INTEGER DEFAULT 3
);

-- Индексы для эффективной работы
CREATE INDEX idx_queue_pending ON task_queue(priority DESC, created_at)
    WHERE status = 'pending';
CREATE INDEX idx_queue_status ON task_queue(status);
```

### Атомарное получение задачи (с блокировкой)

```sql
-- Функция для получения следующей задачи
CREATE OR REPLACE FUNCTION get_next_task()
RETURNS task_queue AS $$
DECLARE
    task task_queue;
BEGIN
    -- FOR UPDATE SKIP LOCKED позволяет конкурентным worker-ам
    -- пропускать заблокированные строки
    SELECT * INTO task
    FROM task_queue
    WHERE status = 'pending'
    ORDER BY priority DESC, created_at
    LIMIT 1
    FOR UPDATE SKIP LOCKED;

    IF FOUND THEN
        UPDATE task_queue
        SET status = 'processing',
            started_at = NOW()
        WHERE id = task.id;

        task.status := 'processing';
        task.started_at := NOW();
    END IF;

    RETURN task;
END;
$$ LANGUAGE plpgsql;

-- Использование
SELECT * FROM get_next_task();
```

### Завершение задачи

```sql
-- Успешное завершение
CREATE OR REPLACE FUNCTION complete_task(task_id BIGINT)
RETURNS VOID AS $$
BEGIN
    UPDATE task_queue
    SET status = 'completed',
        completed_at = NOW()
    WHERE id = task_id;
END;
$$ LANGUAGE plpgsql;

-- Завершение с ошибкой (с retry)
CREATE OR REPLACE FUNCTION fail_task(task_id BIGINT, error_msg TEXT)
RETURNS VOID AS $$
DECLARE
    task task_queue;
BEGIN
    SELECT * INTO task FROM task_queue WHERE id = task_id;

    IF task.retry_count < task.max_retries THEN
        UPDATE task_queue
        SET status = 'pending',
            retry_count = retry_count + 1,
            error_message = error_msg,
            started_at = NULL
        WHERE id = task_id;
    ELSE
        UPDATE task_queue
        SET status = 'failed',
            error_message = error_msg,
            completed_at = NOW()
        WHERE id = task_id;
    END IF;
END;
$$ LANGUAGE plpgsql;
```

### Python Worker

```python
import psycopg2
import time
import json

class TaskWorker:
    def __init__(self, db_url):
        self.conn = psycopg2.connect(db_url)
        self.conn.autocommit = True

    def get_task(self):
        with self.conn.cursor() as cursor:
            cursor.execute("SELECT * FROM get_next_task()")
            row = cursor.fetchone()
            if row and row[0]:  # Если задача найдена
                return {
                    'id': row[0],
                    'task_type': row[1],
                    'payload': row[2]
                }
        return None

    def complete_task(self, task_id):
        with self.conn.cursor() as cursor:
            cursor.execute("SELECT complete_task(%s)", (task_id,))

    def fail_task(self, task_id, error):
        with self.conn.cursor() as cursor:
            cursor.execute("SELECT fail_task(%s, %s)", (task_id, str(error)))

    def process_task(self, task):
        task_type = task['task_type']
        payload = task['payload']

        # Dispatch по типу задачи
        if task_type == 'send_email':
            self.send_email(payload)
        elif task_type == 'process_payment':
            self.process_payment(payload)
        else:
            raise ValueError(f"Unknown task type: {task_type}")

    def send_email(self, payload):
        print(f"Sending email to {payload['to']}")
        # Логика отправки email

    def process_payment(self, payload):
        print(f"Processing payment: {payload['amount']}")
        # Логика обработки платежа

    def run(self):
        print("Worker started...")
        while True:
            task = self.get_task()

            if task:
                print(f"Processing task {task['id']}: {task['task_type']}")
                try:
                    self.process_task(task)
                    self.complete_task(task['id'])
                    print(f"Task {task['id']} completed")
                except Exception as e:
                    self.fail_task(task['id'], e)
                    print(f"Task {task['id']} failed: {e}")
            else:
                time.sleep(1)  # Нет задач, ждем

if __name__ == "__main__":
    worker = TaskWorker("dbname=mydb user=postgres")
    worker.run()
```

## Advisory Locks для координации

```sql
-- Использование advisory locks для эксклюзивной обработки
CREATE OR REPLACE FUNCTION get_task_with_advisory_lock()
RETURNS task_queue AS $$
DECLARE
    task task_queue;
BEGIN
    FOR task IN
        SELECT * FROM task_queue
        WHERE status = 'pending'
        ORDER BY priority DESC, created_at
    LOOP
        -- Пробуем получить advisory lock
        IF pg_try_advisory_lock(task.id) THEN
            UPDATE task_queue
            SET status = 'processing',
                started_at = NOW()
            WHERE id = task.id;

            RETURN task;
        END IF;
    END LOOP;

    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

-- Освобождение блокировки после обработки
SELECT pg_advisory_unlock(task_id);
```

## pgq - расширение для очередей

pgq - это расширение от Skype/Skytools для надежных очередей.

### Установка и настройка

```sql
-- Установка расширения
CREATE EXTENSION pgq;

-- Создание очереди
SELECT pgq.create_queue('my_queue');

-- Регистрация consumer
SELECT pgq.register_consumer('my_queue', 'my_consumer');
```

### Работа с pgq

```sql
-- Добавление события в очередь
SELECT pgq.insert_event('my_queue', 'order_created', '{"order_id": 123}');

-- Получение batch событий
SELECT pgq.next_batch('my_queue', 'my_consumer');
-- Возвращает batch_id

-- Получение событий из batch
SELECT * FROM pgq.get_batch_events(batch_id);

-- Подтверждение обработки batch
SELECT pgq.finish_batch(batch_id);
```

### Python клиент для pgq

```python
import psycopg2
import json

class PgqConsumer:
    def __init__(self, db_url, queue_name, consumer_name):
        self.conn = psycopg2.connect(db_url)
        self.conn.autocommit = True
        self.queue_name = queue_name
        self.consumer_name = consumer_name

    def process_batch(self):
        with self.conn.cursor() as cursor:
            # Получение batch
            cursor.execute(
                "SELECT pgq.next_batch(%s, %s)",
                (self.queue_name, self.consumer_name)
            )
            batch_id = cursor.fetchone()[0]

            if batch_id is None:
                return False  # Нет событий

            # Получение событий
            cursor.execute(
                "SELECT * FROM pgq.get_batch_events(%s)",
                (batch_id,)
            )
            events = cursor.fetchall()

            for event in events:
                event_id, batch_id, ev_type, ev_data, ev_extra1, ev_extra2, ev_extra3, ev_extra4, ev_time, ev_txid, ev_retry = event

                try:
                    self.handle_event(ev_type, json.loads(ev_data))
                except Exception as e:
                    print(f"Error processing event {event_id}: {e}")

            # Завершение batch
            cursor.execute("SELECT pgq.finish_batch(%s)", (batch_id,))
            return True

    def handle_event(self, event_type, data):
        print(f"Processing {event_type}: {data}")

    def run(self):
        while True:
            if not self.process_batch():
                time.sleep(1)
```

## SKIP LOCKED Pattern

Современный паттерн для конкурентной обработки очередей.

```sql
-- Оптимальная реализация с SKIP LOCKED
CREATE OR REPLACE FUNCTION dequeue_task(batch_size INTEGER DEFAULT 1)
RETURNS SETOF task_queue AS $$
BEGIN
    RETURN QUERY
    WITH selected AS (
        SELECT id
        FROM task_queue
        WHERE status = 'pending'
        ORDER BY priority DESC, created_at
        LIMIT batch_size
        FOR UPDATE SKIP LOCKED
    )
    UPDATE task_queue t
    SET status = 'processing',
        started_at = NOW()
    FROM selected s
    WHERE t.id = s.id
    RETURNING t.*;
END;
$$ LANGUAGE plpgsql;

-- Использование
BEGIN;
SELECT * FROM dequeue_task(10);  -- Получить до 10 задач
-- Обработка...
COMMIT;
```

## Отложенные задачи (Delayed Jobs)

```sql
-- Таблица с поддержкой отложенного выполнения
CREATE TABLE scheduled_tasks (
    id BIGSERIAL PRIMARY KEY,
    task_type VARCHAR(50) NOT NULL,
    payload JSONB NOT NULL,
    status VARCHAR(20) DEFAULT 'scheduled',
    scheduled_at TIMESTAMP NOT NULL,  -- Когда выполнить
    created_at TIMESTAMP DEFAULT NOW(),
    started_at TIMESTAMP,
    completed_at TIMESTAMP
);

-- Индекс для эффективной выборки
CREATE INDEX idx_scheduled_ready ON scheduled_tasks(scheduled_at)
    WHERE status = 'scheduled';

-- Получение готовых к выполнению задач
CREATE OR REPLACE FUNCTION get_ready_tasks(batch_size INTEGER DEFAULT 10)
RETURNS SETOF scheduled_tasks AS $$
BEGIN
    RETURN QUERY
    WITH selected AS (
        SELECT id
        FROM scheduled_tasks
        WHERE status = 'scheduled'
          AND scheduled_at <= NOW()
        ORDER BY scheduled_at
        LIMIT batch_size
        FOR UPDATE SKIP LOCKED
    )
    UPDATE scheduled_tasks t
    SET status = 'processing',
        started_at = NOW()
    FROM selected s
    WHERE t.id = s.id
    RETURNING t.*;
END;
$$ LANGUAGE plpgsql;

-- Создание отложенной задачи
INSERT INTO scheduled_tasks (task_type, payload, scheduled_at)
VALUES ('send_reminder', '{"user_id": 123}', NOW() + INTERVAL '1 hour');
```

## Очереди с приоритетами

```sql
-- Расширенная таблица с приоритетами
CREATE TABLE priority_queue (
    id BIGSERIAL PRIMARY KEY,
    task_type VARCHAR(50) NOT NULL,
    payload JSONB NOT NULL,
    priority INTEGER DEFAULT 0,  -- Выше число = выше приоритет
    status VARCHAR(20) DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT NOW()
);

-- Partial index для разных приоритетов
CREATE INDEX idx_pq_high ON priority_queue(created_at)
    WHERE status = 'pending' AND priority >= 8;

CREATE INDEX idx_pq_medium ON priority_queue(created_at)
    WHERE status = 'pending' AND priority >= 4 AND priority < 8;

CREATE INDEX idx_pq_low ON priority_queue(created_at)
    WHERE status = 'pending' AND priority < 4;

-- Получение с учетом приоритета
SELECT * FROM priority_queue
WHERE status = 'pending'
ORDER BY priority DESC, created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;
```

## Dead Letter Queue (DLQ)

```sql
-- Таблица для "мертвых" сообщений
CREATE TABLE dead_letter_queue (
    id BIGSERIAL PRIMARY KEY,
    original_id BIGINT,
    original_table VARCHAR(50),
    task_type VARCHAR(50),
    payload JSONB,
    error_message TEXT,
    retry_count INTEGER,
    failed_at TIMESTAMP DEFAULT NOW()
);

-- Перемещение в DLQ после максимума retry
CREATE OR REPLACE FUNCTION move_to_dlq(task_id BIGINT, error_msg TEXT)
RETURNS VOID AS $$
BEGIN
    INSERT INTO dead_letter_queue (original_id, original_table, task_type, payload, error_message, retry_count)
    SELECT id, 'task_queue', task_type, payload, error_msg, retry_count
    FROM task_queue
    WHERE id = task_id;

    DELETE FROM task_queue WHERE id = task_id;
END;
$$ LANGUAGE plpgsql;
```

## Мониторинг очередей

```sql
-- Статистика очереди
CREATE VIEW queue_stats AS
SELECT
    status,
    COUNT(*) as count,
    MIN(created_at) as oldest,
    MAX(created_at) as newest,
    AVG(EXTRACT(EPOCH FROM (NOW() - created_at))) as avg_age_seconds
FROM task_queue
GROUP BY status;

-- Мониторинг отставания
SELECT
    COUNT(*) FILTER (WHERE status = 'pending') as pending,
    COUNT(*) FILTER (WHERE status = 'processing') as processing,
    COUNT(*) FILTER (WHERE status = 'completed') as completed,
    COUNT(*) FILTER (WHERE status = 'failed') as failed,
    COUNT(*) FILTER (WHERE status = 'pending' AND created_at < NOW() - INTERVAL '5 minutes') as stale
FROM task_queue;

-- Throughput за последний час
SELECT
    DATE_TRUNC('minute', completed_at) as minute,
    COUNT(*) as completed_count
FROM task_queue
WHERE completed_at > NOW() - INTERVAL '1 hour'
GROUP BY DATE_TRUNC('minute', completed_at)
ORDER BY minute;
```

## Очистка старых записей

```sql
-- Автоматическая очистка завершенных задач
CREATE OR REPLACE FUNCTION cleanup_completed_tasks(retention_days INTEGER DEFAULT 7)
RETURNS INTEGER AS $$
DECLARE
    deleted_count INTEGER;
BEGIN
    WITH deleted AS (
        DELETE FROM task_queue
        WHERE status IN ('completed', 'failed')
          AND completed_at < NOW() - (retention_days || ' days')::INTERVAL
        RETURNING id
    )
    SELECT COUNT(*) INTO deleted_count FROM deleted;

    RETURN deleted_count;
END;
$$ LANGUAGE plpgsql;

-- Запуск очистки (можно через cron или pg_cron)
SELECT cleanup_completed_tasks(7);
```

## Best Practices

### 1. Идемпотентность обработчиков

```python
def process_payment(payload):
    payment_id = payload['payment_id']

    # Проверка, не обработан ли уже платеж
    if is_payment_processed(payment_id):
        return  # Уже обработан, пропускаем

    # Обработка платежа
    execute_payment(payload)
    mark_payment_processed(payment_id)
```

### 2. Timeout для processing задач

```sql
-- Возврат "застрявших" задач
UPDATE task_queue
SET status = 'pending',
    started_at = NULL,
    retry_count = retry_count + 1
WHERE status = 'processing'
  AND started_at < NOW() - INTERVAL '10 minutes';
```

### 3. Батчевая обработка

```sql
-- Получение нескольких задач за раз
SELECT * FROM dequeue_task(100);  -- Batch из 100 задач
```

## Типичные ошибки

1. **Отсутствие SKIP LOCKED** - блокировки между worker-ами
2. **Нет индексов** - медленная выборка задач
3. **Polling без паузы** - высокая нагрузка на БД
4. **Нет retry логики** - потеря задач при ошибках
5. **Нет мониторинга** - не видно проблем с очередью
6. **Не очищаются старые записи** - рост таблицы
