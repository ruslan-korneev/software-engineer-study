# Точный контроль с помощью wait

[prev: ./05-as-completed.md](./05-as-completed.md) | [next: ../05-databases/readme.md](../05-databases/readme.md)

---

## Что такое asyncio.wait?

`asyncio.wait()` - это низкоуровневая функция для ожидания завершения нескольких задач. В отличие от `gather`, она предоставляет более тонкий контроль над тем, как и когда ожидать задачи.

```python
import asyncio

async def task(name: str, delay: float) -> str:
    await asyncio.sleep(delay)
    return f"Задача {name}"

async def main():
    tasks = [
        asyncio.create_task(task("A", 1)),
        asyncio.create_task(task("B", 2)),
        asyncio.create_task(task("C", 3)),
    ]

    # wait возвращает два множества: завершенные и ожидающие
    done, pending = await asyncio.wait(tasks)

    print(f"Завершено: {len(done)}")
    print(f"В ожидании: {len(pending)}")

    for t in done:
        print(f"Результат: {t.result()}")

asyncio.run(main())
```

## Сигнатура функции

```python
asyncio.wait(aws, *, timeout=None, return_when=ALL_COMPLETED)
```

- `aws` - итерируемый объект с задачами (НЕ корутинами!)
- `timeout` - максимальное время ожидания в секундах
- `return_when` - условие возврата:
  - `asyncio.FIRST_COMPLETED` - вернуться при завершении первой задачи
  - `asyncio.FIRST_EXCEPTION` - вернуться при первом исключении (или когда все завершены)
  - `asyncio.ALL_COMPLETED` - ждать завершения всех задач (по умолчанию)

## Важное отличие от gather

```python
import asyncio

async def example():
    # gather принимает корутины или задачи
    results = await asyncio.gather(
        some_coroutine(),  # Корутина - OK
        asyncio.create_task(another_coroutine()),  # Задача - OK
    )

    # wait принимает ТОЛЬКО задачи!
    tasks = [
        asyncio.create_task(some_coroutine()),
        asyncio.create_task(another_coroutine()),
    ]
    done, pending = await asyncio.wait(tasks)

    # ОШИБКА: передача корутин
    # done, pending = await asyncio.wait([some_coroutine()])  # DeprecationWarning!
```

## Режимы return_when

### FIRST_COMPLETED

Возвращается, как только завершилась хотя бы одна задача:

```python
import asyncio
import time

async def task(name: str, delay: float) -> str:
    await asyncio.sleep(delay)
    return f"Задача {name}"

async def main():
    tasks = [
        asyncio.create_task(task("A", 3)),
        asyncio.create_task(task("B", 1)),  # Завершится первой
        asyncio.create_task(task("C", 2)),
    ]

    start = time.time()

    done, pending = await asyncio.wait(
        tasks,
        return_when=asyncio.FIRST_COMPLETED
    )

    print(f"Время: {time.time() - start:.1f}с")
    print(f"Завершено: {len(done)}, В ожидании: {len(pending)}")

    for t in done:
        print(f"Результат: {t.result()}")

    # Не забудьте отменить или дождаться pending задачи!
    for t in pending:
        t.cancel()

asyncio.run(main())

# Вывод:
# Время: 1.0с
# Завершено: 1, В ожидании: 2
# Результат: Задача B
```

### FIRST_EXCEPTION

Возвращается при первом исключении или когда все задачи завершены:

```python
import asyncio

async def success_task(id: int) -> str:
    await asyncio.sleep(2)
    return f"Успех {id}"

async def failing_task() -> str:
    await asyncio.sleep(0.5)
    raise ValueError("Ошибка!")

async def main():
    tasks = [
        asyncio.create_task(success_task(1)),
        asyncio.create_task(failing_task()),
        asyncio.create_task(success_task(2)),
    ]

    done, pending = await asyncio.wait(
        tasks,
        return_when=asyncio.FIRST_EXCEPTION
    )

    print(f"Завершено: {len(done)}, В ожидании: {len(pending)}")

    for t in done:
        if t.exception():
            print(f"Исключение: {t.exception()}")
        else:
            print(f"Результат: {t.result()}")

    # Отменяем незавершенные задачи
    for t in pending:
        t.cancel()

asyncio.run(main())

# Вывод:
# Завершено: 1, В ожидании: 2
# Исключение: Ошибка!
```

### ALL_COMPLETED

Ожидает завершения всех задач (поведение по умолчанию):

```python
import asyncio
import time

async def task(name: str, delay: float) -> str:
    await asyncio.sleep(delay)
    return f"Задача {name}"

async def main():
    tasks = [
        asyncio.create_task(task("A", 1)),
        asyncio.create_task(task("B", 2)),
        asyncio.create_task(task("C", 3)),
    ]

    start = time.time()

    done, pending = await asyncio.wait(
        tasks,
        return_when=asyncio.ALL_COMPLETED
    )

    print(f"Время: {time.time() - start:.1f}с")
    print(f"Завершено: {len(done)}, В ожидании: {len(pending)}")

asyncio.run(main())

# Вывод:
# Время: 3.0с
# Завершено: 3, В ожидании: 0
```

## Таймаут

Таймаут позволяет ограничить время ожидания:

```python
import asyncio

async def slow_task(id: int) -> str:
    await asyncio.sleep(id)
    return f"Задача {id}"

async def main():
    tasks = [
        asyncio.create_task(slow_task(1)),
        asyncio.create_task(slow_task(5)),  # Не успеет
        asyncio.create_task(slow_task(2)),
    ]

    # Ждем максимум 3 секунды
    done, pending = await asyncio.wait(tasks, timeout=3)

    print(f"Завершено за 3с: {len(done)}")
    print(f"Не успели: {len(pending)}")

    # Результаты завершенных
    for t in done:
        print(f"Результат: {t.result()}")

    # Отменяем незавершенные
    for t in pending:
        print(f"Отмена задачи...")
        t.cancel()

asyncio.run(main())

# Вывод:
# Завершено за 3с: 2
# Не успели: 1
# Результат: Задача 1
# Результат: Задача 2
# Отмена задачи...
```

## Практические примеры с aiohttp

### Получение первого успешного ответа

```python
import aiohttp
import asyncio

async def fetch(session: aiohttp.ClientSession, url: str) -> dict:
    async with session.get(url) as response:
        return {'url': url, 'data': await response.json()}

async def fetch_first_available(urls: list[str]) -> dict:
    """Возвращает результат первого успешного запроса"""
    async with aiohttp.ClientSession() as session:
        tasks = [asyncio.create_task(fetch(session, url)) for url in urls]

        while tasks:
            done, pending = await asyncio.wait(
                tasks,
                return_when=asyncio.FIRST_COMPLETED
            )

            for task in done:
                try:
                    result = task.result()
                    # Успех! Отменяем остальные
                    for p in pending:
                        p.cancel()
                    return result
                except Exception:
                    # Эта задача упала, продолжаем ждать
                    pass

            tasks = list(pending)

        raise Exception("Все запросы не удались")

async def main():
    urls = [
        'https://httpbin.org/delay/3',
        'https://httpbin.org/delay/1',  # Самый быстрый
        'https://httpbin.org/delay/2',
    ]

    result = await fetch_first_available(urls)
    print(f"Получено с {result['url']}")

asyncio.run(main())
```

### Загрузка с таймаутом и повторными попытками

```python
import aiohttp
import asyncio
from typing import Optional

async def fetch_with_timeout(
    session: aiohttp.ClientSession,
    url: str,
    timeout: float
) -> Optional[dict]:
    """Загружает URL с таймаутом"""
    task = asyncio.create_task(
        session.get(url)
    )

    done, pending = await asyncio.wait([task], timeout=timeout)

    if pending:
        # Таймаут - отменяем
        task.cancel()
        try:
            await task
        except asyncio.CancelledError:
            pass
        return None

    # Задача завершена
    response = task.result()
    async with response:
        return {'url': url, 'data': await response.json()}

async def fetch_with_retry(
    session: aiohttp.ClientSession,
    url: str,
    timeout: float = 5.0,
    max_retries: int = 3
) -> Optional[dict]:
    """Загружает с повторными попытками"""
    for attempt in range(1, max_retries + 1):
        print(f"Попытка {attempt}/{max_retries} для {url}")
        result = await fetch_with_timeout(session, url, timeout)
        if result:
            return result
        await asyncio.sleep(0.5)  # Пауза между попытками

    return None

async def main():
    async with aiohttp.ClientSession() as session:
        result = await fetch_with_retry(
            session,
            'https://httpbin.org/delay/2',
            timeout=1.0,  # Слишком короткий таймаут
            max_retries=3
        )

        if result:
            print(f"Успех: {result['url']}")
        else:
            print("Все попытки неудачны")

asyncio.run(main())
```

### Пакетная обработка с контролем прогресса

```python
import aiohttp
import asyncio
import time

async def process_url(session: aiohttp.ClientSession, url: str, id: int) -> dict:
    async with session.get(url) as response:
        data = await response.json()
        return {'id': id, 'url': url, 'status': response.status}

async def process_batch(urls: list[str], batch_timeout: float = 10.0):
    """Обрабатывает пакет URL с общим таймаутом"""
    async with aiohttp.ClientSession() as session:
        tasks = [
            asyncio.create_task(process_url(session, url, i))
            for i, url in enumerate(urls)
        ]

        start = time.time()
        results = []
        remaining_timeout = batch_timeout

        while tasks and remaining_timeout > 0:
            done, pending = await asyncio.wait(
                tasks,
                timeout=remaining_timeout,
                return_when=asyncio.FIRST_COMPLETED
            )

            for task in done:
                try:
                    result = task.result()
                    results.append(result)
                    elapsed = time.time() - start
                    print(f"[{len(results)}/{len(urls)}] {elapsed:.1f}с - ID {result['id']}")
                except Exception as e:
                    print(f"Ошибка: {e}")

            tasks = list(pending)
            remaining_timeout = batch_timeout - (time.time() - start)

        # Отменяем незавершенные
        if tasks:
            print(f"\nТаймаут! Отмена {len(tasks)} задач")
            for task in tasks:
                task.cancel()

        return results

async def main():
    urls = [
        'https://httpbin.org/delay/1',
        'https://httpbin.org/delay/3',
        'https://httpbin.org/delay/2',
        'https://httpbin.org/delay/5',
    ]

    results = await process_batch(urls, batch_timeout=4.0)
    print(f"\nОбработано: {len(results)} из {len(urls)}")

asyncio.run(main())
```

## Сравнение wait, gather и as_completed

| Характеристика | wait | gather | as_completed |
|---------------|------|--------|--------------|
| Принимает корутины | Нет (только Task) | Да | Да |
| Возвращает | (done, pending) | Список результатов | Итератор |
| Порядок результатов | Не определен | Сохраняется | По мере готовности |
| Частичный результат | Да | Нет | Да (по итерации) |
| Гибкий контроль | Высокий | Низкий | Средний |
| return_when | Да | Нет | Нет |

## Когда использовать wait

**Используйте wait когда:**
- Нужно получить первый завершенный результат
- Нужно обработать исключение немедленно
- Требуется таймаут на группу задач
- Нужен контроль над незавершенными задачами
- Реализуете сложную логику ожидания

**Используйте gather когда:**
- Нужны все результаты
- Важен порядок результатов
- Достаточно простой обработки ошибок

**Используйте as_completed когда:**
- Нужно обрабатывать результаты по мере поступления
- Не нужен доступ к pending задачам

## Паттерн: цикл обработки с wait

```python
import asyncio

async def worker(id: int, duration: float):
    await asyncio.sleep(duration)
    if id == 2:
        raise ValueError(f"Worker {id} failed")
    return f"Worker {id} done"

async def process_with_recovery():
    """Обрабатывает задачи с восстановлением после ошибок"""
    tasks = {
        asyncio.create_task(worker(1, 2)): 1,
        asyncio.create_task(worker(2, 1)): 2,  # Упадет
        asyncio.create_task(worker(3, 3)): 3,
    }

    results = []
    errors = []

    while tasks:
        done, pending = await asyncio.wait(
            tasks.keys(),
            return_when=asyncio.FIRST_COMPLETED
        )

        for task in done:
            worker_id = tasks.pop(task)
            try:
                result = task.result()
                results.append((worker_id, result))
                print(f"Worker {worker_id}: успех")
            except Exception as e:
                errors.append((worker_id, str(e)))
                print(f"Worker {worker_id}: ошибка - {e}")

                # Можно перезапустить задачу
                # new_task = asyncio.create_task(worker(worker_id, 1))
                # tasks[new_task] = worker_id

    return results, errors

async def main():
    results, errors = await process_with_recovery()
    print(f"\nУспешно: {len(results)}, Ошибок: {len(errors)}")

asyncio.run(main())
```

## Паттерн: конкурентный запрос с fallback

```python
import aiohttp
import asyncio

async def fetch(session: aiohttp.ClientSession, url: str) -> dict:
    async with session.get(url) as response:
        return {'url': url, 'data': await response.json()}

async def fetch_with_fallback(
    primary_url: str,
    fallback_urls: list[str],
    timeout: float = 5.0
) -> dict:
    """Запрашивает primary, при неудаче пробует fallback"""
    async with aiohttp.ClientSession() as session:
        # Сначала пробуем primary
        primary_task = asyncio.create_task(fetch(session, primary_url))

        done, pending = await asyncio.wait(
            [primary_task],
            timeout=timeout
        )

        if done:
            try:
                return primary_task.result()
            except Exception:
                pass  # Primary упал, пробуем fallback
        else:
            primary_task.cancel()

        # Пробуем fallback конкурентно
        if fallback_urls:
            fallback_tasks = [
                asyncio.create_task(fetch(session, url))
                for url in fallback_urls
            ]

            while fallback_tasks:
                done, pending = await asyncio.wait(
                    fallback_tasks,
                    timeout=timeout,
                    return_when=asyncio.FIRST_COMPLETED
                )

                for task in done:
                    try:
                        result = task.result()
                        # Успех! Отменяем остальные
                        for t in pending:
                            t.cancel()
                        return result
                    except Exception:
                        pass

                fallback_tasks = list(pending)
                if not done:  # Таймаут
                    break

        raise Exception("Все источники недоступны")

async def main():
    try:
        result = await fetch_with_fallback(
            primary_url='https://httpbin.org/delay/10',  # Слишком медленный
            fallback_urls=[
                'https://httpbin.org/delay/2',
                'https://httpbin.org/get',  # Быстрый
            ],
            timeout=3.0
        )
        print(f"Получено с {result['url']}")
    except Exception as e:
        print(f"Ошибка: {e}")

asyncio.run(main())
```

## Лучшие практики

1. **Всегда обрабатывайте pending задачи** - отменяйте или ожидайте их
2. **Используйте Task, не корутины** - wait требует объекты Task
3. **Проверяйте исключения** через `task.exception()` перед `task.result()`
4. **Устанавливайте разумные таймауты** для защиты от зависания
5. **Используйте словарь task->metadata** для отслеживания задач

## Частые ошибки

```python
# ОШИБКА: Передача корутин вместо задач
done, pending = await asyncio.wait([some_coroutine()])  # DeprecationWarning!

# ПРАВИЛЬНО:
tasks = [asyncio.create_task(some_coroutine())]
done, pending = await asyncio.wait(tasks)

# ОШИБКА: Игнорирование pending задач
done, pending = await asyncio.wait(tasks, timeout=5)
for t in done:
    print(t.result())
# pending задачи продолжают выполняться! Утечка ресурсов

# ПРАВИЛЬНО:
done, pending = await asyncio.wait(tasks, timeout=5)
for t in done:
    print(t.result())
for t in pending:
    t.cancel()

# ОШИБКА: Вызов result() без проверки исключения
for t in done:
    print(t.result())  # Может выбросить исключение!

# ПРАВИЛЬНО:
for t in done:
    if t.exception():
        print(f"Ошибка: {t.exception()}")
    else:
        print(t.result())
```

---

[prev: ./05-as-completed.md](./05-as-completed.md) | [next: ../05-databases/readme.md](../05-databases/readme.md)
