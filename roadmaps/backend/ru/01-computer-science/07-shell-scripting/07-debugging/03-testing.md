# Тестирование скриптов

[prev: 02-logical-errors](./02-logical-errors.md) | [next: 04-tracing](./04-tracing.md)

---
## Зачем тестировать

- Раннее обнаружение ошибок
- Уверенность при изменениях
- Документация поведения
- Упрощение отладки

## Ручное тестирование

### Тестовые данные

```bash
# Создание тестовых файлов
mkdir -p /tmp/test_data
echo "test content" > /tmp/test_data/file1.txt
echo "another test" > /tmp/test_data/file2.txt

# Запуск скрипта с тестовыми данными
./my_script.sh /tmp/test_data

# Очистка
rm -rf /tmp/test_data
```

### Тестирование с разными входными данными

```bash
# Нормальный ввод
./script.sh "normal input"

# Пустой ввод
./script.sh ""

# Специальные символы
./script.sh "hello world"
./script.sh "test\$var"
./script.sh "file with spaces.txt"

# Граничные значения
./script.sh 0
./script.sh -1
./script.sh 999999999
```

## Встроенные тесты

### Функция тестирования

```bash
#!/bin/bash

# Цвета для вывода
RED='\033[0;31m'
GREEN='\033[0;32m'
NC='\033[0m'

# Счётчики
tests_passed=0
tests_failed=0

# Функция assert
assert_equals() {
    local expected=$1
    local actual=$2
    local message=${3:-""}

    if [[ "$expected" == "$actual" ]]; then
        echo -e "${GREEN}PASS${NC}: $message"
        ((tests_passed++))
    else
        echo -e "${RED}FAIL${NC}: $message"
        echo "  Expected: '$expected'"
        echo "  Actual:   '$actual'"
        ((tests_failed++))
    fi
}

assert_true() {
    local condition=$1
    local message=${2:-""}

    if eval "$condition"; then
        echo -e "${GREEN}PASS${NC}: $message"
        ((tests_passed++))
    else
        echo -e "${RED}FAIL${NC}: $message"
        ((tests_failed++))
    fi
}

assert_false() {
    local condition=$1
    local message=${2:-""}

    if ! eval "$condition"; then
        echo -e "${GREEN}PASS${NC}: $message"
        ((tests_passed++))
    else
        echo -e "${RED}FAIL${NC}: $message"
        ((tests_failed++))
    fi
}

# Вывод итогов
print_summary() {
    echo ""
    echo "================================"
    echo "Tests passed: $tests_passed"
    echo "Tests failed: $tests_failed"
    echo "================================"

    [[ $tests_failed -eq 0 ]]
}
```

### Пример использования

```bash
#!/bin/bash

source ./test_helpers.sh

# Тестируемая функция
add() {
    echo $(($1 + $2))
}

# Тесты
test_add() {
    assert_equals "5" "$(add 2 3)" "2 + 3 = 5"
    assert_equals "0" "$(add 0 0)" "0 + 0 = 0"
    assert_equals "-3" "$(add -1 -2)" "-1 + -2 = -3"
    assert_equals "100" "$(add 50 50)" "50 + 50 = 100"
}

test_add
print_summary
```

## Тестирование функций

### Изоляция функций для тестирования

```bash
#!/bin/bash
# lib.sh - библиотека функций

is_valid_email() {
    [[ "$1" =~ ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$ ]]
}

is_positive_number() {
    [[ "$1" =~ ^[0-9]+$ ]] && [[ "$1" -gt 0 ]]
}
```

```bash
#!/bin/bash
# test_lib.sh - тесты

source ./lib.sh
source ./test_helpers.sh

test_is_valid_email() {
    assert_true 'is_valid_email "user@example.com"' "valid email"
    assert_true 'is_valid_email "user.name@domain.co.uk"' "complex email"
    assert_false 'is_valid_email "invalid"' "no @ symbol"
    assert_false 'is_valid_email "@example.com"' "no local part"
    assert_false 'is_valid_email "user@"' "no domain"
    assert_false 'is_valid_email ""' "empty string"
}

test_is_positive_number() {
    assert_true 'is_positive_number "1"' "1 is positive"
    assert_true 'is_positive_number "100"' "100 is positive"
    assert_false 'is_positive_number "0"' "0 is not positive"
    assert_false 'is_positive_number "-1"' "negative number"
    assert_false 'is_positive_number "abc"' "not a number"
    assert_false 'is_positive_number ""' "empty string"
}

# Запуск тестов
test_is_valid_email
test_is_positive_number
print_summary
```

## Тестирование скрипта целиком

### Тест с захватом вывода

```bash
#!/bin/bash

test_script_output() {
    local output
    local exit_code

    # Запуск скрипта и захват вывода
    output=$(./my_script.sh "test_input" 2>&1)
    exit_code=$?

    # Проверка кода возврата
    assert_equals "0" "$exit_code" "exit code should be 0"

    # Проверка вывода
    assert_true '[[ "$output" == *"expected text"* ]]' "output contains expected text"
}
```

### Тест с файлами

```bash
#!/bin/bash

test_file_processing() {
    local temp_dir
    temp_dir=$(mktemp -d)

    # Setup - создание тестовых данных
    echo "line1" > "$temp_dir/input.txt"
    echo "line2" >> "$temp_dir/input.txt"

    # Execute - запуск тестируемого скрипта
    ./process_file.sh "$temp_dir/input.txt" "$temp_dir/output.txt"

    # Verify - проверка результата
    assert_true '[[ -f "$temp_dir/output.txt" ]]' "output file exists"

    local line_count
    line_count=$(wc -l < "$temp_dir/output.txt")
    assert_equals "2" "$line_count" "output has 2 lines"

    # Cleanup - очистка
    rm -rf "$temp_dir"
}
```

## Фреймворки для тестирования

### Bats (Bash Automated Testing System)

```bash
# Установка
npm install -g bats

# или
git clone https://github.com/bats-core/bats-core.git
cd bats-core
./install.sh /usr/local
```

Пример теста (test.bats):

```bash
#!/usr/bin/env bats

@test "addition using bc" {
    result="$(echo 2+2 | bc)"
    [ "$result" -eq 4 ]
}

@test "script returns 0 on success" {
    run ./my_script.sh valid_input
    [ "$status" -eq 0 ]
}

@test "script returns 1 on invalid input" {
    run ./my_script.sh invalid_input
    [ "$status" -eq 1 ]
}

@test "output contains expected string" {
    run ./my_script.sh test
    [ "${lines[0]}" = "Expected first line" ]
}
```

```bash
# Запуск тестов
bats test.bats
```

### shUnit2

```bash
#!/bin/bash

# Подключение shUnit2
. /path/to/shunit2

# Функция setup выполняется перед каждым тестом
setUp() {
    TEST_DIR=$(mktemp -d)
}

# Функция teardown выполняется после каждого теста
tearDown() {
    rm -rf "$TEST_DIR"
}

# Тесты начинаются с test
testAddition() {
    result=$(( 2 + 2 ))
    assertEquals 4 $result
}

testFileCreation() {
    touch "$TEST_DIR/test.txt"
    assertTrue "[ -f '$TEST_DIR/test.txt' ]"
}

testStringEquality() {
    assertEquals "hello" "hello"
}
```

## Непрерывное тестирование

### Makefile для тестов

```makefile
.PHONY: test lint all

all: lint test

lint:
	shellcheck *.sh

test:
	./run_tests.sh

watch:
	while true; do \
		inotifywait -e modify *.sh; \
		make test; \
	done
```

### GitHub Actions для тестирования

```yaml
# .github/workflows/test.yml
name: Test Shell Scripts

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Install ShellCheck
        run: sudo apt-get install -y shellcheck

      - name: Run ShellCheck
        run: shellcheck *.sh

      - name: Install bats
        run: npm install -g bats

      - name: Run tests
        run: bats tests/*.bats
```

## Best Practices

1. **Тестируйте граничные случаи** - пустой ввод, нулевые значения, отрицательные числа
2. **Используйте временные директории** для файловых тестов
3. **Всегда очищайте** тестовые данные (setup/teardown)
4. **Именуйте тесты описательно** - что тестируется и ожидаемый результат
5. **Автоматизируйте запуск тестов** - CI/CD, git hooks
6. **Изолируйте тесты** - каждый тест должен быть независимым

---

[prev: 02-logical-errors](./02-logical-errors.md) | [next: 04-tracing](./04-tracing.md)
