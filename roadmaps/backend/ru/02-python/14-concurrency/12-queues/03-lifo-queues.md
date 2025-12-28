# LIFO-очереди

[prev: ./02-priority-queues.md](./02-priority-queues.md) | [next: ../13-subprocesses/readme.md](../13-subprocesses/readme.md)

---

## Введение

`asyncio.LifoQueue` - это асинхронная очередь, работающая по принципу LIFO (Last In, First Out - последний вошел, первый вышел). По сути, это стек, реализованный как асинхронная очередь.

## Принцип работы

В отличие от обычной FIFO-очереди, `LifoQueue` извлекает элементы в обратном порядке:

```python
import asyncio

async def lifo_vs_fifo():
    # Обычная очередь (FIFO)
    fifo = asyncio.Queue()
    await fifo.put(1)
    await fifo.put(2)
    await fifo.put(3)

    print("FIFO:")
    while not fifo.empty():
        item = await fifo.get()
        print(f"  Извлечен: {item}")
        fifo.task_done()

    # LIFO очередь (стек)
    lifo = asyncio.LifoQueue()
    await lifo.put(1)
    await lifo.put(2)
    await lifo.put(3)

    print("\nLIFO:")
    while not lifo.empty():
        item = await lifo.get()
        print(f"  Извлечен: {item}")
        lifo.task_done()

asyncio.run(lifo_vs_fifo())
# Вывод:
# FIFO:
#   Извлечен: 1
#   Извлечен: 2
#   Извлечен: 3
#
# LIFO:
#   Извлечен: 3
#   Извлечен: 2
#   Извлечен: 1
```

## Основные операции

API `LifoQueue` полностью совпадает с обычной `Queue`:

```python
import asyncio

async def lifo_operations():
    stack = asyncio.LifoQueue(maxsize=5)

    # Добавление элементов (push)
    await stack.put("first")
    await stack.put("second")
    await stack.put("third")

    # Проверка состояния
    print(f"Размер: {stack.qsize()}")      # 3
    print(f"Пуста: {stack.empty()}")        # False
    print(f"Полна: {stack.full()}")         # False

    # Извлечение элементов (pop)
    top = await stack.get()
    print(f"Верхний элемент: {top}")  # third
    stack.task_done()

    # Неблокирующие операции
    stack.put_nowait("fourth")
    top = stack.get_nowait()
    print(f"Новый верхний: {top}")  # fourth
    stack.task_done()

asyncio.run(lifo_operations())
```

## Практические применения

### 1. Обход в глубину (DFS)

LIFO-очереди идеально подходят для асинхронного обхода в глубину:

```python
import asyncio
from typing import Dict, List, Set

async def async_dfs(graph: Dict[str, List[str]], start: str):
    """Асинхронный обход графа в глубину"""
    stack = asyncio.LifoQueue()
    visited: Set[str] = set()
    result: List[str] = []

    await stack.put(start)

    while not stack.empty():
        node = await stack.get()
        stack.task_done()

        if node in visited:
            continue

        visited.add(node)
        result.append(node)
        print(f"Посещаем узел: {node}")

        # Имитация асинхронной операции (загрузка данных узла)
        await asyncio.sleep(0.1)

        # Добавляем соседей в обратном порядке для правильного обхода
        for neighbor in reversed(graph.get(node, [])):
            if neighbor not in visited:
                await stack.put(neighbor)

    return result

async def main():
    graph = {
        'A': ['B', 'C'],
        'B': ['D', 'E'],
        'C': ['F'],
        'D': [],
        'E': ['F'],
        'F': []
    }

    result = await async_dfs(graph, 'A')
    print(f"\nПорядок обхода: {result}")

asyncio.run(main())
```

### 2. История операций с возможностью отмены (Undo)

```python
import asyncio
from dataclasses import dataclass
from typing import Any, Callable, Awaitable

@dataclass
class Operation:
    name: str
    execute: Callable[[], Awaitable[Any]]
    undo: Callable[[], Awaitable[Any]]

class UndoableOperationManager:
    """Менеджер операций с поддержкой отмены"""

    def __init__(self):
        self._history = asyncio.LifoQueue()
        self._data = {"value": 0}  # Пример состояния

    async def execute(self, operation: Operation):
        """Выполнить операцию и сохранить в историю"""
        print(f"Выполняем: {operation.name}")
        await operation.execute()
        await self._history.put(operation)
        print(f"  Текущее состояние: {self._data}")

    async def undo(self):
        """Отменить последнюю операцию"""
        if self._history.empty():
            print("Нечего отменять!")
            return

        operation = await self._history.get()
        self._history.task_done()

        print(f"Отменяем: {operation.name}")
        await operation.undo()
        print(f"  Текущее состояние: {self._data}")

    async def undo_all(self):
        """Отменить все операции"""
        while not self._history.empty():
            await self.undo()

async def main():
    manager = UndoableOperationManager()

    # Создаем операции
    async def add_10():
        manager._data["value"] += 10

    async def subtract_10():
        manager._data["value"] -= 10

    async def multiply_2():
        manager._data["value"] *= 2

    async def divide_2():
        manager._data["value"] //= 2

    op1 = Operation("Добавить 10", add_10, subtract_10)
    op2 = Operation("Умножить на 2", multiply_2, divide_2)
    op3 = Operation("Добавить еще 10", add_10, subtract_10)

    # Выполняем операции
    await manager.execute(op1)  # value = 10
    await manager.execute(op2)  # value = 20
    await manager.execute(op3)  # value = 30

    print("\n--- Отмена операций ---")
    await manager.undo()  # value = 20
    await manager.undo()  # value = 10

    print(f"\nФинальное значение: {manager._data['value']}")

asyncio.run(main())
```

### 3. Асинхронный парсер с обходом вложенных структур

```python
import asyncio
from dataclasses import dataclass
from typing import Optional, List

@dataclass
class TreeNode:
    name: str
    children: List['TreeNode']
    depth: int = 0

async def fetch_children(node_name: str) -> List[str]:
    """Имитация асинхронной загрузки дочерних узлов"""
    await asyncio.sleep(0.1)

    structure = {
        "root": ["folder1", "folder2"],
        "folder1": ["file1.txt", "subfolder1"],
        "folder2": ["file2.txt", "file3.txt"],
        "subfolder1": ["deep_file.txt"],
    }
    return structure.get(node_name, [])

async def async_tree_traversal(root_name: str):
    """Асинхронный обход дерева в глубину"""
    stack = asyncio.LifoQueue()
    await stack.put((root_name, 0))

    result = []

    while not stack.empty():
        name, depth = await stack.get()
        stack.task_done()

        indent = "  " * depth
        print(f"{indent}{name}")
        result.append(name)

        # Загружаем дочерние элементы
        children = await fetch_children(name)

        # Добавляем в обратном порядке для правильной последовательности
        for child in reversed(children):
            await stack.put((child, depth + 1))

    return result

async def main():
    print("Структура файловой системы:")
    await async_tree_traversal("root")

asyncio.run(main())
```

### 4. Backtracking алгоритм (поиск с возвратом)

```python
import asyncio
from typing import List, Optional

async def solve_maze_async(maze: List[List[int]],
                           start: tuple,
                           end: tuple) -> Optional[List[tuple]]:
    """
    Асинхронное решение лабиринта с использованием backtracking.
    0 - проход, 1 - стена
    """
    rows, cols = len(maze), len(maze[0])
    stack = asyncio.LifoQueue()

    # (позиция, путь до этой позиции)
    await stack.put((start, [start]))
    visited = set()

    directions = [(0, 1), (1, 0), (0, -1), (-1, 0)]  # право, вниз, лево, вверх

    while not stack.empty():
        (row, col), path = await stack.get()
        stack.task_done()

        if (row, col) == end:
            return path

        if (row, col) in visited:
            continue

        visited.add((row, col))

        # Имитация асинхронной проверки (например, запрос к серверу)
        await asyncio.sleep(0.01)

        for dr, dc in directions:
            new_row, new_col = row + dr, col + dc

            if (0 <= new_row < rows and
                0 <= new_col < cols and
                maze[new_row][new_col] == 0 and
                (new_row, new_col) not in visited):

                await stack.put(((new_row, new_col), path + [(new_row, new_col)]))

    return None  # Путь не найден

async def main():
    maze = [
        [0, 0, 1, 0],
        [1, 0, 1, 0],
        [0, 0, 0, 0],
        [0, 1, 1, 0]
    ]

    start = (0, 0)
    end = (3, 3)

    print("Лабиринт:")
    for row in maze:
        print(row)

    path = await solve_maze_async(maze, start, end)

    if path:
        print(f"\nПуть найден: {path}")
    else:
        print("\nПуть не найден!")

asyncio.run(main())
```

### 5. Обработка рекурсивных зависимостей

```python
import asyncio
from typing import Dict, Set, List

async def resolve_dependencies(
    dependencies: Dict[str, List[str]],
    package: str
) -> List[str]:
    """
    Разрешение зависимостей пакетов с использованием LIFO-очереди.
    Возвращает список пакетов в порядке установки.
    """
    stack = asyncio.LifoQueue()
    resolved: List[str] = []
    visited: Set[str] = set()
    in_progress: Set[str] = set()

    await stack.put(package)

    while not stack.empty():
        current = await stack.get()
        stack.task_done()

        if current in resolved:
            continue

        if current in in_progress:
            # Все зависимости текущего пакета разрешены
            in_progress.remove(current)
            resolved.append(current)
            print(f"Установлен: {current}")
            continue

        if current in visited:
            continue

        visited.add(current)
        in_progress.add(current)

        # Возвращаем пакет в стек для финальной обработки
        await stack.put(current)

        # Имитация асинхронной загрузки информации о зависимостях
        await asyncio.sleep(0.05)

        deps = dependencies.get(current, [])
        for dep in deps:
            if dep not in resolved:
                await stack.put(dep)

    return resolved

async def main():
    dependencies = {
        "app": ["web-framework", "database-driver"],
        "web-framework": ["http-client", "json-parser"],
        "database-driver": ["connection-pool"],
        "http-client": ["ssl-lib"],
        "connection-pool": [],
        "json-parser": [],
        "ssl-lib": [],
    }

    print("Разрешение зависимостей для 'app':\n")
    install_order = await resolve_dependencies(dependencies, "app")
    print(f"\nПорядок установки: {install_order}")

asyncio.run(main())
```

## Ограниченная LIFO-очередь

```python
import asyncio

async def bounded_lifo_example():
    """Пример ограниченной LIFO-очереди"""
    stack = asyncio.LifoQueue(maxsize=3)

    await stack.put(1)
    await stack.put(2)
    await stack.put(3)

    print(f"Очередь заполнена: {stack.full()}")

    # Попытка добавить в полную очередь
    async def try_push():
        print("Пытаюсь добавить элемент в полный стек...")
        await stack.put(4)  # Будет ждать!
        print("Элемент добавлен!")

    async def pop_one():
        await asyncio.sleep(1)
        item = await stack.get()
        print(f"Извлечен: {item}")
        stack.task_done()

    await asyncio.gather(try_push(), pop_one())

    # Извлекаем оставшиеся элементы
    while not stack.empty():
        item = await stack.get()
        print(f"Извлечен: {item}")
        stack.task_done()

asyncio.run(bounded_lifo_example())
```

## Best Practices

### 1. Используйте LIFO для алгоритмов с возвратом

```python
# LIFO идеально подходит для:
# - DFS (обход в глубину)
# - Backtracking
# - Undo/Redo функциональности
# - Обработки вложенных структур
```

### 2. Следите за порядком добавления для DFS

```python
# Для правильного порядка обхода добавляйте соседей в обратном порядке
for neighbor in reversed(neighbors):
    await stack.put(neighbor)
```

### 3. Храните контекст вместе с данными

```python
# Вместо просто данных, храните кортеж (данные, контекст)
await stack.put((node, depth, path))
```

### 4. Используйте try/finally для task_done()

```python
async def safe_pop(stack):
    item = await stack.get()
    try:
        await process(item)
    finally:
        stack.task_done()
```

## Распространенные ошибки

### 1. Путаница с порядком элементов

```python
# Помните: последний добавленный извлекается первым!
await stack.put("first")
await stack.put("second")
item = await stack.get()  # "second", не "first"!
```

### 2. Использование LIFO вместо FIFO

```python
# Для обычной очереди задач используйте Queue, не LifoQueue
# LIFO подходит только для специфических алгоритмов
```

### 3. Бесконечный цикл при обходе графов

```python
# Всегда проверяйте посещенные узлы!
if node in visited:
    continue
visited.add(node)
```

## Сравнение типов очередей

| Характеристика | Queue (FIFO) | LifoQueue | PriorityQueue |
|----------------|--------------|-----------|---------------|
| Порядок извлечения | По порядку добавления | Обратный порядок | По приоритету |
| Типичное применение | Очередь задач | DFS, Undo | Задачи с приоритетами |
| Сложность операций | O(1) | O(1) | O(log n) |
| Аналог структуры данных | Очередь | Стек | Куча |

## Когда использовать LifoQueue

**Используйте LifoQueue когда:**
- Нужен обход в глубину (DFS)
- Реализуете функцию отмены (Undo)
- Обрабатываете вложенные/рекурсивные структуры
- Нужен backtracking алгоритм
- Последние данные должны обрабатываться первыми

**Не используйте LifoQueue когда:**
- Нужна обычная очередь задач (используйте Queue)
- Важен порядок добавления (используйте Queue)
- Нужна приоритизация (используйте PriorityQueue)

---

[prev: ./02-priority-queues.md](./02-priority-queues.md) | [next: ../13-subprocesses/readme.md](../13-subprocesses/readme.md)
