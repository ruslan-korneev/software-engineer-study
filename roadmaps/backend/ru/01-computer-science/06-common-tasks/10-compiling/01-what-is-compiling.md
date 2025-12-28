# Что такое компиляция

[prev: 01-lpr-lpstat](../09-printing/01-lpr-lpstat.md) | [next: 02-make](./02-make.md)

---
## Введение

**Компиляция** - это процесс преобразования исходного кода программы, написанного на языке программирования высокого уровня, в машинный код, который может выполняться компьютером.

## Этапы компиляции

### 1. Препроцессинг (Preprocessing)

Обработка директив препроцессора:

```c
// Исходный код
#include <stdio.h>
#define MAX 100

int main() {
    printf("Max: %d\n", MAX);
}
```

После препроцессинга:
- `#include` заменяется содержимым файла
- `#define` заменяет макросы значениями
- Удаляются комментарии

```bash
# Только препроцессинг
gcc -E source.c -o source.i
```

### 2. Компиляция (Compilation)

Преобразование в ассемблерный код:

```bash
# Получить ассемблерный код
gcc -S source.c -o source.s
```

```asm
; Пример ассемблера (x86-64)
    .section .text
    .globl main
main:
    pushq   %rbp
    movq    %rsp, %rbp
    movl    $100, %esi
    leaq    .LC0(%rip), %rdi
    call    printf@PLT
    ...
```

### 3. Ассемблирование (Assembly)

Преобразование ассемблера в объектный код:

```bash
# Получить объектный файл
gcc -c source.c -o source.o

# Или из ассемблера
as source.s -o source.o
```

Объектный файл содержит машинный код, но ещё не готов к выполнению.

### 4. Компоновка (Linking)

Объединение объектных файлов и библиотек в исполняемый файл:

```bash
# Финальная компоновка
gcc source.o -o program

# Или напрямую из исходника
gcc source.c -o program
```

```
[source1.o] + [source2.o] + [libc.so] → [executable]
```

## GCC - GNU Compiler Collection

### Установка

```bash
# Debian/Ubuntu
sudo apt install build-essential

# RHEL/CentOS
sudo yum groupinstall "Development Tools"

# Fedora
sudo dnf groupinstall "Development Tools"

# Проверка
gcc --version
```

### Базовое использование

```bash
# Компиляция и компоновка
gcc source.c -o program

# Запуск
./program

# Компилировать несколько файлов
gcc main.c utils.c -o program

# С отладочной информацией
gcc -g source.c -o program

# С оптимизацией
gcc -O2 source.c -o program
```

### Уровни оптимизации

```bash
gcc -O0 source.c -o program  # Без оптимизации (по умолчанию)
gcc -O1 source.c -o program  # Базовая оптимизация
gcc -O2 source.c -o program  # Умеренная оптимизация
gcc -O3 source.c -o program  # Агрессивная оптимизация
gcc -Os source.c -o program  # Оптимизация по размеру
gcc -Ofast source.c -o program  # Максимальная скорость
```

### Предупреждения

```bash
# Включить все предупреждения
gcc -Wall source.c -o program

# Дополнительные предупреждения
gcc -Wall -Wextra source.c -o program

# Предупреждения как ошибки
gcc -Wall -Werror source.c -o program

# Педантичный режим
gcc -Wall -Wextra -pedantic source.c -o program
```

### Стандарты языка

```bash
# C стандарты
gcc -std=c89 source.c -o program
gcc -std=c99 source.c -o program
gcc -std=c11 source.c -o program
gcc -std=c17 source.c -o program
gcc -std=c2x source.c -o program  # будущий C23

# C++ стандарты
g++ -std=c++11 source.cpp -o program
g++ -std=c++14 source.cpp -o program
g++ -std=c++17 source.cpp -o program
g++ -std=c++20 source.cpp -o program
```

## Библиотеки

### Статические библиотеки (.a)

```bash
# Создать статическую библиотеку
gcc -c utils.c -o utils.o
ar rcs libutils.a utils.o

# Использовать библиотеку
gcc main.c -L. -lutils -o program
```

### Динамические библиотеки (.so)

```bash
# Создать shared library
gcc -fPIC -c utils.c -o utils.o
gcc -shared -o libutils.so utils.o

# Использовать
gcc main.c -L. -lutils -o program

# Путь к библиотеке при запуске
export LD_LIBRARY_PATH=.:$LD_LIBRARY_PATH
./program
```

### Системные библиотеки

```bash
# Подключить библиотеку math
gcc source.c -lm -o program

# pthread
gcc source.c -lpthread -o program

# OpenSSL
gcc source.c -lssl -lcrypto -o program
```

## Заголовочные файлы

```bash
# Указать директорию с заголовками
gcc -I/path/to/headers source.c -o program

# Несколько директорий
gcc -I./include -I/usr/local/include source.c -o program
```

## Препроцессор

```bash
# Определить макрос
gcc -DDEBUG source.c -o program

# Макрос со значением
gcc -DVERSION=2 source.c -o program

# Эквивалент в коде
#define DEBUG
#define VERSION 2
```

## Пример полной компиляции

### Структура проекта

```
project/
├── include/
│   └── utils.h
├── src/
│   ├── main.c
│   └── utils.c
└── Makefile
```

### Файлы

```c
// include/utils.h
#ifndef UTILS_H
#define UTILS_H

int add(int a, int b);
void greet(const char *name);

#endif
```

```c
// src/utils.c
#include <stdio.h>
#include "utils.h"

int add(int a, int b) {
    return a + b;
}

void greet(const char *name) {
    printf("Hello, %s!\n", name);
}
```

```c
// src/main.c
#include <stdio.h>
#include "utils.h"

int main() {
    greet("World");
    printf("2 + 3 = %d\n", add(2, 3));
    return 0;
}
```

### Ручная компиляция

```bash
# Компилировать объектные файлы
gcc -c -I./include src/main.c -o main.o
gcc -c -I./include src/utils.c -o utils.o

# Скомпоновать
gcc main.o utils.o -o program

# Запустить
./program
```

## Отладка

```bash
# Компиляция с отладочной информацией
gcc -g source.c -o program

# Запуск в отладчике
gdb ./program

# Команды GDB
(gdb) break main      # точка останова
(gdb) run             # запуск
(gdb) next            # следующая строка
(gdb) step            # войти в функцию
(gdb) print variable  # показать переменную
(gdb) continue        # продолжить
(gdb) quit            # выход
```

## Альтернативные компиляторы

### Clang

```bash
# Установка
sudo apt install clang

# Использование (совместим с gcc)
clang source.c -o program
clang++ source.cpp -o program

# Преимущества:
# - Лучшие сообщения об ошибках
# - Быстрее компиляция
# - Лучше для статического анализа
```

### ICC (Intel)

```bash
# Для высокопроизводительных вычислений
icc source.c -o program
```

## Резюме команд

| Команда | Описание |
|---------|----------|
| `gcc source.c -o prog` | Скомпилировать |
| `gcc -c source.c` | Только объектный файл |
| `gcc -S source.c` | Только ассемблер |
| `gcc -E source.c` | Только препроцессинг |
| `gcc -g` | Отладочная информация |
| `gcc -O2` | Оптимизация |
| `gcc -Wall` | Все предупреждения |
| `gcc -I/path` | Путь к заголовкам |
| `gcc -L/path` | Путь к библиотекам |
| `gcc -lname` | Подключить библиотеку |
| `gcc -DMACRO` | Определить макрос |
| `gcc -std=c11` | Стандарт языка |

---

[prev: 01-lpr-lpstat](../09-printing/01-lpr-lpstat.md) | [next: 02-make](./02-make.md)
