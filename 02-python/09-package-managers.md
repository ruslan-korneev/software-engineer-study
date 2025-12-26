# Package Managers (Менеджеры пакетов)

Инструменты для установки, обновления и удаления внешних библиотек.

---

## 1. pip — стандартный менеджер

Встроен в Python. Устанавливает пакеты из [PyPI](https://pypi.org/).

### Основные команды

```bash
# Установка
pip install requests
pip install requests==2.31.0      # Конкретная версия
pip install "requests>=2.28,<3"   # Диапазон версий

# Обновление
pip install --upgrade requests

# Удаление
pip uninstall requests

# Информация
pip list              # Все пакеты
pip show requests     # Инфо о пакете
```

### requirements.txt

```txt
requests==2.31.0
flask>=2.0
pytest
```

```bash
pip freeze > requirements.txt     # Создать
pip install -r requirements.txt   # Установить
```

---

## 2. Poetry — современный инструмент

Управляет зависимостями + виртуальными окружениями + сборкой пакетов.

### Установка

```bash
curl -sSL https://install.python-poetry.org | python3 -
# или
pipx install poetry
```

### Основные команды

```bash
poetry new myproject        # Новый проект
poetry init                 # Инициализировать в существующей папке

poetry add requests         # Добавить зависимость
poetry add pytest --group dev   # Dev-зависимость
poetry remove requests      # Удалить

poetry install              # Установить все
poetry run python main.py   # Запуск
poetry shell                # Активировать окружение
```

### pyproject.toml

```toml
[tool.poetry]
name = "myproject"
version = "0.1.0"

[tool.poetry.dependencies]
python = "^3.11"
requests = "^2.31.0"

[tool.poetry.group.dev.dependencies]
pytest = "^7.4.0"
```

### poetry.lock

Фиксирует точные версии всех зависимостей. **Коммить в git!**

---

## 3. uv — быстрый современный менеджер

Написан на Rust, в 10-100x быстрее pip.

### Установка

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### Основные команды

```bash
# Как pip (совместимый интерфейс)
uv pip install requests
uv pip install -r requirements.txt

# Как полноценный менеджер (с версии 0.4+)
uv init myproject           # Новый проект
uv add requests             # Добавить
uv add pytest --dev         # Dev-зависимость
uv sync                     # Синхронизировать
uv run python main.py       # Запуск
```

---

## Сравнение

| Инструмент | Скорость | Lock-файл | Venv | Сборка |
|------------|----------|-----------|------|--------|
| pip | Медленно | Нет | Нет | Нет |
| Poetry | Средне | Да | Да | Да |
| uv | Очень быстро | Да | Да | Да |

## Что выбрать?

| Ситуация | Рекомендация |
|----------|--------------|
| Простые скрипты | pip |
| Существующие проекты | Poetry (популярен) |
| Новые проекты | uv (современный стандарт) |

---

## Резюме

- **pip** — базовый, знать обязательно
- **Poetry** — популярен в индустрии, хороший DX
- **uv** — будущее Python, очень быстрый
- **pyproject.toml** — современный стандарт конфигурации (PEP 518)
- **Lock-файлы** — фиксируют версии, коммить в git
