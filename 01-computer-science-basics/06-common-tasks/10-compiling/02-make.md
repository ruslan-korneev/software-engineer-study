# make и Makefile - автоматизация сборки

## Что такое make?

**make** - утилита для автоматизации сборки программ. Она использует файл `Makefile` с правилами, определяющими как собирать проект.

## Зачем нужен make?

- **Инкрементальная сборка** - пересобирает только изменённые файлы
- **Управление зависимостями** - знает порядок сборки
- **Повторяемость** - одна команда для сборки
- **Документация** - Makefile описывает процесс сборки

## Базовый синтаксис Makefile

```makefile
target: dependencies
	command
	command
```

**Важно:** Команды должны начинаться с TAB, не с пробелов!

### Простой пример

```makefile
# Makefile

program: main.o utils.o
	gcc main.o utils.o -o program

main.o: main.c utils.h
	gcc -c main.c -o main.o

utils.o: utils.c utils.h
	gcc -c utils.c -o utils.o

clean:
	rm -f *.o program
```

```bash
# Сборка
make

# Конкретная цель
make program
make clean

# Указать файл
make -f MyMakefile
```

## Переменные

### Определение переменных

```makefile
# Простые переменные
CC = gcc
CFLAGS = -Wall -g
LDFLAGS = -lm

# Немедленное присваивание
CC := gcc

# Условное (только если не задано)
CC ?= gcc

# Добавление
CFLAGS += -O2
```

### Использование переменных

```makefile
CC = gcc
CFLAGS = -Wall -g
OBJECTS = main.o utils.o

program: $(OBJECTS)
	$(CC) $(CFLAGS) $(OBJECTS) -o program

main.o: main.c
	$(CC) $(CFLAGS) -c main.c

utils.o: utils.c
	$(CC) $(CFLAGS) -c utils.c
```

### Автоматические переменные

```makefile
# $@ - имя цели
# $< - первая зависимость
# $^ - все зависимости
# $* - основа имени (без расширения)

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@
```

## Шаблонные правила

### Неявные правила

```makefile
# Правило для всех .o из .c
%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

# Теперь достаточно
program: main.o utils.o
	$(CC) $(LDFLAGS) $^ -o $@
```

### Встроенные неявные правила

make уже знает базовые правила:

```bash
# Показать встроенные правила
make -p | grep -A3 '\.c\.o:'

# По умолчанию:
# .c.o:
#     $(CC) $(CFLAGS) $(CPPFLAGS) -c -o $@ $<
```

## Фальшивые цели (Phony)

```makefile
.PHONY: all clean install test

all: program

program: main.o utils.o
	$(CC) $^ -o $@

clean:
	rm -f *.o program

install: program
	cp program /usr/local/bin/

test: program
	./test.sh
```

`.PHONY` говорит make, что эти цели - не файлы.

## Полный пример Makefile

```makefile
# Компилятор и флаги
CC = gcc
CFLAGS = -Wall -Wextra -g
LDFLAGS = -lm

# Директории
SRCDIR = src
INCDIR = include
OBJDIR = obj
BINDIR = bin

# Файлы
SOURCES = $(wildcard $(SRCDIR)/*.c)
OBJECTS = $(patsubst $(SRCDIR)/%.c,$(OBJDIR)/%.o,$(SOURCES))
TARGET = $(BINDIR)/program

# Основная цель
all: directories $(TARGET)

# Создать директории
directories:
	@mkdir -p $(OBJDIR) $(BINDIR)

# Сборка программы
$(TARGET): $(OBJECTS)
	$(CC) $(LDFLAGS) $^ -o $@

# Компиляция исходников
$(OBJDIR)/%.o: $(SRCDIR)/%.c
	$(CC) $(CFLAGS) -I$(INCDIR) -c $< -o $@

# Очистка
clean:
	rm -rf $(OBJDIR) $(BINDIR)

# Переустановка
rebuild: clean all

# Запуск
run: $(TARGET)
	./$(TARGET)

# Зависимости
-include $(OBJECTS:.o=.d)

# Генерация зависимостей
$(OBJDIR)/%.d: $(SRCDIR)/%.c
	@mkdir -p $(OBJDIR)
	$(CC) -MM -MT '$(OBJDIR)/$*.o' $< > $@

.PHONY: all directories clean rebuild run
```

## Условия

```makefile
# ifdef / ifndef
ifdef DEBUG
    CFLAGS += -g -DDEBUG
else
    CFLAGS += -O2
endif

# ifeq / ifneq
ifeq ($(CC),gcc)
    CFLAGS += -std=gnu11
else
    CFLAGS += -std=c11
endif

# Проверка ОС
ifeq ($(shell uname),Linux)
    LDFLAGS += -lpthread
endif
```

## Функции

### Работа со строками

```makefile
# $(subst from,to,text) - замена
FILES = foo.c bar.c
OBJS = $(subst .c,.o,$(FILES))
# OBJS = foo.o bar.o

# $(patsubst pattern,replacement,text)
OBJS = $(patsubst %.c,%.o,$(FILES))

# $(strip string) - удалить пробелы
# $(findstring find,in) - поиск
# $(filter pattern,text) - фильтрация
# $(filter-out pattern,text) - исключение
```

### Работа с файлами

```makefile
# $(wildcard pattern) - найти файлы
SOURCES = $(wildcard src/*.c)

# $(dir names) - директория
# $(notdir names) - только имя файла
# $(basename names) - без расширения
# $(suffix names) - только расширение
```

### Выполнение команд

```makefile
# $(shell command) - выполнить команду
DATE = $(shell date +%Y%m%d)
GIT_HASH = $(shell git rev-parse --short HEAD)

# Использование
CFLAGS += -DBUILD_DATE=\"$(DATE)\"
CFLAGS += -DGIT_HASH=\"$(GIT_HASH)\"
```

## Параллельная сборка

```bash
# Использовать N процессов
make -j4       # 4 процесса
make -j$(nproc)  # все ядра

# В Makefile можно указать зависимости для корректной параллелизации
```

## Отладка Makefile

```bash
# Показать команды без выполнения
make -n

# Показать базу данных правил
make -p

# Подробный вывод
make --debug=all

# Показать почему цель пересобирается
make --debug=why target

# Игнорировать ошибки
make -k
```

## Распространённые рецепты

### Проект на C

```makefile
CC = gcc
CFLAGS = -Wall -Wextra -std=c11 -g
LDFLAGS =

SRCS = $(wildcard *.c)
OBJS = $(SRCS:.c=.o)
TARGET = program

all: $(TARGET)

$(TARGET): $(OBJS)
	$(CC) $(LDFLAGS) $^ -o $@

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

clean:
	rm -f $(OBJS) $(TARGET)

.PHONY: all clean
```

### Проект на C++

```makefile
CXX = g++
CXXFLAGS = -Wall -Wextra -std=c++17 -g
LDFLAGS =

SRCS = $(wildcard *.cpp)
OBJS = $(SRCS:.cpp=.o)
TARGET = program

all: $(TARGET)

$(TARGET): $(OBJS)
	$(CXX) $(LDFLAGS) $^ -o $@

%.o: %.cpp
	$(CXX) $(CXXFLAGS) -c $< -o $@

clean:
	rm -f $(OBJS) $(TARGET)

.PHONY: all clean
```

### С тестами

```makefile
.PHONY: all test clean

all: program

program: main.o
	$(CC) $^ -o $@

test: program
	./program --test
	@echo "All tests passed!"

clean:
	rm -f *.o program
```

## Альтернативы make

### CMake

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.10)
project(MyProject)

set(CMAKE_C_STANDARD 11)
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -Wextra")

add_executable(program main.c utils.c)
```

```bash
mkdir build && cd build
cmake ..
make
```

### Ninja

```bash
# Быстрее чем make для больших проектов
ninja
```

### Meson

```python
# meson.build
project('myproject', 'c')
executable('program', 'main.c', 'utils.c')
```

## Резюме

### Структура Makefile

```makefile
# Переменные
CC = gcc
CFLAGS = -Wall

# Цель по умолчанию
all: program

# Правила
target: dependencies
	commands

# Шаблоны
%.o: %.c
	$(CC) -c $< -o $@

# Фальшивые цели
.PHONY: all clean
```

### Команды make

| Команда | Описание |
|---------|----------|
| `make` | Собрать цель по умолчанию |
| `make target` | Собрать конкретную цель |
| `make -j4` | Параллельно (4 потока) |
| `make -n` | Показать команды |
| `make -f file` | Использовать другой файл |
| `make clean` | Очистить (если есть цель) |

### Автоматические переменные

| Переменная | Значение |
|------------|----------|
| `$@` | Имя цели |
| `$<` | Первая зависимость |
| `$^` | Все зависимости |
| `$*` | Основа имени файла |
| `$?` | Изменённые зависимости |
