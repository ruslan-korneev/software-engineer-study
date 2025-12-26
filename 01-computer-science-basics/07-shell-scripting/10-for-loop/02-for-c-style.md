# C-style цикл for

## Базовый синтаксис

```bash
for ((init; condition; step)); do
    commands
done
```

Это синтаксис, знакомый программистам C/C++/Java:
- `init` - инициализация (выполняется один раз)
- `condition` - условие продолжения
- `step` - действие после каждой итерации

## Простые примеры

### Счётчик от 0 до 4

```bash
#!/bin/bash

for ((i = 0; i < 5; i++)); do
    echo "i = $i"
done
```

Вывод:
```
i = 0
i = 1
i = 2
i = 3
i = 4
```

### Обратный отсчёт

```bash
#!/bin/bash

for ((i = 10; i >= 0; i--)); do
    echo "Осталось: $i"
done
echo "Старт!"
```

### С шагом

```bash
#!/bin/bash

# Чётные числа
for ((i = 0; i <= 10; i += 2)); do
    echo $i
done

# С шагом 5
for ((i = 0; i <= 100; i += 5)); do
    echo $i
done
```

## Особенности синтаксиса

### Без знака $ внутри (( ))

```bash
#!/bin/bash

# Внутри (( )) $ не нужен
for ((i = 0; i < 5; i++)); do
    echo $i  # Но снаружи нужен
done
```

### Пустые части

```bash
#!/bin/bash

# Инициализация снаружи
i=0
for ((; i < 5; i++)); do
    echo $i
done

# Шаг внутри тела
for ((i = 0; i < 5; )); do
    echo $i
    ((i++))
done

# Бесконечный цикл
for ((; ; )); do
    echo "Бесконечный..."
    sleep 1
done
```

### Множественные переменные

```bash
#!/bin/bash

# Две переменных в одном цикле
for ((i = 0, j = 10; i < 5; i++, j--)); do
    echo "i=$i, j=$j"
done
```

Вывод:
```
i=0, j=10
i=1, j=9
i=2, j=8
i=3, j=7
i=4, j=6
```

## Сравнение с традиционным for

```bash
#!/bin/bash

# Традиционный for - итерация по списку
for i in {1..5}; do
    echo $i
done

# C-style for - числовые итерации
for ((i = 1; i <= 5; i++)); do
    echo $i
done

# Оба выводят одно и то же, но:
# - Brace expansion {1..1000} генерирует весь список сразу
# - C-style вычисляет значения по мере необходимости
```

## Практические примеры

### Обработка массива по индексу

```bash
#!/bin/bash

arr=("apple" "banana" "cherry" "date")

for ((i = 0; i < ${#arr[@]}; i++)); do
    echo "[$i] = ${arr[$i]}"
done
```

### Обход с конца

```bash
#!/bin/bash

arr=("one" "two" "three" "four" "five")

for ((i = ${#arr[@]} - 1; i >= 0; i--)); do
    echo "${arr[$i]}"
done
```

### Обработка строк в цикле

```bash
#!/bin/bash

text="Hello, World!"

for ((i = 0; i < ${#text}; i++)); do
    char="${text:$i:1}"
    echo "Символ $i: '$char'"
done
```

### Генерация файлов с нумерацией

```bash
#!/bin/bash

for ((i = 1; i <= 10; i++)); do
    # printf для ведущих нулей
    filename=$(printf "file_%03d.txt" $i)
    touch "$filename"
    echo "Создан: $filename"
done
```

### Прогресс-бар

```bash
#!/bin/bash

total=50

for ((i = 0; i <= total; i++)); do
    # Вычисление процентов
    percent=$((i * 100 / total))

    # Формирование бара
    bar=""
    for ((j = 0; j < i; j++)); do
        bar+="="
    done
    for ((j = i; j < total; j++)); do
        bar+=" "
    done

    # Вывод с возвратом каретки
    printf "\r[%s] %d%%" "$bar" "$percent"
    sleep 0.1
done
echo
```

### Вложенные C-style циклы

```bash
#!/bin/bash

# Таблица умножения
for ((i = 1; i <= 10; i++)); do
    for ((j = 1; j <= 10; j++)); do
        printf "%4d" $((i * j))
    done
    echo
done
```

### Двумерный массив (симуляция)

```bash
#!/bin/bash

# "Двумерный массив" через одномерный
rows=3
cols=4
declare -a matrix

# Заполнение
for ((i = 0; i < rows; i++)); do
    for ((j = 0; j < cols; j++)); do
        matrix[$((i * cols + j))]=$((i + j))
    done
done

# Вывод
for ((i = 0; i < rows; i++)); do
    for ((j = 0; j < cols; j++)); do
        printf "%3d" "${matrix[$((i * cols + j))]}"
    done
    echo
done
```

## Арифметические выражения в условии

```bash
#!/bin/bash

# Любое арифметическое выражение
for ((i = 1; i * i <= 100; i++)); do
    echo "$i^2 = $((i * i))"
done
```

## break и continue

```bash
#!/bin/bash

# break
for ((i = 0; i < 10; i++)); do
    if ((i == 5)); then
        break
    fi
    echo $i
done

# continue
for ((i = 0; i < 10; i++)); do
    if ((i % 2 == 0)); then
        continue
    fi
    echo $i  # Только нечётные
done
```

## Сравнение производительности

```bash
#!/bin/bash

# Для больших диапазонов C-style эффективнее
time for i in {1..100000}; do :; done
time for ((i = 1; i <= 100000; i++)); do :; done

# Brace expansion создаёт список в памяти
# C-style вычисляет значения на лету
```

## Когда использовать C-style

1. **Числовые итерации** с известным диапазоном
2. **Индексный доступ** к массивам
3. **Сложные условия** продолжения цикла
4. **Множественные переменные** цикла
5. **Большие диапазоны** (эффективнее по памяти)

## Когда использовать традиционный for

1. **Итерация по списку** значений
2. **Обход файлов** по glob-паттерну
3. **Обработка аргументов** `"$@"`
4. **Итерация по массиву** `"${array[@]}"`
