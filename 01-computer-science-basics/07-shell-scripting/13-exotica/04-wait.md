# Команда wait

## Что такое wait

Команда `wait` приостанавливает выполнение скрипта до завершения фоновых процессов. Используется для:
- Синхронизации параллельных задач
- Получения кода возврата фоновых процессов
- Ожидания завершения дочерних процессов

## Базовое использование

```bash
#!/bin/bash

# Запуск фонового процесса
sleep 5 &

echo "Процесс запущен"

# Ожидание завершения всех фоновых процессов
wait

echo "Процесс завершён"
```

## Переменная $!

`$!` содержит PID последнего фонового процесса:

```bash
#!/bin/bash

sleep 10 &
pid=$!

echo "Запущен процесс с PID: $pid"

wait $pid
echo "Процесс $pid завершён"
```

## Варианты использования

### wait без аргументов

Ожидает ВСЕ дочерние процессы:

```bash
#!/bin/bash

sleep 2 &
sleep 3 &
sleep 1 &

echo "Все процессы запущены"
wait
echo "Все процессы завершены"
```

### wait с PID

Ожидает конкретный процесс:

```bash
#!/bin/bash

sleep 5 &
pid1=$!

sleep 3 &
pid2=$!

wait $pid1
echo "Первый процесс завершён"

wait $pid2
echo "Второй процесс завершён"
```

### wait с несколькими PID

```bash
#!/bin/bash

sleep 2 &
pid1=$!

sleep 3 &
pid2=$!

# Ожидание нескольких конкретных процессов
wait $pid1 $pid2
echo "Оба процесса завершены"
```

## Получение кода возврата

```bash
#!/bin/bash

(
    sleep 1
    exit 42
) &
pid=$!

wait $pid
exit_code=$?

echo "Процесс завершился с кодом: $exit_code"  # 42
```

### wait -n (bash 4.3+)

Ожидает завершения ЛЮБОГО фонового процесса:

```bash
#!/bin/bash

sleep 5 &
pid1=$!

sleep 1 &
pid2=$!

sleep 3 &
pid3=$!

# Ждём первый завершившийся
wait -n
echo "Один из процессов завершился"

# Ждём ещё один
wait -n
echo "Ещё один завершился"

# Ждём последний
wait -n
echo "Последний завершился"
```

## Практические примеры

### Параллельная обработка файлов

```bash
#!/bin/bash

process_file() {
    local file=$1
    echo "Обработка: $file"
    sleep $((RANDOM % 5))  # Имитация работы
    echo "Завершено: $file"
}

# Параллельный запуск
for file in *.txt; do
    process_file "$file" &
done

# Ожидание всех
wait
echo "Все файлы обработаны"
```

### Ограничение параллельности

```bash
#!/bin/bash

MAX_JOBS=3
pids=()

run_job() {
    local job=$1
    echo "Запуск задачи $job"
    sleep $((RANDOM % 5))
    echo "Задача $job завершена"
}

for i in {1..10}; do
    # Ждём, если достигнут лимит
    while [[ ${#pids[@]} -ge $MAX_JOBS ]]; do
        # Проверяем, какие процессы завершились
        for idx in "${!pids[@]}"; do
            if ! kill -0 "${pids[$idx]}" 2>/dev/null; then
                wait "${pids[$idx]}"
                unset 'pids[$idx]'
            fi
        done
        sleep 0.1
    done

    run_job $i &
    pids+=($!)
done

wait
echo "Все задачи завершены"
```

### Сбор результатов

```bash
#!/bin/bash

# Временные файлы для результатов
declare -a result_files

process_and_save() {
    local id=$1
    local result_file=$(mktemp)
    echo "$result_file"

    # Записываем результат
    sleep $((RANDOM % 3))
    echo "Result from process $id" > "$result_file"
}

# Запуск процессов
for i in {1..5}; do
    result_file=$(mktemp)
    result_files+=("$result_file")
    (
        sleep $((RANDOM % 3))
        echo "Result from process $i" > "$result_file"
    ) &
done

wait

# Сбор результатов
echo "=== Результаты ==="
for file in "${result_files[@]}"; do
    cat "$file"
    rm "$file"
done
```

### Обработка с таймаутом

```bash
#!/bin/bash

with_timeout() {
    local timeout=$1
    shift
    local cmd="$@"

    $cmd &
    local pid=$!

    (
        sleep $timeout
        kill $pid 2>/dev/null
    ) &
    local killer=$!

    wait $pid 2>/dev/null
    local exit_code=$?

    kill $killer 2>/dev/null
    wait $killer 2>/dev/null

    return $exit_code
}

if with_timeout 5 sleep 10; then
    echo "Завершено вовремя"
else
    echo "Таймаут или ошибка"
fi
```

### Проверка статуса процессов

```bash
#!/bin/bash

declare -A jobs

# Запуск задач
jobs[task1]=$( (sleep 2; exit 0) & echo $! )
jobs[task2]=$( (sleep 3; exit 1) & echo $! )
jobs[task3]=$( (sleep 1; exit 0) & echo $! )

# Ожидание и проверка
for name in "${!jobs[@]}"; do
    wait "${jobs[$name]}"
    code=$?
    if [[ $code -eq 0 ]]; then
        echo "$name: OK"
    else
        echo "$name: FAILED (code $code)"
    fi
done
```

### Цепочка зависимых задач

```bash
#!/bin/bash

step1() {
    echo "Step 1: Загрузка данных"
    sleep 2
}

step2() {
    echo "Step 2: Обработка"
    sleep 3
}

step3() {
    echo "Step 3: Сохранение"
    sleep 1
}

# Запуск step1
step1 &
pid1=$!

wait $pid1
echo "Step 1 завершён"

# Запуск step2 (зависит от step1)
step2 &
pid2=$!

# step3 не зависит от step2
step3 &
pid3=$!

wait $pid2 $pid3
echo "Все шаги завершены"
```

## Обработка ошибок

```bash
#!/bin/bash

set -e

run_task() {
    local name=$1
    local should_fail=$2

    if $should_fail; then
        exit 1
    fi
    exit 0
}

# Запуск задач
run_task "task1" false &
pid1=$!

run_task "task2" true &
pid2=$!

# Проверка результатов
failed=false

wait $pid1 || { echo "task1 failed"; failed=true; }
wait $pid2 || { echo "task2 failed"; failed=true; }

if $failed; then
    echo "Некоторые задачи завершились с ошибкой"
    exit 1
fi
```
