# Синтаксические ошибки

[prev: 04-reading-files](../06-flow-control-loops/04-reading-files.md) | [next: 02-logical-errors](./02-logical-errors.md)

---
## Типы синтаксических ошибок

Синтаксические ошибки возникают, когда код нарушает правила языка bash. Такие ошибки обнаруживаются при парсинге скрипта, до его выполнения.

## Распространённые синтаксические ошибки

### 1. Пропущенные закрывающие элементы

```bash
# Незакрытая кавычка
echo "Hello world
# bash: unexpected EOF while looking for matching `"'

# Исправление
echo "Hello world"

# Незакрытая скобка в if
if [[ $x -eq 1 ]
# bash: syntax error near unexpected token `then'

# Исправление
if [[ $x -eq 1 ]]; then

# Пропущен fi
if [[ $x -eq 1 ]]; then
    echo "One"
# bash: syntax error: unexpected end of file

# Исправление
if [[ $x -eq 1 ]]; then
    echo "One"
fi
```

### 2. Пробелы в присваивании

```bash
# НЕПРАВИЛЬНО - пробелы вокруг =
name = "John"
# bash: name: command not found

# ПРАВИЛЬНО
name="John"

# НЕПРАВИЛЬНО - пробелы в условии
if [$x -eq 1]; then
# bash: [1: command not found

# ПРАВИЛЬНО
if [ $x -eq 1 ]; then
# или лучше
if [[ $x -eq 1 ]]; then
```

### 3. Отсутствие then в if

```bash
# НЕПРАВИЛЬНО
if [[ $x -eq 1 ]]
    echo "One"
fi
# bash: syntax error near unexpected token `echo'

# ПРАВИЛЬНО
if [[ $x -eq 1 ]]; then
    echo "One"
fi
```

### 4. Ошибки в case

```bash
# Пропущена закрывающая скобка
case $x in
    1 echo "One" ;;
esac
# bash: syntax error near unexpected token `echo'

# ПРАВИЛЬНО
case $x in
    1) echo "One" ;;
esac

# Пропущен ;;
case $x in
    1) echo "One"
    2) echo "Two" ;;
esac
# bash: syntax error near unexpected token `)'

# ПРАВИЛЬНО
case $x in
    1) echo "One" ;;
    2) echo "Two" ;;
esac
```

### 5. Ошибки в циклах

```bash
# Пропущен do
for i in 1 2 3
    echo $i
done
# bash: syntax error near unexpected token `echo'

# ПРАВИЛЬНО
for i in 1 2 3; do
    echo $i
done

# Пропущен done
while true; do
    echo "Loop"
# bash: syntax error: unexpected end of file

# ПРАВИЛЬНО
while true; do
    echo "Loop"
done
```

### 6. Ошибки в функциях

```bash
# Пропущены скобки или фигурные скобки
function greet
    echo "Hello"
# bash: syntax error near unexpected token `echo'

# ПРАВИЛЬНО
greet() {
    echo "Hello"
}
# или
function greet {
    echo "Hello"
}

# Пробел после {
greet() {echo "Hello"}
# bash: syntax error near unexpected token `}'

# ПРАВИЛЬНО
greet() { echo "Hello"; }
```

### 7. Неправильные операторы

```bash
# Неправильный оператор сравнения в [ ]
if [ $x == $y ]; then  # == работает, но = предпочтительнее в [ ]

# Неправильный оператор для чисел
if [[ $x = 5 ]]; then  # Это строковое сравнение!

# ПРАВИЛЬНО
if [[ $x -eq 5 ]]; then  # Числовое сравнение
```

## Проверка синтаксиса

### Опция -n

Проверяет синтаксис без выполнения скрипта:

```bash
bash -n script.sh
# Выведет ошибки, если есть
# Ничего не выведет, если синтаксис корректен

# Пример с ошибкой
echo 'if [[ $x' | bash -n
# bash: line 1: unexpected EOF while looking for matching `]]'
```

### ShellCheck

Статический анализатор для shell-скриптов:

```bash
# Установка
sudo apt install shellcheck  # Debian/Ubuntu
brew install shellcheck      # macOS

# Использование
shellcheck script.sh
```

ShellCheck находит:
- Синтаксические ошибки
- Потенциальные баги
- Стилистические проблемы
- Проблемы совместимости

```bash
# Пример вывода shellcheck
$ shellcheck script.sh
In script.sh line 3:
echo $name
     ^----^ SC2086: Double quote to prevent globbing and word splitting.
```

## Отладка синтаксических ошибок

### Шаг 1: Локализация ошибки

```bash
# Сообщение об ошибке указывает номер строки
bash: line 15: syntax error near unexpected token `fi'

# Но реальная ошибка может быть раньше!
# Проверьте строки выше на:
# - Незакрытые кавычки
# - Незакрытые скобки
# - Пропущенные then/do/esac
```

### Шаг 2: Изоляция проблемы

```bash
# Закомментируйте части кода
# Уменьшайте код до минимального примера с ошибкой

# Проверяйте каждую конструкцию отдельно
if [[ $x -eq 1 ]]; then
    echo "test"
fi  # Работает?

# Добавляйте код постепенно
```

### Шаг 3: Проверка парности

```bash
# Подсчитайте количество:
grep -c 'if' script.sh   # должно равняться
grep -c 'fi' script.sh   # этому

grep -c 'do' script.sh   # должно равняться
grep -c 'done' script.sh # этому

grep -c 'case' script.sh # должно равняться
grep -c 'esac' script.sh # этому
```

## Предотвращение синтаксических ошибок

### 1. Используйте редактор с подсветкой синтаксиса

Редакторы покажут незакрытые кавычки и скобки:
- VS Code с расширением ShellCheck
- Vim с плагинами
- Sublime Text
- JetBrains IDEs

### 2. Используйте форматирование и отступы

```bash
# Плохо
if [[ $x -eq 1 ]]; then
echo "one"
if [[ $y -eq 2 ]]; then
echo "two"
fi
fi

# Хорошо
if [[ $x -eq 1 ]]; then
    echo "one"
    if [[ $y -eq 2 ]]; then
        echo "two"
    fi
fi
```

### 3. Используйте шаблоны

```bash
# Шаблон if
if [[ условие ]]; then
    команды
fi

# Шаблон for
for item in список; do
    команды
done

# Шаблон while
while [[ условие ]]; do
    команды
done

# Шаблон case
case "$var" in
    pattern1)
        команды
        ;;
    pattern2)
        команды
        ;;
    *)
        команды
        ;;
esac
```

### 4. Проверяйте перед выполнением

```bash
# Добавьте в workflow
bash -n script.sh && shellcheck script.sh && ./script.sh
```

## Пример отладки

```bash
#!/bin/bash
# Найдите ошибки в этом коде:

name="John
age=25

if [ $age -gt 18 ]
    echo "Adult"
fi

for i in 1 2 3
    echo $i
done

case $name in
    John echo "Hello John"
esac
```

Исправленная версия:

```bash
#!/bin/bash

name="John"  # Закрыта кавычка
age=25

if [ $age -gt 18 ]; then  # Добавлены ; then
    echo "Adult"
fi

for i in 1 2 3; do  # Добавлены ; do
    echo $i
done

case $name in
    John) echo "Hello John" ;;  # Добавлены ) и ;;
esac
```

---

[prev: 04-reading-files](../06-flow-control-loops/04-reading-files.md) | [next: 02-logical-errors](./02-logical-errors.md)
