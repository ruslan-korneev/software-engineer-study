# pyproject.toml (Конфигурация проекта)

[prev: 10-environments](./10-environments.md) | [next: 10b-common-packages](./10b-common-packages.md)
---

Стандартный файл конфигурации Python-проекта (PEP 518, 621).

---

## Базовая структура

```toml
[project]
name = "myproject"
version = "0.1.0"
description = "Описание"
readme = "README.md"
requires-python = ">=3.11"
license = {text = "MIT"}
authors = [{name = "Name", email = "email@example.com"}]

dependencies = [
    "requests>=2.28",
    "pydantic>=2.0",
]

[project.optional-dependencies]
dev = ["pytest>=7.0", "ruff>=0.1"]

[project.scripts]
myapp = "myproject.cli:main"

[build-system]
requires = ["setuptools>=61.0"]
build-backend = "setuptools.build_meta"
```

---

## Секции

### `[project]` — метаданные

```toml
[project]
name = "myproject"
version = "1.2.3"
requires-python = ">=3.11"
```

### `dependencies` — зависимости

```toml
dependencies = [
    "requests",           # Любая версия
    "flask>=2.0",         # Минимум 2.0
    "django>=4.0,<5.0",   # Диапазон
    "numpy~=1.24.0",      # Совместимые (1.24.x)
    "pydantic==2.5.2",    # Точная
]
```

### `[project.optional-dependencies]`

```toml
[project.optional-dependencies]
dev = ["pytest", "ruff"]
docs = ["sphinx"]
```

```bash
pip install myproject[dev]
```

### `[project.scripts]` — CLI

```toml
[project.scripts]
myapp = "myproject.cli:main"
```

---

## Конфигурация инструментов

### Ruff

```toml
[tool.ruff]
line-length = 88

[tool.ruff.lint]
select = ["E", "F", "I"]
```

### Black

```toml
[tool.black]
line-length = 88
target-version = ["py311"]
```

### mypy

```toml
[tool.mypy]
python_version = "3.11"
strict = true
```

### pytest

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-v"
```

---

## Build Systems

```toml
# setuptools
[build-system]
requires = ["setuptools>=61.0"]
build-backend = "setuptools.build_meta"

# poetry
[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"

# hatch
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

---

## Резюме

| Секция | Назначение |
|--------|-----------|
| `[project]` | Метаданные пакета |
| `dependencies` | Основные зависимости |
| `[project.optional-dependencies]` | Dev/test зависимости |
| `[project.scripts]` | CLI entry points |
| `[build-system]` | Система сборки |
| `[tool.*]` | Конфиги инструментов |

- Один файл вместо setup.py, setup.cfg, requirements.txt
- Стандарт PEP 518/621
- Все инструменты в одном месте

---
[prev: 10-environments](./10-environments.md) | [next: 10b-common-packages](./10b-common-packages.md)