# Команды type и which

## Типы команд в shell

В shell существует несколько типов команд:

1. **Встроенные команды (builtins)** — часть самого shell
2. **Внешние программы** — исполняемые файлы в файловой системе
3. **Функции shell** — определённые пользователем функции
4. **Алиасы** — псевдонимы для других команд

## Команда type

**type** — встроенная команда shell для определения типа команды.

### Синтаксис
```bash
type [опции] имя...
```

### Базовое использование

```bash
$ type cd
cd is a shell builtin

$ type ls
ls is aliased to 'ls --color=auto'

$ type python
python is /usr/bin/python

$ type for
for is a shell keyword
```

### Типы результатов

```bash
# Встроенная команда shell
$ type cd
cd is a shell builtin

# Алиас
$ type ll
ll is aliased to 'ls -la'

# Внешняя программа
$ type grep
grep is /usr/bin/grep

# Ключевое слово shell
$ type if
if is a shell keyword

# Функция
$ type my_function
my_function is a function
```

### Полезные опции

#### -a — показать все варианты
```bash
$ type -a echo
echo is a shell builtin
echo is /bin/echo

# echo существует и как builtin, и как программа
```

#### -t — только тип (без пояснений)
```bash
$ type -t cd
builtin

$ type -t ls
alias

$ type -t grep
file
```

#### -p — путь к исполняемому файлу
```bash
$ type -p python
/usr/bin/python
```

#### -P — путь (игнорируя builtins и aliases)
```bash
$ type -P echo
/bin/echo
```

## Команда which

**which** — показывает путь к исполняемому файлу.

### Синтаксис
```bash
which [опции] команда...
```

### Базовое использование

```bash
$ which python
/usr/bin/python

$ which ls
/bin/ls

$ which node npm
/usr/bin/node
/usr/bin/npm
```

### Особенности which

`which` ищет только исполняемые файлы в PATH, игнорируя:
- Встроенные команды shell
- Алиасы
- Функции

```bash
$ which cd
# Ничего не выводит (cd — builtin)

$ type cd
cd is a shell builtin
```

### Полезные опции

#### -a — показать все совпадения
```bash
$ which -a python
/usr/bin/python
/usr/local/bin/python
```

## Сравнение type и which

| Аспект | type | which |
|--------|------|-------|
| Builtins | Да | Нет |
| Aliases | Да | Нет (обычно) |
| Functions | Да | Нет |
| Внешние программы | Да | Да |
| Рекомендуется в скриптах | Да | Нет |

## Практические примеры

### Проверка установлена ли программа
```bash
# С which
$ which python3 > /dev/null && echo "Python установлен"

# С type (предпочтительно)
$ type python3 &> /dev/null && echo "Python установлен"
```

### Понимание что выполняется
```bash
$ type -a python
python is /usr/local/bin/python     # это выполняется первым
python is /usr/bin/python           # альтернативный путь
```

### Найти все версии программы
```bash
$ which -a node
/usr/local/bin/node
/usr/bin/node
```

### В скриптах
```bash
#!/bin/bash

# Проверка зависимостей
for cmd in git docker python3; do
    if ! type "$cmd" &> /dev/null; then
        echo "Ошибка: $cmd не установлен"
        exit 1
    fi
done
```

### Понять алиас
```bash
$ type ls
ls is aliased to 'ls --color=auto'

# Чтобы использовать оригинальную команду:
$ \ls                    # экранирование
$ command ls             # явный вызов
$ /bin/ls                # полный путь
```

## Дополнительные команды

### command — выполнить команду (не алиас)
```bash
$ alias ls='ls --color=auto'
$ command ls              # выполнит /bin/ls без цвета
```

### whereis — расширенный поиск
```bash
$ whereis python
python: /usr/bin/python /usr/lib/python3.10 /usr/share/man/man1/python.1.gz

# Показывает бинарник, исходники, man-страницы
```

### hash — кэш путей команд
```bash
$ hash
hits    command
   3    /usr/bin/git
   5    /bin/ls
   2    /usr/bin/vim
```

## Переменная PATH

`which` и `type` ищут команды в директориях из PATH:

```bash
$ echo $PATH
/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin

$ type -P python
/usr/bin/python     # первое совпадение в PATH
```

## Советы

1. **Используйте `type` в скриптах** — он надёжнее
2. **`type -t`** для проверки типа в условиях
3. **`which -a`** чтобы найти все версии программы
4. **`type -a`** чтобы понять приоритет команд
5. **Помните**: алиас имеет приоритет над внешней командой
