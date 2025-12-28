# Команда read

[prev: 04-logical-operators](../04-flow-control-if/04-logical-operators.md) | [next: 02-ifs](./02-ifs.md)

---
## Основы команды read

Команда `read` считывает строку из стандартного ввода и присваивает её переменной:

```bash
#!/bin/bash

echo "Введите ваше имя:"
read name
echo "Привет, $name!"
```

## Базовый синтаксис

```bash
read [опции] [переменная1] [переменная2] ...
```

### Чтение в одну переменную

```bash
read name
echo "Вы ввели: $name"
```

### Чтение в несколько переменных

Слова из ввода распределяются по переменным:

```bash
echo "Введите имя и фамилию:"
read first_name last_name
echo "Имя: $first_name, Фамилия: $last_name"

# Ввод: Иван Петров
# Вывод: Имя: Иван, Фамилия: Петров

# Если слов больше, чем переменных, последняя получает остаток
echo "Введите три слова:"
read word1 word2
echo "word1=$word1, word2=$word2"

# Ввод: one two three four
# Вывод: word1=one, word2=two three four
```

### Чтение без указания переменной

Если переменная не указана, ввод сохраняется в `$REPLY`:

```bash
echo "Введите что-нибудь:"
read
echo "Вы ввели: $REPLY"
```

## Основные опции

### -p (prompt) - приглашение к вводу

```bash
read -p "Введите ваше имя: " name
echo "Привет, $name!"
```

### -s (silent) - скрытый ввод

Полезно для паролей:

```bash
read -sp "Введите пароль: " password
echo  # Перенос строки после скрытого ввода
echo "Пароль принят"
```

### -n (number) - ограничение количества символов

```bash
# Прочитать ровно 1 символ
read -n 1 -p "Продолжить? (y/n): " answer
echo
echo "Ваш ответ: $answer"

# Прочитать максимум 5 символов
read -n 5 -p "PIN: " pin
```

### -t (timeout) - таймаут

```bash
# Ждать ответа 5 секунд
if read -t 5 -p "Введите имя (5 сек): " name; then
    echo "Привет, $name!"
else
    echo "Время вышло!"
fi
```

### -r (raw) - отключение escape-последовательностей

```bash
# Без -r обратный слеш обрабатывается специально
read line
# Ввод: path\to\file
# line содержит: pathtofile

# С -r читает "как есть"
read -r line
# Ввод: path\to\file
# line содержит: path\to\file
```

**Рекомендация:** Всегда используйте `-r`, если не нужна обработка escape-последовательностей.

### -a (array) - чтение в массив

```bash
echo "Введите числа через пробел:"
read -a numbers
echo "Первое число: ${numbers[0]}"
echo "Второе число: ${numbers[1]}"
echo "Все числа: ${numbers[@]}"
```

### -d (delimiter) - указание разделителя

```bash
# Читать до символа ":" вместо новой строки
read -d ":" data
echo "Прочитано до двоеточия: $data"
```

### -e (readline) - редактирование строки

Включает поддержку readline (история, редактирование):

```bash
read -e -p "Команда: " cmd
# Можно использовать стрелки, Ctrl+A, Ctrl+E и т.д.
```

### -i (initial) - начальное значение

В сочетании с -e позволяет задать значение по умолчанию:

```bash
read -e -i "default_value" -p "Значение: " value
# Можно отредактировать предзаполненное значение
```

## Комбинирование опций

```bash
# Скрытый ввод с таймаутом
read -t 10 -sp "Пароль (10 сек): " password

# Один символ без Enter
read -n 1 -p "Нажмите любую клавишу..."

# Чтение с приглашением и сохранением escape
read -rp "Путь к файлу: " filepath
```

## Чтение из pipe и перенаправления

### Чтение из pipe

```bash
# Проблема: read в pipe выполняется в subshell
echo "hello" | read var
echo $var  # Пусто! var не видна

# Решение 1: here string
read var <<< "hello"
echo $var  # hello

# Решение 2: process substitution
while read line; do
    echo "Line: $line"
done < <(echo -e "line1\nline2")
```

### Чтение из файла

```bash
# Построчное чтение файла
while IFS= read -r line; do
    echo "Строка: $line"
done < filename.txt
```

### Чтение из переменной

```bash
data="line1
line2
line3"

while IFS= read -r line; do
    echo "Строка: $line"
done <<< "$data"
```

## Практические примеры

### Подтверждение действия

```bash
confirm() {
    local message=${1:-"Продолжить?"}

    while true; do
        read -rp "$message [y/n]: " answer
        case $answer in
            [Yy]|[Yy][Ee][Ss]|[Дд][Аа])
                return 0
                ;;
            [Nn]|[Nn][Oo]|[Нн][Ее][Тт])
                return 1
                ;;
            *)
                echo "Введите y или n"
                ;;
        esac
    done
}

if confirm "Удалить файлы?"; then
    echo "Удаление..."
else
    echo "Отмена"
fi
```

### Ввод пароля с подтверждением

```bash
read_password() {
    local password password2

    while true; do
        read -rsp "Введите пароль: " password
        echo
        read -rsp "Повторите пароль: " password2
        echo

        if [[ "$password" == "$password2" ]]; then
            if [[ ${#password} -ge 8 ]]; then
                echo "$password"
                return 0
            else
                echo "Пароль должен быть не менее 8 символов"
            fi
        else
            echo "Пароли не совпадают"
        fi
    done
}

password=$(read_password)
echo "Пароль установлен"
```

### Ввод с значением по умолчанию

```bash
read_with_default() {
    local prompt=$1
    local default=$2
    local value

    read -rp "$prompt [$default]: " value
    echo "${value:-$default}"
}

name=$(read_with_default "Имя" "Anonymous")
port=$(read_with_default "Порт" "8080")

echo "Имя: $name, Порт: $port"
```

### Чтение CSV

```bash
# data.csv:
# name,age,city
# Alice,25,Moscow
# Bob,30,SPb

while IFS=',' read -r name age city; do
    echo "Имя: $name, Возраст: $age, Город: $city"
done < data.csv
```

### Интерактивный выбор из списка

```bash
select_option() {
    local options=("$@")
    local PS3="Выберите опцию: "

    select opt in "${options[@]}"; do
        if [[ -n "$opt" ]]; then
            echo "$opt"
            return 0
        else
            echo "Неверный выбор" >&2
        fi
    done
}

choice=$(select_option "Опция 1" "Опция 2" "Опция 3")
echo "Вы выбрали: $choice"
```

## Best Practices

1. **Всегда используйте -r** для отключения обработки backslash

2. **Используйте IFS= для сохранения пробелов**:
   ```bash
   IFS= read -r line  # Сохраняет ведущие/конечные пробелы
   ```

3. **Проверяйте успешность read**:
   ```bash
   if ! read -t 5 -rp "Input: " value; then
       echo "Timeout or error"
   fi
   ```

4. **Используйте local в функциях**:
   ```bash
   get_input() {
       local value
       read -rp "Enter: " value
       echo "$value"
   }
   ```

---

[prev: 04-logical-operators](../04-flow-control-if/04-logical-operators.md) | [next: 02-ifs](./02-ifs.md)
