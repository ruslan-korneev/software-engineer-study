# break и continue

## Команда break

`break` немедленно прерывает выполнение цикла:

```bash
#!/bin/bash

for i in {1..10}; do
    if [[ $i -eq 5 ]]; then
        echo "Прерывание на $i"
        break
    fi
    echo "Итерация: $i"
done
echo "После цикла"
```

Вывод:
```
Итерация: 1
Итерация: 2
Итерация: 3
Итерация: 4
Прерывание на 5
После цикла
```

## Команда continue

`continue` пропускает оставшуюся часть текущей итерации и переходит к следующей:

```bash
#!/bin/bash

for i in {1..5}; do
    if [[ $i -eq 3 ]]; then
        echo "Пропускаем $i"
        continue
    fi
    echo "Обработка: $i"
done
```

Вывод:
```
Обработка: 1
Обработка: 2
Пропускаем 3
Обработка: 4
Обработка: 5
```

## break с числовым аргументом

`break N` выходит из N вложенных циклов:

```bash
#!/bin/bash

for i in {1..3}; do
    for j in {1..3}; do
        if [[ $i -eq 2 && $j -eq 2 ]]; then
            echo "Выход из обоих циклов"
            break 2
        fi
        echo "i=$i, j=$j"
    done
done
echo "После циклов"
```

Вывод:
```
i=1, j=1
i=1, j=2
i=1, j=3
i=2, j=1
Выход из обоих циклов
После циклов
```

### break 1 vs break 2

```bash
for outer in {1..3}; do
    for inner in {1..3}; do
        if [[ $inner -eq 2 ]]; then
            break 1  # Выход только из внутреннего цикла
            # break 2  # Выход из обоих циклов
        fi
        echo "$outer-$inner"
    done
    echo "---"
done
```

## continue с числовым аргументом

`continue N` переходит к следующей итерации N-го уровня:

```bash
#!/bin/bash

for i in {1..3}; do
    for j in {1..3}; do
        if [[ $j -eq 2 ]]; then
            continue 2  # Переход к следующей итерации внешнего цикла
        fi
        echo "i=$i, j=$j"
    done
    echo "Конец внутреннего цикла"  # Не выполнится при continue 2
done
```

Вывод:
```
i=1, j=1
i=2, j=1
i=3, j=1
```

## Использование в while и until

```bash
#!/bin/bash

# break в while
count=0
while true; do
    ((count++))
    if [[ $count -gt 5 ]]; then
        break
    fi
    echo $count
done

# continue в until
count=0
until [[ $count -ge 10 ]]; do
    ((count++))
    if [[ $((count % 2)) -eq 0 ]]; then
        continue  # Пропускаем чётные
    fi
    echo $count
done
```

## Практические примеры

### Поиск файла

```bash
#!/bin/bash

target="config.yaml"
found=false

for dir in /etc /home /opt /var; do
    if [[ -f "$dir/$target" ]]; then
        echo "Найден: $dir/$target"
        found=true
        break
    fi
done

if ! $found; then
    echo "Файл не найден"
fi
```

### Пропуск определённых элементов

```bash
#!/bin/bash

# Обработка файлов, пропуская скрытые и директории
for item in /home/user/*; do
    # Пропускаем скрытые
    [[ "$(basename "$item")" == .* ]] && continue

    # Пропускаем директории
    [[ -d "$item" ]] && continue

    echo "Обработка файла: $item"
    # ... обработка ...
done
```

### Валидация с повторным запросом

```bash
#!/bin/bash

while true; do
    read -rp "Введите email: " email

    # Проверка на пустоту
    if [[ -z "$email" ]]; then
        echo "Email не может быть пустым"
        continue
    fi

    # Проверка формата
    if [[ ! "$email" =~ ^[^@]+@[^@]+\.[^@]+$ ]]; then
        echo "Некорректный формат email"
        continue
    fi

    # Если дошли сюда - email корректный
    break
done

echo "Ваш email: $email"
```

### Обработка с пропуском ошибок

```bash
#!/bin/bash

for file in /data/*.csv; do
    # Пропускаем если файл не существует (при пустом glob)
    [[ -e "$file" ]] || continue

    # Пропускаем пустые файлы
    if [[ ! -s "$file" ]]; then
        echo "Пропуск пустого файла: $file"
        continue
    fi

    echo "Обработка: $file"
    # ... обработка ...
done
```

### Поиск во вложенных структурах

```bash
#!/bin/bash

declare -a servers=("server1" "server2" "server3")
declare -a ports=(22 80 443)
target_port=80
found_server=""

for server in "${servers[@]}"; do
    for port in "${ports[@]}"; do
        if nc -z -w1 "$server" "$port" 2>/dev/null; then
            if [[ $port -eq $target_port ]]; then
                echo "Найден $server:$port"
                found_server="$server"
                break 2
            fi
        fi
    done
done

if [[ -n "$found_server" ]]; then
    echo "Используем: $found_server"
fi
```

### Меню с возможностью возврата

```bash
#!/bin/bash

while true; do
    echo "Главное меню:"
    echo "1. Подменю 1"
    echo "2. Подменю 2"
    echo "3. Выход"
    read -rp "Выбор: " main_choice

    case $main_choice in
        1)
            while true; do
                echo "  Подменю 1:"
                echo "  a. Действие A"
                echo "  b. Назад"
                read -rp "  Выбор: " sub_choice

                case $sub_choice in
                    a) echo "  Выполняем A" ;;
                    b) break ;;  # Возврат в главное меню
                esac
            done
            ;;
        2)
            echo "Подменю 2"
            ;;
        3)
            echo "Выход"
            break  # Выход из главного цикла
            ;;
    esac
done
```

## Альтернативы break и continue

### Использование return в функциях

```bash
process_files() {
    for file in "$@"; do
        if [[ ! -f "$file" ]]; then
            echo "Файл не найден: $file"
            return 1  # Выход из функции (не из цикла в вызывающем коде)
        fi
        cat "$file"
    done
}
```

### Использование флагов

```bash
#!/bin/bash

found=false

for item in "${array[@]}"; do
    if [[ "$item" == "$target" ]]; then
        found=true
        # Вместо break можно продолжить для сбора всех совпадений
    fi
done

if $found; then
    echo "Найдено"
fi
```

## Best Practices

1. **Избегайте глубокой вложенности** - используйте функции
2. **break N с осторожностью** - сложно читать при N > 2
3. **Комментируйте break/continue** в сложных циклах
4. **Предпочитайте ранний return** в функциях вместо break

```bash
# Вместо сложной логики с break
process_items() {
    for item in "$@"; do
        # Ранний выход при ошибке
        process_one "$item" || return 1
    done
}

# Вместо
for item in "$@"; do
    if ! process_one "$item"; then
        break
    fi
done
```
