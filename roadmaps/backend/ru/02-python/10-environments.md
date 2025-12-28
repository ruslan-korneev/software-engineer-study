# Environments (Виртуальные окружения)

[prev: 09-package-managers](./09-package-managers.md) | [next: 10a-pyproject-toml](./10a-pyproject-toml.md)
---

## Зачем нужны?

Изоляция зависимостей между проектами:

```
Проект A: requests==2.25.0
Проект B: requests==2.31.0
```

**Виртуальное окружение** = изолированная копия Python с отдельными пакетами.

---

## 1. venv — встроенный модуль

```bash
# Создать
python -m venv .venv

# Активировать
source .venv/bin/activate      # Linux/macOS
.venv\Scripts\activate         # Windows

# Установка пакетов (в .venv)
pip install requests

# Деактивировать
deactivate
```

### Структура

```
.venv/
├── bin/           # python, pip
├── lib/           # Пакеты
└── pyvenv.cfg
```

### .gitignore

```gitignore
.venv/
```

**Не коммить!** Только `requirements.txt` / `pyproject.toml`.

---

## 2. Poetry — автоматическое управление

```bash
poetry install              # Создаёт .venv + устанавливает
poetry shell                # Активирует
poetry run python main.py   # Запуск без активации
```

Создавать .venv в папке проекта:

```bash
poetry config virtualenvs.in-project true
```

---

## 3. uv — быстрое создание

```bash
uv venv                     # Создать (мгновенно)
source .venv/bin/activate
uv pip install requests

# Или без активации
uv run python main.py
```

---

## 4. conda — для Data Science

Управляет Python-пакетами + системными библиотеками.

```bash
conda create -n myenv python=3.11
conda activate myenv
conda install numpy pandas
conda env list
conda env remove -n myenv
```

---

## 5. pyenv — управление версиями Python

Менеджер версий Python (не виртуальных окружений).

```bash
# Установить версию Python
pyenv install 3.12.0

# Глобально
pyenv global 3.12.0

# Для текущей папки
pyenv local 3.11.0   # Создаёт .python-version
```

### pyenv + venv

```bash
pyenv install 3.12.0
pyenv local 3.12.0
python -m venv .venv
```

---

## Сравнение

| Инструмент | Назначение |
|------------|-----------|
| venv | Виртуальные окружения (стандарт) |
| Poetry | Venv + зависимости |
| uv | Быстрый venv + pip |
| conda | Venv + системные пакеты (DS) |
| pyenv | Управление версиями Python |

---

## Резюме

- **Всегда используй виртуальные окружения** для проектов
- **venv** — базовый, встроен в Python
- **Poetry/uv** — автоматизируют создание и управление
- **.venv/** — не коммить в git
- **pyenv** — когда нужны разные версии Python

---
[prev: 09-package-managers](./09-package-managers.md) | [next: 10a-pyproject-toml](./10a-pyproject-toml.md)