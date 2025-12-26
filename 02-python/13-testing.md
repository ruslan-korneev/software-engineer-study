# Testing (Тестирование)

## pytest — стандарт

```bash
pip install pytest
pytest              # Запустить все
pytest -v           # Verbose
pytest -x           # Стоп на первой ошибке
pytest -k "user"    # Фильтр по имени
```

---

## Базовые тесты

```python
import pytest
from myproject import add, divide

def test_add():
    assert add(2, 3) == 5

def test_divide_by_zero():
    with pytest.raises(ValueError, match="divide by zero"):
        divide(10, 0)
```

---

## Фикстуры

```python
@pytest.fixture
def user():
    return {"name": "Анна", "age": 25}

def test_user_name(user):
    assert user["name"] == "Анна"
```

### Scope

```python
@pytest.fixture(scope="function")  # Каждый тест (default)
@pytest.fixture(scope="module")    # Один раз на файл
@pytest.fixture(scope="session")   # Один раз на запуск
```

### Setup / Teardown

```python
@pytest.fixture
def db():
    conn = create_connection()  # Setup
    yield conn
    conn.close()                # Teardown
```

### conftest.py

```python
# tests/conftest.py — общие фикстуры
@pytest.fixture
def client():
    return TestClient(app)
```

---

## Параметризация

```python
@pytest.mark.parametrize("a, b, expected", [
    (2, 3, 5),
    (0, 0, 0),
    (-1, 1, 0),
])
def test_add(a, b, expected):
    assert add(a, b) == expected
```

---

## Маркеры

```python
@pytest.mark.slow
def test_heavy(): ...

@pytest.mark.skip(reason="Not implemented")
def test_future(): ...

@pytest.mark.skipif(sys.platform == "win32", reason="Unix only")
def test_unix(): ...

@pytest.mark.xfail(reason="Known bug")
def test_bug(): ...
```

```bash
pytest -m "not slow"
```

---

## Моки

```python
from unittest.mock import patch, Mock

@patch("mymodule.requests.get")
def test_api(mock_get):
    mock_get.return_value.json.return_value = {"name": "Анна"}

    result = get_user()

    assert result["name"] == "Анна"
    mock_get.assert_called_once()
```

### pytest-mock

```python
def test_api(mocker):
    mock = mocker.patch("requests.get")
    mock.return_value.json.return_value = {"data": 1}
```

---

## Async тесты

```bash
pip install pytest-asyncio
```

```python
@pytest.mark.asyncio
async def test_async():
    result = await fetch_data()
    assert result == expected
```

---

## Coverage

```bash
pip install pytest-cov
pytest --cov=myproject --cov-report=html
```

---

## Конфигурация

```toml
# pyproject.toml
[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-v --tb=short"

[tool.coverage.run]
source = ["src/myproject"]

[tool.coverage.report]
fail_under = 80
```

---

## Резюме

| Концепция | Описание |
|-----------|----------|
| `assert` | Проверка условия |
| `pytest.raises` | Проверка исключений |
| `@pytest.fixture` | Подготовка данных |
| `@pytest.mark.parametrize` | Много входных данных |
| `@patch` / `mocker` | Подмена зависимостей |
| `pytest-cov` | Покрытие кода |
