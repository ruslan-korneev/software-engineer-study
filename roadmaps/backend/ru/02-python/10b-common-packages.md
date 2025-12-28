# Common Packages (Популярные пакеты)

[prev: 10a-pyproject-toml](./10a-pyproject-toml.md) | [next: 10c-pattern-matching](./10c-pattern-matching.md)
---

## Встроенные модули

### `json`

```python
import json

data = {"name": "Анна", "age": 25}
json_str = json.dumps(data, ensure_ascii=False)  # → JSON
parsed = json.loads(json_str)                     # → Python

# Файлы
json.dump(data, open("data.json", "w"))
data = json.load(open("data.json"))
```

### `datetime`

```python
from datetime import datetime, date, timedelta

now = datetime.now()
today = date.today()

now.strftime("%Y-%m-%d %H:%M")  # Форматирование
datetime.strptime("2024-12-25", "%Y-%m-%d")  # Парсинг

tomorrow = today + timedelta(days=1)
```

### `pathlib`

```python
from pathlib import Path

p = Path.cwd() / "src" / "main.py"
p.exists()
p.read_text()
p.write_text("content")
list(Path(".").glob("*.py"))
```

### `collections`

```python
from collections import Counter, defaultdict, deque

Counter("abracadabra").most_common(2)  # [('a', 5), ('b', 2)]

d = defaultdict(list)
d["key"].append(1)

q = deque([1, 2, 3])
q.appendleft(0)
```

### `functools`

```python
from functools import lru_cache, partial

@lru_cache
def fib(n): ...

square = partial(pow, exp=2)
```

---

## Внешние пакеты (pip install)

### `requests` — HTTP

```python
import requests

resp = requests.get("https://api.example.com/users")
resp.json()

resp = requests.post(url, json={"name": "Анна"})
resp.raise_for_status()
```

### `httpx` — async HTTP

```python
import httpx

async with httpx.AsyncClient() as client:
    resp = await client.get(url)
```

### `pydantic` — валидация

```python
from pydantic import BaseModel

class User(BaseModel):
    name: str
    age: int

user = User(name="Анна", age=25)
user.model_dump()  # → dict
```

### `python-dotenv` — .env файлы

```python
from dotenv import load_dotenv
import os

load_dotenv()
api_key = os.getenv("API_KEY")
```

### `loguru` — логирование

```python
from loguru import logger

logger.info("User {name} logged in", name="Анна")
logger.add("app.log", rotation="10 MB")
```

### `click` — CLI

```python
import click

@click.command()
@click.option("--name", default="World")
def hello(name):
    click.echo(f"Hello, {name}!")
```

---

## Резюме

| Задача | Встроенный | Внешний |
|--------|-----------|---------|
| JSON | `json` | — |
| Даты | `datetime` | `pendulum`, `arrow` |
| Пути | `pathlib` | — |
| HTTP | `urllib` | `requests`, `httpx` |
| Валидация | — | `pydantic` |
| Логирование | `logging` | `loguru` |
| CLI | `argparse` | `click`, `typer` |
| Env | `os.environ` | `python-dotenv` |

---
[prev: 10a-pyproject-toml](./10a-pyproject-toml.md) | [next: 10c-pattern-matching](./10c-pattern-matching.md)