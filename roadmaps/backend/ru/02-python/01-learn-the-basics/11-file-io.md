# Работа с файлами (File I/O)

## Основы работы с файлами

### Открытие и закрытие

```python
# Ручное закрытие (не рекомендуется)
file = open('example.txt', 'r')
content = file.read()
file.close()  # Легко забыть!

# Context manager (рекомендуется)
with open('example.txt', 'r') as file:
    content = file.read()
# Файл закрывается автоматически
```

### Режимы открытия

| Режим | Описание |
|-------|----------|
| `'r'` | Чтение (по умолчанию). Ошибка если файл не существует |
| `'w'` | Запись. Создаёт файл или перезаписывает существующий |
| `'a'` | Добавление в конец файла |
| `'x'` | Создание. Ошибка если файл существует |
| `'r+'` | Чтение и запись |
| `'w+'` | Чтение и запись (перезаписывает) |
| `'a+'` | Чтение и добавление |

### Бинарный режим

```python
# Добавьте 'b' для работы с бинарными файлами
with open('image.png', 'rb') as f:
    binary_data = f.read()

with open('output.bin', 'wb') as f:
    f.write(binary_data)
```

## Чтение файлов

### read() — весь файл

```python
with open('file.txt', 'r') as f:
    content = f.read()  # Весь файл как строка
    print(content)
```

### read(n) — n символов

```python
with open('file.txt', 'r') as f:
    chunk = f.read(100)  # Первые 100 символов
```

### readline() — одна строка

```python
with open('file.txt', 'r') as f:
    first_line = f.readline()   # Первая строка
    second_line = f.readline()  # Вторая строка
```

### readlines() — все строки как список

```python
with open('file.txt', 'r') as f:
    lines = f.readlines()  # ['line1\n', 'line2\n', ...]
```

### Итерация по строкам (рекомендуется)

```python
# Эффективно для больших файлов — не загружает всё в память
with open('file.txt', 'r') as f:
    for line in f:
        print(line.strip())  # strip() убирает \n
```

### Чтение с encoding

```python
# Явно указывайте кодировку!
with open('file.txt', 'r', encoding='utf-8') as f:
    content = f.read()

# Windows часто использует cp1251
with open('file.txt', 'r', encoding='cp1251') as f:
    content = f.read()
```

## Запись в файлы

### write() — запись строки

```python
with open('output.txt', 'w') as f:
    f.write('Hello, World!\n')
    f.write('Second line\n')
```

### writelines() — запись списка строк

```python
lines = ['Line 1\n', 'Line 2\n', 'Line 3\n']
with open('output.txt', 'w') as f:
    f.writelines(lines)  # Не добавляет \n автоматически!
```

### print() в файл

```python
with open('output.txt', 'w') as f:
    print('Hello', 'World', sep=', ', file=f)
    print('New line', file=f)
```

### Добавление в файл

```python
with open('log.txt', 'a') as f:
    f.write('New log entry\n')
```

## Позиция в файле

### tell() — текущая позиция

```python
with open('file.txt', 'r') as f:
    print(f.tell())    # 0
    f.read(10)
    print(f.tell())    # 10
```

### seek() — перемещение

```python
with open('file.txt', 'r') as f:
    f.seek(5)          # Перейти к 5-му байту
    content = f.read() # Читать с 5-го байта

# seek(offset, whence)
# whence: 0 = начало, 1 = текущая позиция, 2 = конец
with open('file.txt', 'rb') as f:
    f.seek(-10, 2)     # 10 байт до конца файла
    last_bytes = f.read()
```

## pathlib — современный подход

### Создание путей

```python
from pathlib import Path

# Текущая директория
current = Path('.')
home = Path.home()
cwd = Path.cwd()

# Создание пути
path = Path('/users/john/documents')
path = Path('folder') / 'subfolder' / 'file.txt'  # Оператор /
```

### Атрибуты пути

```python
path = Path('/home/user/documents/report.pdf')

path.name       # 'report.pdf'
path.stem       # 'report'
path.suffix     # '.pdf'
path.parent     # Path('/home/user/documents')
path.parts      # ('/', 'home', 'user', 'documents', 'report.pdf')
path.is_absolute()  # True
```

### Проверки

```python
path = Path('some/path')

path.exists()      # Существует ли
path.is_file()     # Это файл?
path.is_dir()      # Это директория?
path.is_symlink()  # Это символическая ссылка?
```

### Операции с файлами

```python
path = Path('file.txt')

# Чтение
content = path.read_text(encoding='utf-8')
data = path.read_bytes()

# Запись
path.write_text('Hello, World!', encoding='utf-8')
path.write_bytes(b'\x00\x01\x02')

# Удаление
path.unlink()  # Удалить файл
path.unlink(missing_ok=True)  # Не ошибка если не существует
```

### Операции с директориями

```python
dir_path = Path('new_folder')

# Создание
dir_path.mkdir()                    # Создать директорию
dir_path.mkdir(parents=True)        # Создать с родителями
dir_path.mkdir(exist_ok=True)       # Не ошибка если существует

# Удаление (только пустые!)
dir_path.rmdir()

# Поиск файлов
for file in Path('.').iterdir():    # Все элементы директории
    print(file)

for py_file in Path('.').glob('*.py'):      # Все .py файлы
    print(py_file)

for py_file in Path('.').rglob('*.py'):     # Рекурсивно
    print(py_file)
```

### Примеры с pathlib

```python
from pathlib import Path

# Найти все Python файлы и посчитать строки
total_lines = 0
for py_file in Path('src').rglob('*.py'):
    lines = py_file.read_text().splitlines()
    total_lines += len(lines)
    print(f'{py_file}: {len(lines)} lines')

print(f'Total: {total_lines}')
```

## Работа с JSON

### Чтение JSON

```python
import json

# Из файла
with open('data.json', 'r', encoding='utf-8') as f:
    data = json.load(f)

# Из строки
json_string = '{"name": "Alice", "age": 30}'
data = json.loads(json_string)
```

### Запись JSON

```python
import json

data = {'name': 'Alice', 'age': 30, 'hobbies': ['reading', 'coding']}

# В файл
with open('data.json', 'w', encoding='utf-8') as f:
    json.dump(data, f, indent=2, ensure_ascii=False)

# В строку
json_string = json.dumps(data, indent=2, ensure_ascii=False)
```

### Параметры форматирования

```python
json.dump(data, f,
    indent=2,              # Отступы для читаемости
    ensure_ascii=False,    # Сохранять Unicode (кириллицу)
    sort_keys=True,        # Сортировать ключи
    default=str            # Как сериализовать неизвестные типы
)
```

## Работа с CSV

### Чтение CSV

```python
import csv

# Как список списков
with open('data.csv', 'r', encoding='utf-8') as f:
    reader = csv.reader(f)
    for row in reader:
        print(row)  # ['value1', 'value2', ...]

# Как словари (с заголовками)
with open('data.csv', 'r', encoding='utf-8') as f:
    reader = csv.DictReader(f)
    for row in reader:
        print(row)  # {'column1': 'value1', 'column2': 'value2'}
```

### Запись CSV

```python
import csv

# Из списков
data = [['name', 'age'], ['Alice', 30], ['Bob', 25]]
with open('output.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.writer(f)
    writer.writerows(data)

# Из словарей
data = [{'name': 'Alice', 'age': 30}, {'name': 'Bob', 'age': 25}]
with open('output.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.DictWriter(f, fieldnames=['name', 'age'])
    writer.writeheader()
    writer.writerows(data)
```

### Разделители и кавычки

```python
# Точка с запятой вместо запятой
reader = csv.reader(f, delimiter=';')
writer = csv.writer(f, delimiter=';')

# Кавычки для всех полей
writer = csv.writer(f, quoting=csv.QUOTE_ALL)
```

## Временные файлы

```python
import tempfile

# Временный файл (автоудаление)
with tempfile.NamedTemporaryFile(mode='w', delete=False) as f:
    f.write('temporary data')
    print(f.name)  # Путь к временному файлу

# Временная директория
with tempfile.TemporaryDirectory() as tmpdir:
    print(tmpdir)  # Путь к временной директории
    # Директория удалится после выхода из with
```

## Обработка ошибок

```python
from pathlib import Path

path = Path('file.txt')

try:
    content = path.read_text()
except FileNotFoundError:
    print('Файл не найден')
except PermissionError:
    print('Нет доступа к файлу')
except UnicodeDecodeError:
    print('Ошибка кодировки')
except IOError as e:
    print(f'Ошибка ввода-вывода: {e}')
```

## Best Practices

### 1. Всегда используйте context manager

```python
# ✅ Правильно
with open('file.txt') as f:
    data = f.read()

# ❌ Неправильно
f = open('file.txt')
data = f.read()
f.close()
```

### 2. Указывайте encoding

```python
# ✅ Явная кодировка
with open('file.txt', encoding='utf-8') as f:
    ...

# ❌ Зависит от системы
with open('file.txt') as f:
    ...
```

### 3. Используйте pathlib для путей

```python
# ✅ pathlib
from pathlib import Path
path = Path('folder') / 'file.txt'

# ❌ Конкатенация строк
path = 'folder' + '/' + 'file.txt'
```

### 4. Обрабатывайте большие файлы построчно

```python
# ✅ Построчно — O(1) памяти
with open('large.txt') as f:
    for line in f:
        process(line)

# ❌ Весь файл — O(n) памяти
with open('large.txt') as f:
    lines = f.readlines()  # Всё в память!
```

## Q&A

**Q: Чем pathlib лучше os.path?**
A: pathlib — объектно-ориентированный API, более читаемый и удобный. Оператор `/` для путей, методы read_text/write_text.

**Q: Когда использовать бинарный режим?**
A: Для не-текстовых файлов: изображения, аудио, архивы, исполняемые файлы. Также для точного копирования файлов.

**Q: Как читать файл в обратном порядке?**
A: Для небольших файлов: `reversed(path.read_text().splitlines())`. Для больших — специальные библиотеки.

**Q: Почему newline='' при записи CSV?**
A: Модуль csv сам управляет переносами строк. Без `newline=''` могут появиться лишние пустые строки на Windows.
