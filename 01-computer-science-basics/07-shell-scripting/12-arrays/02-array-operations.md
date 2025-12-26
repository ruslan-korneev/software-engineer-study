# Операции с массивами

## Доступ к элементам

### Отдельный элемент

```bash
#!/bin/bash

arr=("zero" "one" "two" "three" "four")

echo "${arr[0]}"   # zero
echo "${arr[2]}"   # two
echo "${arr[-1]}"  # four (последний, bash 4.2+)
echo "${arr[-2]}"  # three (предпоследний)
```

### Все элементы

```bash
#!/bin/bash

arr=("one" "two" "three")

# Все элементы как отдельные слова (рекомендуется)
echo "${arr[@]}"   # one two three

# Все элементы как одна строка
echo "${arr[*]}"   # one two three

# Разница в кавычках (важно!)
for item in "${arr[@]}"; do  # Правильно
    echo "$item"
done

for item in "${arr[*]}"; do  # Одна итерация со всеми элементами
    echo "$item"
done
```

### Все индексы

```bash
#!/bin/bash

arr=("one" "two" "three")

echo "${!arr[@]}"  # 0 1 2

# Итерация по индексам
for i in "${!arr[@]}"; do
    echo "arr[$i] = ${arr[$i]}"
done
```

## Длина массива и элементов

```bash
#!/bin/bash

arr=("apple" "banana" "cherry")

# Количество элементов
echo "${#arr[@]}"  # 3

# Длина конкретного элемента
echo "${#arr[0]}"  # 5 (длина "apple")
echo "${#arr[1]}"  # 6 (длина "banana")
```

## Добавление элементов

```bash
#!/bin/bash

arr=("one" "two")

# В конец
arr+=("three")
arr+=("four" "five")  # Несколько сразу

echo "${arr[@]}"  # one two three four five

# По индексу
arr[10]="ten"
echo "${arr[10]}"  # ten
```

## Удаление элементов

```bash
#!/bin/bash

arr=("one" "two" "three" "four" "five")

# Удаление по индексу (оставляет дыру)
unset arr[2]
echo "${arr[@]}"       # one two four five
echo "${!arr[@]}"      # 0 1 3 4

# Удаление всего массива
unset arr
```

### Удаление без дыр (переиндексация)

```bash
#!/bin/bash

arr=("one" "two" "three" "four")

# Удаление элемента по значению
new_arr=()
for item in "${arr[@]}"; do
    [[ "$item" != "two" ]] && new_arr+=("$item")
done
arr=("${new_arr[@]}")

echo "${arr[@]}"  # one three four
```

## Срезы (Slicing)

```bash
#!/bin/bash

arr=("zero" "one" "two" "three" "four" "five")

# ${arr[@]:offset:length}
echo "${arr[@]:2}"     # two three four five (с индекса 2 до конца)
echo "${arr[@]:2:3}"   # two three four (3 элемента с индекса 2)
echo "${arr[@]: -2}"   # four five (последние 2)
echo "${arr[@]:0:3}"   # zero one two (первые 3)
```

## Замена в элементах

```bash
#!/bin/bash

arr=("hello world" "hello bash" "hello shell")

# Замена первого вхождения во всех элементах
echo "${arr[@]/hello/hi}"
# hi world hi bash hi shell

# Замена всех вхождений
echo "${arr[@]//l/L}"
# heLLo worLd heLLo bash heLLo sheLL

# Удаление (замена на пустую строку)
echo "${arr[@]//hello /}"
# world bash shell
```

## Поиск в массиве

```bash
#!/bin/bash

arr=("apple" "banana" "cherry" "date")

# Проверка наличия элемента
contains() {
    local item=$1
    shift
    local arr=("$@")

    for element in "${arr[@]}"; do
        [[ "$element" == "$item" ]] && return 0
    done
    return 1
}

if contains "banana" "${arr[@]}"; then
    echo "banana найден"
fi

# Поиск индекса
find_index() {
    local item=$1
    shift
    local arr=("$@")

    for i in "${!arr[@]}"; do
        if [[ "${arr[$i]}" == "$item" ]]; then
            echo $i
            return 0
        fi
    done
    return 1
}

idx=$(find_index "cherry" "${arr[@]}")
echo "Индекс cherry: $idx"  # 2
```

## Сортировка

```bash
#!/bin/bash

arr=("banana" "apple" "date" "cherry")

# Через printf и sort
sorted=($(printf '%s\n' "${arr[@]}" | sort))
echo "${sorted[@]}"  # apple banana cherry date

# Обратная сортировка
sorted=($(printf '%s\n' "${arr[@]}" | sort -r))
echo "${sorted[@]}"  # date cherry banana apple

# Числовая сортировка
nums=(10 2 33 4 15)
sorted=($(printf '%s\n' "${nums[@]}" | sort -n))
echo "${sorted[@]}"  # 2 4 10 15 33
```

## Объединение массивов

```bash
#!/bin/bash

arr1=("one" "two")
arr2=("three" "four")

# Конкатенация
combined=("${arr1[@]}" "${arr2[@]}")
echo "${combined[@]}"  # one two three four
```

## Уникальные элементы

```bash
#!/bin/bash

arr=("apple" "banana" "apple" "cherry" "banana")

# Удаление дубликатов
unique=($(printf '%s\n' "${arr[@]}" | sort -u))
echo "${unique[@]}"  # apple banana cherry

# С сохранением порядка
declare -A seen
unique=()
for item in "${arr[@]}"; do
    if [[ -z "${seen[$item]}" ]]; then
        seen[$item]=1
        unique+=("$item")
    fi
done
echo "${unique[@]}"  # apple banana cherry
```

## Практические примеры

### Обработка файлов

```bash
#!/bin/bash

files=()
shopt -s nullglob

for file in *.txt; do
    files+=("$file")
done

echo "Найдено файлов: ${#files[@]}"
for file in "${files[@]}"; do
    echo "  - $file"
done
```

### Стек (LIFO)

```bash
#!/bin/bash

stack=()

push() { stack+=("$1"); }
pop() {
    if [[ ${#stack[@]} -gt 0 ]]; then
        echo "${stack[-1]}"
        unset stack[-1]
    fi
}
peek() { echo "${stack[-1]}"; }

push "first"
push "second"
push "third"

echo "Peek: $(peek)"  # third
echo "Pop: $(pop)"    # third
echo "Pop: $(pop)"    # second
```

### Очередь (FIFO)

```bash
#!/bin/bash

queue=()

enqueue() { queue+=("$1"); }
dequeue() {
    if [[ ${#queue[@]} -gt 0 ]]; then
        echo "${queue[0]}"
        queue=("${queue[@]:1}")
    fi
}

enqueue "first"
enqueue "second"
enqueue "third"

echo "Dequeue: $(dequeue)"  # first
echo "Dequeue: $(dequeue)"  # second
```

### Фильтрация

```bash
#!/bin/bash

numbers=(1 2 3 4 5 6 7 8 9 10)

# Только чётные
even=()
for n in "${numbers[@]}"; do
    ((n % 2 == 0)) && even+=("$n")
done
echo "Чётные: ${even[@]}"  # 2 4 6 8 10

# Больше 5
greater=()
for n in "${numbers[@]}"; do
    ((n > 5)) && greater+=("$n")
done
echo "Больше 5: ${greater[@]}"  # 6 7 8 9 10
```
