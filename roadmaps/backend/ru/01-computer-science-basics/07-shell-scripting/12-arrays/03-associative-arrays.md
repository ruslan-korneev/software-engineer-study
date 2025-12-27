# Ассоциативные массивы

## Что такое ассоциативные массивы

Ассоциативные массивы (хеш-таблицы, словари) используют строковые ключи вместо числовых индексов. Доступны в bash 4.0+.

## Создание

### Объявление обязательно

```bash
#!/bin/bash

# ОБЯЗАТЕЛЬНО использовать declare -A
declare -A user

user[name]="John"
user[age]="25"
user[city]="Moscow"

# Без declare -A это будет индексированный массив!
```

### Инициализация при объявлении

```bash
#!/bin/bash

declare -A colors=(
    [red]="#FF0000"
    [green]="#00FF00"
    [blue]="#0000FF"
)

# Или в одну строку
declare -A config=([host]="localhost" [port]="8080")
```

## Доступ к элементам

```bash
#!/bin/bash

declare -A user=([name]="John" [age]="25" [city]="Moscow")

# Получение значения по ключу
echo "${user[name]}"   # John
echo "${user[age]}"    # 25

# Все значения
echo "${user[@]}"      # John 25 Moscow (порядок не гарантирован)

# Все ключи
echo "${!user[@]}"     # name age city (порядок не гарантирован)

# Количество элементов
echo "${#user[@]}"     # 3
```

## Проверка наличия ключа

```bash
#!/bin/bash

declare -A data=([key1]="value1" [key2]="value2")

# Способ 1: через -v
if [[ -v data[key1] ]]; then
    echo "key1 существует"
fi

if [[ ! -v data[key3] ]]; then
    echo "key3 не существует"
fi

# Способ 2: проверка пустоты (менее надёжно)
if [[ -n "${data[key1]+x}" ]]; then
    echo "key1 существует"
fi
```

## Итерация

```bash
#!/bin/bash

declare -A user=([name]="John" [age]="25" [city]="Moscow")

# По значениям
for value in "${user[@]}"; do
    echo "Value: $value"
done

# По ключам
for key in "${!user[@]}"; do
    echo "Key: $key"
done

# По парам ключ-значение
for key in "${!user[@]}"; do
    echo "$key = ${user[$key]}"
done
```

## Модификация

### Добавление/изменение

```bash
#!/bin/bash

declare -A config

# Добавление
config[host]="localhost"
config[port]="8080"

# Изменение
config[port]="3000"

# Добавление с проверкой
if [[ ! -v config[timeout] ]]; then
    config[timeout]="30"
fi
```

### Удаление

```bash
#!/bin/bash

declare -A data=([a]="1" [b]="2" [c]="3")

# Удаление элемента
unset data[b]
echo "${!data[@]}"  # a c

# Удаление всего массива
unset data
```

## Практические примеры

### Конфигурация

```bash
#!/bin/bash

declare -A config

# Значения по умолчанию
config=(
    [log_level]="INFO"
    [max_retries]="3"
    [timeout]="30"
)

# Загрузка из файла (если существует)
if [[ -f "config.ini" ]]; then
    while IFS='=' read -r key value; do
        [[ -z "$key" || "$key" == \#* ]] && continue
        key=$(echo "$key" | xargs)    # trim
        value=$(echo "$value" | xargs)
        config[$key]="$value"
    done < "config.ini"
fi

# Использование
echo "Log level: ${config[log_level]}"
echo "Timeout: ${config[timeout]}"
```

### Подсчёт частоты

```bash
#!/bin/bash

declare -A word_count

text="apple banana apple cherry banana apple"

for word in $text; do
    ((word_count[$word]++))
done

# Вывод результата
for word in "${!word_count[@]}"; do
    echo "$word: ${word_count[$word]}"
done
# apple: 3
# banana: 2
# cherry: 1
```

### Кеширование

```bash
#!/bin/bash

declare -A cache

expensive_operation() {
    local input=$1

    # Проверяем кеш
    if [[ -v cache[$input] ]]; then
        echo "${cache[$input]}"
        return
    fi

    # Дорогая операция (симуляция)
    sleep 1
    local result="Result for $input"

    # Сохраняем в кеш
    cache[$input]="$result"
    echo "$result"
}

# Первый вызов - долго
expensive_operation "query1"

# Второй вызов - из кеша (быстро)
expensive_operation "query1"
```

### Маппинг значений

```bash
#!/bin/bash

# Коды HTTP статусов
declare -A http_status=(
    [200]="OK"
    [201]="Created"
    [400]="Bad Request"
    [401]="Unauthorized"
    [403]="Forbidden"
    [404]="Not Found"
    [500]="Internal Server Error"
)

get_status_text() {
    local code=$1
    echo "${http_status[$code]:-Unknown Status}"
}

echo "200: $(get_status_text 200)"  # OK
echo "404: $(get_status_text 404)"  # Not Found
echo "999: $(get_status_text 999)"  # Unknown Status
```

### Группировка данных

```bash
#!/bin/bash

declare -A departments

# Данные: имя:отдел
data="
John:Engineering
Alice:Marketing
Bob:Engineering
Carol:Marketing
Dave:Sales
"

# Группировка
while IFS=':' read -r name dept; do
    [[ -z "$name" ]] && continue
    departments[$dept]+="$name,"
done <<< "$data"

# Вывод
for dept in "${!departments[@]}"; do
    # Убираем последнюю запятую
    members="${departments[$dept]%,}"
    echo "$dept: $members"
done
```

### Конвертация кодировок

```bash
#!/bin/bash

declare -A html_entities=(
    ['&']='&amp;'
    ['<']='&lt;'
    ['>']='&gt;'
    ['"']='&quot;'
    ["'"]='&#39;'
)

html_escape() {
    local text=$1
    for char in "${!html_entities[@]}"; do
        text="${text//$char/${html_entities[$char]}}"
    done
    echo "$text"
}

html_escape '<script>alert("XSS")</script>'
# &lt;script&gt;alert(&quot;XSS&quot;)&lt;/script&gt;
```

## Ограничения

```bash
#!/bin/bash

# Ключи - только строки
declare -A arr
arr[123]="value"  # 123 становится строкой "123"

# Нет вложенных ассоциативных массивов
# Но можно симулировать через ключи
declare -A data
data["user.name"]="John"
data["user.age"]="25"
data["user.address.city"]="Moscow"

# Порядок элементов не гарантирован
# Нельзя сортировать как индексированный массив
```

## Проверка версии bash

```bash
#!/bin/bash

if [[ ${BASH_VERSINFO[0]} -lt 4 ]]; then
    echo "Требуется bash 4.0 или выше для ассоциативных массивов"
    exit 1
fi

declare -A my_hash
# ...
```
