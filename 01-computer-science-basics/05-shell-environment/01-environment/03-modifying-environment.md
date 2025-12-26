# Изменение окружения (Modifying Environment)

## Создание и изменение переменных

### Создание shell-переменной

Shell-переменные существуют только в текущем процессе shell:

```bash
# Создание переменной (без пробелов вокруг =)
MY_VAR="Hello World"

# НЕПРАВИЛЬНО (пробелы вокруг =):
MY_VAR = "value"  # Ошибка: MY_VAR: command not found

# Проверить значение
echo $MY_VAR
# Hello World

# Переменная НЕ видна дочерним процессам
bash -c 'echo $MY_VAR'
# (пусто)
```

### Экспорт переменной в окружение

Команда `export` делает переменную доступной для дочерних процессов:

```bash
# Способ 1: Создать и экспортировать отдельно
MY_VAR="Hello"
export MY_VAR

# Способ 2: Создать и экспортировать одной командой
export MY_VAR="Hello"

# Теперь переменная видна дочерним процессам
bash -c 'echo $MY_VAR'
# Hello

# Посмотреть все экспортированные переменные
export -p
```

### Различие между присваиванием и export

```bash
# Shell-переменная (локальная)
LOCAL_VAR="local value"

# Переменная окружения (экспортированная)
export ENV_VAR="exported value"

# Запустить скрипт — увидит только ENV_VAR
cat > /tmp/test.sh << 'EOF'
#!/bin/bash
echo "LOCAL_VAR: $LOCAL_VAR"
echo "ENV_VAR: $ENV_VAR"
EOF
chmod +x /tmp/test.sh
/tmp/test.sh
# LOCAL_VAR:
# ENV_VAR: exported value
```

## Команда export

### Синтаксис

```bash
# Экспортировать переменную
export VAR_NAME="value"

# Экспортировать несколько переменных
export VAR1="value1" VAR2="value2"

# Экспортировать существующую переменную
MY_VAR="hello"
export MY_VAR

# Просмотр экспортированных переменных
export -p

# Удалить переменную из окружения (но не из shell)
export -n VAR_NAME
```

### Практические примеры

```bash
# Добавить директорию в PATH
export PATH="$HOME/bin:$PATH"

# Установить редактор по умолчанию
export EDITOR="vim"
export VISUAL="vim"

# Настройки для программ
export LESS="-R"  # Цветной вывод в less
export GREP_OPTIONS="--color=auto"

# Локаль
export LANG="en_US.UTF-8"
export LC_ALL="en_US.UTF-8"
```

## Команда source (и точка)

Команда `source` выполняет скрипт в текущем shell (а не в дочернем процессе):

```bash
# Два способа записи (эквивалентны)
source ~/.bashrc
. ~/.bashrc

# Разница между source и обычным запуском:

# Обычный запуск — переменные остаются в дочернем процессе
./script.sh  # переменные из скрипта не попадут в текущий shell

# source — переменные появятся в текущем shell
source ./script.sh  # переменные из скрипта доступны в текущем shell
```

### Когда использовать source

```bash
# 1. Применить изменения в конфигурации
vim ~/.bashrc
source ~/.bashrc  # изменения вступят в силу

# 2. Загрузить переменные из файла
# Файл config.sh:
# DB_HOST="localhost"
# DB_PORT="5432"

source config.sh
echo $DB_HOST  # localhost

# 3. Загрузить функции
# Файл functions.sh:
# greet() { echo "Hello, $1!"; }

source functions.sh
greet "World"  # Hello, World!

# 4. Работа с виртуальными окружениями
source venv/bin/activate  # активировать Python venv
```

### Разница между source и exec

```bash
# source — выполняет в текущем shell, возвращает управление
source script.sh
echo "Это выполнится"

# exec — заменяет текущий shell
exec script.sh
echo "Это НИКОГДА не выполнится"
```

## Изменение PATH

PATH — критически важная переменная. Изменять её нужно аккуратно:

### Добавление директории в PATH

```bash
# Добавить в начало PATH (высший приоритет)
export PATH="$HOME/bin:$PATH"

# Добавить в конец PATH (низший приоритет)
export PATH="$PATH:/opt/myapp/bin"

# Добавить несколько директорий
export PATH="$HOME/bin:$HOME/.local/bin:$PATH"
```

### Проверка PATH

```bash
# Вывести PATH читаемо (каждая директория на новой строке)
echo $PATH | tr ':' '\n'

# Проверить, где находится команда
which python
# /usr/bin/python

# Показать все найденные версии
which -a python
# /usr/local/bin/python
# /usr/bin/python

# Или использовать type
type -a python
```

### Безопасное изменение PATH

```bash
# Добавить директорию только если она существует
if [ -d "$HOME/bin" ]; then
    export PATH="$HOME/bin:$PATH"
fi

# Добавить директорию только если её ещё нет в PATH
add_to_path() {
    if [[ ":$PATH:" != *":$1:"* ]]; then
        export PATH="$1:$PATH"
    fi
}
add_to_path "$HOME/bin"
```

## Удаление переменных

### Команда unset

```bash
# Создать переменную
MY_VAR="hello"
echo $MY_VAR  # hello

# Удалить переменную полностью
unset MY_VAR
echo $MY_VAR  # (пусто)

# unset работает и для экспортированных переменных
export MY_VAR="hello"
unset MY_VAR

# Удалить функцию
my_func() { echo "test"; }
unset -f my_func
```

### Установка пустого значения vs unset

```bash
# Установить пустое значение — переменная существует
MY_VAR=""
[ -z "$MY_VAR" ] && echo "Пусто"  # Пусто
[ -v MY_VAR ] && echo "Существует"  # Существует

# unset — переменная не существует
unset MY_VAR
[ -v MY_VAR ] && echo "Существует"  # (ничего)
```

## Команда env

Команда `env` позволяет запускать программы с изменённым окружением:

```bash
# Показать все переменные окружения
env

# Запустить команду с дополнительной переменной
env MY_VAR=hello bash -c 'echo $MY_VAR'
# hello

# Запустить команду с чистым окружением
env -i bash -c 'echo "PATH: $PATH"'
# PATH: (пусто или минимальный)

# Запустить с несколькими переменными
env VAR1=a VAR2=b command

# Удалить переменную для команды
env -u HOME bash -c 'echo $HOME'
# (пусто)
```

### Практические применения env

```bash
# Запустить скрипт с определённой локалью
env LANG=C sort file.txt

# Запустить Python с другим интерпретатором
env PATH="/opt/python3.11/bin:$PATH" python --version

# В shebang скрипта (для переносимости)
#!/usr/bin/env python3
# env найдёт python3 в PATH
```

## Временные переменные для одной команды

```bash
# Установить переменную только для одной команды
MY_VAR=hello bash -c 'echo $MY_VAR'
# hello

echo $MY_VAR
# (пусто — переменная не установлена в текущем shell)

# Несколько переменных
DB_HOST=localhost DB_PORT=5432 ./myapp

# Изменить PATH для одной команды
PATH="/custom/path:$PATH" mycommand
```

## Постоянное сохранение переменных

Чтобы переменные сохранялись между сессиями, добавьте их в файлы запуска:

### В ~/.bash_profile (для login shell)

```bash
# ~/.bash_profile

# Пути
export PATH="$HOME/bin:$PATH"

# Редактор
export EDITOR="vim"

# Локаль
export LANG="en_US.UTF-8"

# Приложения
export JAVA_HOME="/usr/lib/jvm/java-17"
export GOPATH="$HOME/go"
```

### В ~/.bashrc (для интерактивной оболочки)

```bash
# ~/.bashrc

# Загрузить переменные из отдельного файла
if [ -f ~/.env ]; then
    source ~/.env
fi
```

### Отдельный файл для секретов

```bash
# ~/.env (добавить в .gitignore!)
export API_KEY="secret-key-here"
export DATABASE_URL="postgres://user:pass@host/db"
```

## Полезные паттерны

### Значения по умолчанию

```bash
# Использовать значение по умолчанию, если переменная не установлена
echo ${MY_VAR:-"default value"}

# Установить значение по умолчанию, если переменная не установлена
echo ${MY_VAR:="default value"}
# MY_VAR теперь равна "default value"

# Ошибка, если переменная не установлена
echo ${MY_VAR:?"MY_VAR must be set"}
```

### Проверка переменных в скриптах

```bash
#!/bin/bash

# Проверить обязательные переменные
: "${DB_HOST:?DB_HOST must be set}"
: "${DB_PORT:?DB_PORT must be set}"

# Теперь безопасно использовать
echo "Connecting to $DB_HOST:$DB_PORT"
```

## Резюме

| Действие | Команда |
|----------|---------|
| Создать shell-переменную | `VAR="value"` |
| Экспортировать в окружение | `export VAR="value"` |
| Применить файл настроек | `source file` или `. file` |
| Удалить переменную | `unset VAR` |
| Запустить с изменённым окружением | `env VAR=value command` |
| Временная переменная для команды | `VAR=value command` |

- Shell-переменные локальны, экспортированные — наследуются
- Используйте `source` для применения изменений конфигурации
- Добавляйте директории в PATH с сохранением старого значения
- Сохраняйте постоянные переменные в `~/.bash_profile` или `~/.bashrc`
