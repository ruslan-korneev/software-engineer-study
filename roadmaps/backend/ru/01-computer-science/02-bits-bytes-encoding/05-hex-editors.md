# Hex Editors (Hex-редакторы и работа с бинарными данными)

[prev: 04-character-encoding](./04-character-encoding.md) | [next: 01-what-is-os](../03-operating-systems/01-what-is-os.md)

---

## Определение

**Hex-редактор** — это программа для просмотра и редактирования файлов на уровне отдельных байтов. Данные отображаются в шестнадцатеричном формате, что делает удобным анализ бинарных файлов, исполняемых программ, сетевых дампов и любых других данных.

## Зачем нужны hex-редакторы?

1. **Анализ форматов файлов** — изучение структуры бинарных форматов
2. **Reverse engineering** — анализ исполняемых файлов и протоколов
3. **Восстановление данных** — ремонт повреждённых файлов
4. **Отладка** — поиск проблем в бинарных данных
5. **Криминалистика** — анализ файлов при расследованиях
6. **Патчинг** — изменение поведения программ

## Интерфейс hex-редактора

Типичный вид:

```
Адрес    | Hex-данные                           | ASCII
---------|--------------------------------------|--------
00000000 | 89 50 4E 47 0D 0A 1A 0A 00 00 00 0D | .PNG........
00000010 | 49 48 44 52 00 00 01 00 00 00 01 00 | IHDR........
00000020 | 08 06 00 00 00 5C 72 A8 66 00 00 00 | .....\r.f...
```

| Колонка | Описание |
|---------|----------|
| Адрес (offset) | Позиция в файле (обычно в hex) |
| Hex-данные | Байты в шестнадцатеричном виде |
| ASCII | Текстовое представление (. для непечатаемых) |

## Популярные hex-редакторы

### Консольные

| Редактор | Платформа | Особенности |
|----------|-----------|-------------|
| `xxd` | Linux/macOS | Входит в vim, конвертирует в hex и обратно |
| `hexdump` | Linux/macOS | Встроенная утилита Unix |
| `hexyl` | Кроссплатформенный | Цветной вывод, написан на Rust |
| `od` | Linux/macOS | Классическая утилита Unix |

### GUI

| Редактор | Платформа | Особенности |
|----------|-----------|-------------|
| **HxD** | Windows | Бесплатный, быстрый, популярный |
| **010 Editor** | Кроссплатформенный | Шаблоны для разбора форматов |
| **Hex Fiend** | macOS | Бесплатный, открытый код |
| **ImHex** | Кроссплатформенный | Современный, паттерн-язык |
| **Bless** | Linux | GTK-интерфейс |

### Плагины для IDE

- **VS Code**: Hex Editor (официальный)
- **JetBrains**: BinEd plugin
- **Vim**: встроенный `xxd`

## Консольные инструменты

### xxd

```bash
# Просмотр файла в hex
xxd file.bin

# Вывод:
# 00000000: 8950 4e47 0d0a 1a0a 0000 000d 4948 4452  .PNG........IHDR

# Ограничить вывод
xxd -l 32 file.bin      # Первые 32 байта
xxd -s 100 file.bin     # Начиная с позиции 100

# Только hex (без ASCII)
xxd -p file.bin

# Обратное преобразование (hex → binary)
xxd -r hexdump.txt > file.bin

# Редактирование в vim
vim file.bin
# В vim: :%!xxd
# Редактируем hex
# В vim: :%!xxd -r
# :w
```

### hexdump

```bash
# Канонический формат (похож на hex-редактор)
hexdump -C file.bin

# Вывод:
# 00000000  89 50 4e 47 0d 0a 1a 0a  00 00 00 0d 49 48 44 52  |.PNG........IHDR|

# Только первые 64 байта
hexdump -C -n 64 file.bin

# Начиная с позиции 100
hexdump -C -s 100 file.bin

# Форматированный вывод
hexdump -e '16/1 "%02x " "\n"' file.bin
```

### hexyl (современная альтернатива)

```bash
# Установка
cargo install hexyl  # или brew install hexyl

# Использование
hexyl file.bin
hexyl --length 256 file.bin
hexyl --skip 100 file.bin

# Цветной вывод:
# - NULL байты подсвечены одним цветом
# - ASCII — другим
# - Непечатаемые — третьим
```

## Работа с бинарными данными в Python

### Чтение и запись байтов

```python
# Чтение бинарного файла
with open('file.bin', 'rb') as f:
    data = f.read()
    print(f"Размер: {len(data)} байт")
    print(f"Первые 16 байт: {data[:16].hex()}")

# Запись бинарного файла
with open('output.bin', 'wb') as f:
    f.write(b'\x89PNG\r\n\x1a\n')  # Сигнатура PNG
    f.write(bytes([0x00, 0x00, 0x00, 0x0D]))  # Длина чанка

# Позиционирование в файле
with open('file.bin', 'rb') as f:
    f.seek(100)  # Перейти к позиции 100
    chunk = f.read(32)  # Прочитать 32 байта
    print(f"Позиция: {f.tell()}")  # 132
```

### Hex-дамп на Python

```python
def hexdump(data, width=16):
    """Создание hex-дампа как в hex-редакторе"""
    result = []
    for i in range(0, len(data), width):
        chunk = data[i:i+width]

        # Адрес
        addr = f"{i:08x}"

        # Hex-представление
        hex_part = ' '.join(f"{b:02x}" for b in chunk)
        hex_part = hex_part.ljust(width * 3 - 1)

        # ASCII-представление
        ascii_part = ''.join(
            chr(b) if 32 <= b < 127 else '.'
            for b in chunk
        )

        result.append(f"{addr}  {hex_part}  |{ascii_part}|")

    return '\n'.join(result)

# Использование
with open('file.bin', 'rb') as f:
    print(hexdump(f.read(256)))
```

### Поиск паттернов в бинарных данных

```python
def find_pattern(data, pattern):
    """Найти все вхождения паттерна"""
    positions = []
    start = 0
    while True:
        pos = data.find(pattern, start)
        if pos == -1:
            break
        positions.append(pos)
        start = pos + 1
    return positions

# Поиск строки
with open('file.bin', 'rb') as f:
    data = f.read()

# Найти все вхождения 'PNG'
positions = find_pattern(data, b'PNG')
print(f"Найдено {len(positions)} вхождений: {positions}")

# Поиск hex-паттерна
magic = bytes.fromhex('89504e47')  # PNG signature
pos = data.find(magic)
if pos != -1:
    print(f"PNG найден на позиции {pos} (0x{pos:x})")
```

### Модификация бинарных данных

```python
# Патчинг файла
with open('file.bin', 'rb') as f:
    data = bytearray(f.read())

# Изменение байта по адресу
data[0x100] = 0xFF

# Замена последовательности
data[0x200:0x204] = b'\xDE\xAD\xBE\xEF'

# Сохранение
with open('patched.bin', 'wb') as f:
    f.write(data)
```

## Анализ форматов файлов

### Сигнатуры файлов (Magic Numbers)

```python
FILE_SIGNATURES = {
    b'\x89PNG\r\n\x1a\n': 'PNG image',
    b'\xff\xd8\xff': 'JPEG image',
    b'GIF87a': 'GIF image (87a)',
    b'GIF89a': 'GIF image (89a)',
    b'PK\x03\x04': 'ZIP archive',
    b'%PDF': 'PDF document',
    b'\x7fELF': 'ELF executable',
    b'MZ': 'DOS/Windows executable',
    b'\xca\xfe\xba\xbe': 'Java class / Mach-O fat binary',
    b'\xfe\xed\xfa\xce': 'Mach-O 32-bit',
    b'\xfe\xed\xfa\xcf': 'Mach-O 64-bit',
    b'SQLite format 3': 'SQLite database',
}

def identify_file(filepath):
    """Определить тип файла по сигнатуре"""
    with open(filepath, 'rb') as f:
        header = f.read(16)

    for signature, file_type in FILE_SIGNATURES.items():
        if header.startswith(signature):
            return file_type

    return 'Unknown'

# Пример
print(identify_file('image.png'))  # 'PNG image'
```

### Разбор структуры файла (на примере PNG)

```python
import struct

def parse_png(filepath):
    """Разбор структуры PNG файла"""
    with open(filepath, 'rb') as f:
        # Проверка сигнатуры
        signature = f.read(8)
        if signature != b'\x89PNG\r\n\x1a\n':
            raise ValueError("Not a PNG file")

        chunks = []
        while True:
            # Чтение заголовка чанка
            chunk_header = f.read(8)
            if len(chunk_header) < 8:
                break

            length, chunk_type = struct.unpack('>I4s', chunk_header)
            chunk_type = chunk_type.decode('ascii')

            # Чтение данных и CRC
            data = f.read(length)
            crc = f.read(4)

            chunks.append({
                'type': chunk_type,
                'length': length,
                'offset': f.tell() - length - 12
            })

            if chunk_type == 'IEND':
                break

        return chunks

# Пример использования
for chunk in parse_png('image.png'):
    print(f"{chunk['type']}: {chunk['length']} bytes at 0x{chunk['offset']:x}")
```

## Модуль struct

Модуль `struct` используется для работы с бинарными данными:

```python
import struct

# Форматы
# >  Big Endian
# <  Little Endian
# =  Native byte order
# B  unsigned char (1 byte)
# H  unsigned short (2 bytes)
# I  unsigned int (4 bytes)
# Q  unsigned long long (8 bytes)
# s  bytes (строка)

# Упаковка данных
data = struct.pack('>BHI', 255, 1000, 123456)
print(data.hex())  # ff03e8 0001e240

# Распаковка данных
values = struct.unpack('>BHI', data)
print(values)  # (255, 1000, 123456)

# Частичная распаковка
with open('file.bin', 'rb') as f:
    # Прочитать header: magic (4), version (2), size (4)
    header = f.read(10)
    magic, version, size = struct.unpack('>4sHI', header)
    print(f"Magic: {magic}, Version: {version}, Size: {size}")
```

## mmap — работа с большими файлами

```python
import mmap

# Работа с большими файлами без загрузки в память
with open('large_file.bin', 'r+b') as f:
    # Создание memory-mapped файла
    mm = mmap.mmap(f.fileno(), 0)

    # Чтение как из обычного файла
    print(mm[0:16].hex())

    # Поиск
    pos = mm.find(b'PATTERN')
    if pos != -1:
        print(f"Found at {pos}")

    # Модификация (записывается сразу в файл!)
    mm[100:104] = b'\x00\x00\x00\x00'

    mm.close()
```

## Best Practices

1. **Всегда делайте backup** перед редактированием бинарных файлов
2. **Используйте режим 'rb'/'wb'** для бинарных файлов в Python
3. **Проверяйте сигнатуры** перед разбором файла
4. **Учитывайте endianness** при работе со структурами
5. **Используйте mmap** для больших файлов
6. **Документируйте формат** при создании своих бинарных протоколов

```python
# Пример безопасного редактирования
import shutil

original = 'file.bin'
backup = 'file.bin.bak'

# Создание backup
shutil.copy2(original, backup)

# Редактирование
with open(original, 'r+b') as f:
    # ... модификации ...
    pass

# Если что-то пошло не так:
# shutil.copy2(backup, original)
```

## Практические задачи

### 1. Извлечение строк из бинарного файла

```python
def extract_strings(data, min_length=4):
    """Извлечь ASCII-строки из бинарных данных"""
    strings = []
    current = []

    for byte in data:
        if 32 <= byte < 127:  # Печатаемые ASCII
            current.append(chr(byte))
        else:
            if len(current) >= min_length:
                strings.append(''.join(current))
            current = []

    if len(current) >= min_length:
        strings.append(''.join(current))

    return strings

# Использование
with open('executable.bin', 'rb') as f:
    strings = extract_strings(f.read())
    for s in strings[:20]:
        print(s)
```

### 2. Сравнение двух файлов (binary diff)

```python
def binary_diff(file1, file2):
    """Найти различия между двумя файлами"""
    with open(file1, 'rb') as f1, open(file2, 'rb') as f2:
        data1 = f1.read()
        data2 = f2.read()

    differences = []
    max_len = max(len(data1), len(data2))

    for i in range(max_len):
        b1 = data1[i] if i < len(data1) else None
        b2 = data2[i] if i < len(data2) else None

        if b1 != b2:
            differences.append({
                'offset': i,
                'file1': f'0x{b1:02x}' if b1 is not None else 'EOF',
                'file2': f'0x{b2:02x}' if b2 is not None else 'EOF'
            })

    return differences

# Пример
diffs = binary_diff('original.bin', 'modified.bin')
for d in diffs[:10]:
    print(f"0x{d['offset']:08x}: {d['file1']} -> {d['file2']}")
```

## Связь с другими темами

- **Шестнадцатеричная система** — основа отображения в hex-редакторах
- **Биты и байты** — понимание структуры данных
- **Кодировки** — интерпретация текстовых данных в бинарных файлах
- **Сетевые протоколы** — анализ сетевых пакетов
- **Криптография** — анализ зашифрованных данных

---

[prev: 04-character-encoding](./04-character-encoding.md) | [next: 01-what-is-os](../03-operating-systems/01-what-is-os.md)
