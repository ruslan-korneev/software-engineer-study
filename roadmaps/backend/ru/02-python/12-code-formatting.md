# Code Formatting (Форматирование кода)

## Зачем?

- Консистентность кода
- Меньше споров в команде
- Чистые git diffs

---

## 1. Ruff — современный стандарт

Линтер + форматтер на Rust. Заменяет flake8, isort, black.

```bash
pip install ruff

ruff check .          # Линтинг
ruff check --fix .    # Автоисправление
ruff format .         # Форматирование
```

### Конфигурация

```toml
[tool.ruff]
line-length = 88
target-version = "py311"

[tool.ruff.lint]
select = [
    "E",    # pycodestyle
    "F",    # pyflakes
    "I",    # isort
    "N",    # pep8-naming
    "UP",   # pyupgrade
    "B",    # flake8-bugbear
]

[tool.ruff.format]
quote-style = "double"
```

---

## 2. Black — форматтер

```bash
pip install black
black .
black --check .
```

```toml
[tool.black]
line-length = 88
target-version = ["py311"]
```

### Magic trailing comma

```python
# Запятая заставляет разбить на строки
data = {
    "name": "Анна",
}
```

---

## 3. isort — сортировка импортов

```bash
pip install isort
isort .
```

```toml
[tool.isort]
profile = "black"
```

```python
# Результат
import os                    # stdlib
from typing import Optional

import requests              # third-party

from myproject import utils  # first-party
```

---

## Pre-commit

```bash
pip install pre-commit
pre-commit install
```

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.1.6
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
```

---

## Сравнение

| Инструмент | Тип | Рекомендация |
|------------|-----|--------------|
| **Ruff** | Линтер + форматтер | Рекомендуется |
| Black | Форматтер | Хорошо |
| isort | Импорты | Ruff заменяет |
| Flake8 | Линтер | Ruff заменяет |

---

## Рекомендуемый сетап

```toml
# pyproject.toml
[tool.ruff]
line-length = 88

[tool.ruff.lint]
select = ["E", "F", "I", "N", "UP", "B"]

[tool.ruff.format]
quote-style = "double"
```

- **Ruff** — линтинг + форматирование
- **Pre-commit** — автоматизация перед коммитом
- **mypy** — проверка типов
