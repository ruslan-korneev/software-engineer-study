# Bisect

[prev: 01-reflog](./01-reflog.md) | [next: 03-worktree](./03-worktree.md)
---

## Что такое Git Bisect

**Git bisect** - это мощный инструмент для поиска коммита, который внес баг в проект. Он использует алгоритм бинарного поиска, что позволяет быстро найти проблемный коммит даже среди тысяч изменений.

### Принцип работы

1. Вы указываете "плохой" коммит (где баг есть) и "хороший" (где бага не было)
2. Git переключается на коммит посередине
3. Вы проверяете, есть ли баг
4. Git сужает диапазон поиска вдвое
5. Процесс повторяется до нахождения первого "плохого" коммита

### Эффективность бинарного поиска

| Количество коммитов | Максимум проверок |
|---------------------|-------------------|
| 8                   | 3                 |
| 16                  | 4                 |
| 128                 | 7                 |
| 1024                | 10                |
| 4096                | 12                |

## Основные команды

### Запуск bisect

```bash
# Начать сессию bisect
git bisect start

# Указать плохой коммит (текущий, с багом)
git bisect bad

# Указать хороший коммит (где бага точно не было)
git bisect good v1.0.0
# или
git bisect good abc1234

# Можно указать сразу при старте
git bisect start HEAD v1.0.0
# Эквивалентно:
# git bisect start
# git bisect bad HEAD
# git bisect good v1.0.0
```

### Процесс поиска

```bash
# После start, bad и good Git переключается на средний коммит
# Bisecting: 64 revisions left to test after this (roughly 6 steps)
# [abc1234...] Some commit message

# Проверяете код и сообщаете результат:

# Если баг присутствует
git bisect bad

# Если бага нет
git bisect good

# Если невозможно проверить (например, код не компилируется)
git bisect skip
```

### Завершение bisect

```bash
# Когда виновник найден:
# abc1234 is the first bad commit
# commit abc1234
# Author: Developer <dev@example.com>
# Date:   Mon Dec 25 10:00:00 2023
#
#     Add feature X
#
# :100644 100644 abc... def... M  src/feature.js

# Завершить сессию и вернуться к исходной ветке
git bisect reset

# Вернуться к конкретному коммиту вместо исходного
git bisect reset HEAD~3
```

## Автоматизация с git bisect run

### Автоматический поиск с тестовым скриптом

```bash
# Запуск с автоматической проверкой
git bisect start HEAD v1.0.0
git bisect run ./test-script.sh
```

### Правила для тестового скрипта

Скрипт должен возвращать:
- **0** - коммит хороший (тест прошел)
- **1-124, 126-127** - коммит плохой (тест провален)
- **125** - пропустить коммит (невозможно проверить)

### Пример тестового скрипта

```bash
#!/bin/bash
# test-bug.sh

# Собрать проект
make clean && make
if [ $? -ne 0 ]; then
    # Не удалось собрать - пропускаем
    exit 125
fi

# Запустить конкретный тест
./run-tests.sh test_login
if [ $? -ne 0 ]; then
    # Тест провален - плохой коммит
    exit 1
fi

# Тест прошел - хороший коммит
exit 0
```

### Использование с pytest

```bash
# Проверка конкретного теста
git bisect start HEAD v1.0.0
git bisect run pytest tests/test_auth.py::test_login -x

# С установкой зависимостей
git bisect run sh -c 'pip install -e . && pytest tests/test_specific.py -x'
```

### Использование с npm/node

```bash
# Поиск бага в JavaScript проекте
git bisect start HEAD v2.0.0
git bisect run sh -c 'npm install && npm test'

# Проверка конкретного теста
git bisect run sh -c 'npm install && npm test -- --grep "specific test"'
```

## Практические примеры

### Пример 1: Ручной поиск бага в UI

```bash
# Начинаем поиск
git bisect start

# Текущая версия сломана
git bisect bad

# Версия месячной давности работала
git bisect good HEAD~50

# Git переключается на средний коммит
# Bisecting: 25 revisions left to test after this

# Открываем приложение, проверяем UI
# Баг есть - сообщаем
git bisect bad

# Git переключается дальше...
# Bisecting: 12 revisions left to test

# Проверяем - бага нет
git bisect good

# Продолжаем пока не найдем виновника
# ...

# Готово!
# commit abc1234 is the first bad commit

# Смотрим что изменилось
git show abc1234

# Завершаем
git bisect reset
```

### Пример 2: Автоматический поиск регрессии производительности

```bash
#!/bin/bash
# perf-test.sh

# Собираем проект
npm run build 2>/dev/null || exit 125

# Запускаем бенчмарк
RESULT=$(node benchmark.js | grep "time:" | awk '{print $2}')

# Если время выполнения > 1000ms - это регрессия
if [ "$RESULT" -gt 1000 ]; then
    exit 1
fi

exit 0
```

```bash
# Запуск
git bisect start HEAD v1.0.0
git bisect run ./perf-test.sh
```

### Пример 3: Поиск коммита с синтаксической ошибкой

```bash
# Найти коммит, который сломал компиляцию Python
git bisect start HEAD v1.0.0
git bisect run python -m py_compile src/main.py
```

### Пример 4: Поиск изменения в выводе программы

```bash
#!/bin/bash
# check-output.sh

OUTPUT=$(./my-program --version 2>/dev/null)
if [ $? -ne 0 ]; then
    exit 125  # Не удалось запустить
fi

# Проверяем ожидаемый формат вывода
if echo "$OUTPUT" | grep -q "MyProgram v"; then
    exit 0  # Хороший коммит
else
    exit 1  # Плохой коммит - формат изменился
fi
```

## Дополнительные возможности

### Просмотр журнала bisect

```bash
# Показать историю проверок
git bisect log

# Вывод:
# git bisect start
# git bisect bad abc1234
# git bisect good def5678
# git bisect bad ghi9012
# ...
```

### Сохранение и воспроизведение сессии

```bash
# Сохранить журнал в файл
git bisect log > bisect.log

# Воспроизвести сессию
git bisect replay bisect.log
```

### Визуализация оставшихся коммитов

```bash
# Показать коммиты в диапазоне поиска
git bisect visualize

# Или в виде лога
git bisect visualize --oneline
```

### Терминология: old/new вместо good/bad

```bash
# Для случаев, когда ищем не баг, а изменение поведения
git bisect start --term-old=before --term-new=after

# Или использовать встроенные альтернативы
git bisect start
git bisect new  # вместо bad
git bisect old  # вместо good
```

## Работа с merge коммитами

### Особенности

```bash
# При bisect Git по умолчанию проверяет и merge коммиты
# Если merge коммит проблематичный, можно пропустить
git bisect skip

# Для проверки только first-parent (основная ветка)
git bisect start --first-parent HEAD v1.0.0
```

## Советы и лучшие практики

### Подготовка к bisect

1. **Убедитесь, что можете надежно воспроизвести баг**
2. **Выберите хороший "хороший" коммит** - чем ближе к плохому, тем быстрее поиск
3. **Подготовьте тестовый скрипт** для автоматизации, если возможно

### Распространенные проблемы

```bash
# Если забыли в какой ветке были
git bisect reset

# Если нужно прервать bisect без reset
git bisect reset HEAD

# Если коммит невозможно протестировать
git bisect skip

# Пропустить диапазон коммитов
git bisect skip abc1234..def5678
```

### Оптимизация для больших репозиториев

```bash
# Ограничить поиск определенными путями
git bisect start HEAD v1.0.0 -- src/specific/path/

# Это ускорит поиск, игнорируя изменения в других директориях
```

## Полезные алиасы

```bash
# Быстрый старт bisect
git config --global alias.bs "bisect start"
git config --global alias.bb "bisect bad"
git config --global alias.bg "bisect good"
git config --global alias.br "bisect reset"

# Показать журнал текущей сессии
git config --global alias.bl "bisect log"
```

## Заключение

Git bisect - незаменимый инструмент для поиска регрессий. Вместо ручного просмотра десятков или сотен коммитов, bisect находит проблемный коммит за логарифмическое время. Особенно мощен в сочетании с автоматическими тестами - `git bisect run` может найти баг без вашего участия, пока вы пьете кофе.

---
[prev: 01-reflog](./01-reflog.md) | [next: 03-worktree](./03-worktree.md)