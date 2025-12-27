# Логирование (Logging)

## Зачем логирование?

**Логирование** — запись событий в программе для отладки, мониторинга и аудита. В отличие от `print()`, логирование:

- Имеет уровни важности
- Легко включается/выключается
- Направляется в разные места (файл, консоль, сеть)
- Содержит метаданные (время, модуль, строка)

```python
# ❌ print — примитивно
print("User logged in")
print("Error: connection failed")

# ✅ logging — профессионально
import logging

logging.info("User logged in")
logging.error("Connection failed")
```

## Быстрый старт

```python
import logging

# Базовая настройка
logging.basicConfig(level=logging.DEBUG)

# Использование
logging.debug("Debug message")      # Отладка
logging.info("Info message")        # Информация
logging.warning("Warning message")  # Предупреждение
logging.error("Error message")      # Ошибка
logging.critical("Critical message") # Критическая ошибка
```

## Уровни логирования

| Уровень | Числовое значение | Когда использовать |
|---------|-------------------|-------------------|
| DEBUG | 10 | Детальная информация для отладки |
| INFO | 20 | Подтверждение нормальной работы |
| WARNING | 30 | Что-то неожиданное, но работа продолжается |
| ERROR | 40 | Ошибка, функциональность нарушена |
| CRITICAL | 50 | Серьёзная ошибка, приложение может упасть |

```python
import logging

logging.basicConfig(level=logging.WARNING)

logging.debug("Not shown")    # Ниже уровня — не выводится
logging.info("Not shown")     # Ниже уровня — не выводится
logging.warning("Shown")      # Выводится
logging.error("Shown")        # Выводится
```

## basicConfig — простая настройка

```python
import logging

logging.basicConfig(
    level=logging.DEBUG,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    datefmt='%Y-%m-%d %H:%M:%S',
    filename='app.log',      # Если указан — пишет в файл
    filemode='w'             # 'w' — перезаписывать, 'a' — добавлять
)
```

### Форматирование

```python
# Популярные атрибуты форматирования
format_string = (
    '%(asctime)s '           # Время
    '%(name)s '              # Имя логгера
    '%(levelname)s '         # Уровень (DEBUG, INFO, ...)
    '%(filename)s:'          # Имя файла
    '%(lineno)d '            # Номер строки
    '%(funcName)s '          # Имя функции
    '%(message)s'            # Сообщение
)
```

## Loggers — именованные логгеры

```python
import logging

# Создание именованного логгера
logger = logging.getLogger(__name__)

# __name__ даёт имя модуля: 'myapp.utils'
# Это создаёт иерархию: myapp.utils наследует от myapp

def process_data(data):
    logger.info(f"Processing {len(data)} items")
    try:
        # ... обработка ...
        logger.debug("Processing complete")
    except Exception as e:
        logger.error(f"Processing failed: {e}")
```

### Иерархия логгеров

```python
# Иерархия: root -> myapp -> myapp.utils -> myapp.utils.parser
#
# Настройки наследуются вниз по иерархии

root_logger = logging.getLogger()           # root
app_logger = logging.getLogger('myapp')     # myapp
utils_logger = logging.getLogger('myapp.utils')  # myapp.utils

# Настроив myapp, влияете на все дочерние логгеры
app_logger.setLevel(logging.DEBUG)
```

## Handlers — куда выводить

```python
import logging

logger = logging.getLogger('myapp')
logger.setLevel(logging.DEBUG)

# Консольный handler
console_handler = logging.StreamHandler()
console_handler.setLevel(logging.INFO)  # В консоль — только INFO+

# Файловый handler
file_handler = logging.FileHandler('app.log')
file_handler.setLevel(logging.DEBUG)  # В файл — всё включая DEBUG

# Добавляем handlers к логгеру
logger.addHandler(console_handler)
logger.addHandler(file_handler)

# Теперь логи идут и в консоль, и в файл
logger.debug("Only in file")
logger.info("In both console and file")
```

### Типы handlers

```python
import logging
from logging.handlers import (
    RotatingFileHandler,    # Ротация по размеру
    TimedRotatingFileHandler,  # Ротация по времени
    SMTPHandler,            # Email
    HTTPHandler,            # HTTP
    SysLogHandler,          # Syslog
)

# Ротация по размеру (5 MB, 3 бэкапа)
handler = RotatingFileHandler(
    'app.log',
    maxBytes=5*1024*1024,
    backupCount=3
)

# Ротация по времени (ежедневно)
handler = TimedRotatingFileHandler(
    'app.log',
    when='midnight',
    interval=1,
    backupCount=7
)
```

## Formatters — как форматировать

```python
import logging

logger = logging.getLogger('myapp')

# Разные форматы для разных handlers
console_formatter = logging.Formatter('%(levelname)s: %(message)s')
file_formatter = logging.Formatter(
    '%(asctime)s - %(name)s - %(levelname)s - %(filename)s:%(lineno)d - %(message)s'
)

console_handler = logging.StreamHandler()
console_handler.setFormatter(console_formatter)

file_handler = logging.FileHandler('app.log')
file_handler.setFormatter(file_formatter)

logger.addHandler(console_handler)
logger.addHandler(file_handler)
```

### JSON Formatter

```python
import logging
import json
from datetime import datetime

class JSONFormatter(logging.Formatter):
    def format(self, record):
        log_data = {
            'timestamp': datetime.utcnow().isoformat(),
            'level': record.levelname,
            'logger': record.name,
            'message': record.getMessage(),
            'module': record.module,
            'line': record.lineno,
        }
        if record.exc_info:
            log_data['exception'] = self.formatException(record.exc_info)
        return json.dumps(log_data)

handler = logging.StreamHandler()
handler.setFormatter(JSONFormatter())
```

## Логирование исключений

```python
import logging

logger = logging.getLogger(__name__)

try:
    result = 1 / 0
except ZeroDivisionError:
    # exc_info=True добавляет traceback
    logger.error("Division failed", exc_info=True)

    # Или используйте logger.exception() — то же самое
    logger.exception("Division failed")
```

Вывод:
```
ERROR:__main__:Division failed
Traceback (most recent call last):
  File "example.py", line 6, in <module>
    result = 1 / 0
ZeroDivisionError: division by zero
```

## Конфигурация через словарь

```python
import logging.config

LOGGING_CONFIG = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'standard': {
            'format': '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
        },
        'detailed': {
            'format': '%(asctime)s - %(name)s - %(levelname)s - %(filename)s:%(lineno)d - %(message)s'
        },
    },
    'handlers': {
        'console': {
            'class': 'logging.StreamHandler',
            'level': 'INFO',
            'formatter': 'standard',
            'stream': 'ext://sys.stdout',
        },
        'file': {
            'class': 'logging.handlers.RotatingFileHandler',
            'level': 'DEBUG',
            'formatter': 'detailed',
            'filename': 'app.log',
            'maxBytes': 10485760,  # 10 MB
            'backupCount': 5,
        },
    },
    'loggers': {
        '': {  # root logger
            'handlers': ['console', 'file'],
            'level': 'DEBUG',
        },
        'myapp': {
            'handlers': ['console', 'file'],
            'level': 'DEBUG',
            'propagate': False,
        },
    },
}

logging.config.dictConfig(LOGGING_CONFIG)
```

## Structured Logging (structlog)

Для продвинутого логирования используйте `structlog`:

```python
# pip install structlog
import structlog

structlog.configure(
    processors=[
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer()
    ]
)

logger = structlog.get_logger()

# Structured logging с контекстом
logger.info("user_login", user_id=123, ip_address="192.168.1.1")
# {"user_id": 123, "ip_address": "192.168.1.1", "event": "user_login", "timestamp": "..."}
```

## Best Practices

### 1. Используйте __name__ для имени логгера

```python
# ✅ Автоматическая иерархия по модулям
logger = logging.getLogger(__name__)

# ❌ Жёстко закодированное имя
logger = logging.getLogger('my_logger')
```

### 2. Не логируйте чувствительные данные

```python
# ❌ Пароли, токены, персональные данные
logger.info(f"User login: {username}, password: {password}")

# ✅ Только безопасная информация
logger.info(f"User login: {username}")
```

### 3. Используйте lazy formatting

```python
# ❌ Форматирование всегда происходит
logger.debug(f"Processing data: {expensive_operation()}")

# ✅ Форматирование только если уровень подходит
logger.debug("Processing data: %s", expensive_operation())
```

### 4. Логируйте на правильном уровне

```python
# DEBUG — детали для разработчиков
logger.debug(f"Cache hit for key: {key}")

# INFO — значимые бизнес-события
logger.info(f"Order {order_id} created")

# WARNING — потенциальные проблемы
logger.warning(f"API rate limit: {remaining} requests left")

# ERROR — ошибки, требующие внимания
logger.error(f"Payment failed for order {order_id}")

# CRITICAL — система в опасности
logger.critical("Database connection lost")
```

### 5. Добавляйте контекст

```python
# ❌ Мало информации
logger.error("Request failed")

# ✅ Достаточно контекста
logger.error(
    "Request failed",
    extra={
        'url': url,
        'status_code': response.status_code,
        'request_id': request_id
    }
)
```

## Конфигурация для разных окружений

```python
import logging
import os

def setup_logging():
    level = logging.DEBUG if os.getenv('DEBUG') else logging.INFO

    if os.getenv('ENVIRONMENT') == 'production':
        # Production: JSON в файл
        handler = logging.FileHandler('app.log')
        handler.setFormatter(JSONFormatter())
    else:
        # Development: читаемый формат в консоль
        handler = logging.StreamHandler()
        handler.setFormatter(logging.Formatter(
            '%(asctime)s - %(levelname)s - %(message)s'
        ))

    logging.basicConfig(level=level, handlers=[handler])
```

## Интеграция с фреймворками

### FastAPI

```python
import logging
from fastapi import FastAPI, Request

logger = logging.getLogger(__name__)
app = FastAPI()

@app.middleware("http")
async def log_requests(request: Request, call_next):
    logger.info(f"Request: {request.method} {request.url}")
    response = await call_next(request)
    logger.info(f"Response: {response.status_code}")
    return response
```

### Django

```python
# settings.py
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'handlers': {
        'console': {
            'class': 'logging.StreamHandler',
        },
    },
    'root': {
        'handlers': ['console'],
        'level': 'WARNING',
    },
    'loggers': {
        'django': {
            'handlers': ['console'],
            'level': 'INFO',
            'propagate': False,
        },
    },
}
```

## Q&A

**Q: Почему не использовать print()?**
A: logging имеет уровни, форматирование, handlers, и легко отключается. print() не годится для продакшена.

**Q: Как логировать в несколько файлов?**
A: Добавьте несколько FileHandler с разными путями и уровнями.

**Q: Логирование замедляет приложение?**
A: Минимально. Используйте lazy formatting и асинхронные handlers для высоконагруженных систем.

**Q: Как ротировать логи?**
A: Используйте RotatingFileHandler или TimedRotatingFileHandler.

**Q: structlog vs logging?**
A: structlog — для structured logging (JSON). Стандартный logging — для простых случаев.
