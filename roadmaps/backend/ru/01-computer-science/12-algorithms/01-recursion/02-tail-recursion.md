# Хвостовая рекурсия (Tail Recursion)

[prev: 01-recursion-basics](./01-recursion-basics.md) | [next: 03-recursion-vs-iteration](./03-recursion-vs-iteration.md)
---

## Определение

**Хвостовая рекурсия** — это особая форма рекурсии, при которой рекурсивный вызов является **последней операцией** в функции. После возврата из рекурсивного вызова не выполняется никаких дополнительных вычислений.

Ключевое отличие: результат рекурсивного вызова напрямую возвращается, без дополнительной обработки.

## Зачем нужна хвостовая рекурсия

### Преимущества:
- **Оптимизация компилятором (TCO — Tail Call Optimization)**: компилятор может преобразовать рекурсию в цикл
- **Константное использование памяти**: O(1) вместо O(n) для стека вызовов
- **Предотвращение Stack Overflow**: безопасна для глубокой рекурсии
- **Эффективность**: работает так же быстро, как итеративная версия

### Языки с поддержкой TCO:
- Scheme, Lisp (гарантирована)
- Scala, Kotlin (частичная)
- JavaScript (ES6, но не все движки)
- **Python: НЕ поддерживает TCO!**

## Как работает хвостовая рекурсия

### Сравнение обычной и хвостовой рекурсии

```
ОБЫЧНАЯ РЕКУРСИЯ (факториал):

def factorial(n):
    if n <= 1:
        return 1
    return n * factorial(n-1)  # После вызова ещё умножение на n!

Стек вызовов:
factorial(4) ожидает factorial(3)
  factorial(3) ожидает factorial(2)
    factorial(2) ожидает factorial(1)
      factorial(1) = 1
    return 2 * 1 = 2
  return 3 * 2 = 6
return 4 * 6 = 24

Память: O(n) — каждый вызов ждёт в стеке

---

ХВОСТОВАЯ РЕКУРСИЯ (факториал):

def factorial_tail(n, accumulator=1):
    if n <= 1:
        return accumulator
    return factorial_tail(n-1, n * accumulator)  # Последняя операция!

Стек вызовов (с TCO):
factorial_tail(4, 1)
→ factorial_tail(3, 4)
→ factorial_tail(2, 12)
→ factorial_tail(1, 24)
→ 24

Память: O(1) — предыдущий кадр можно переиспользовать
```

### Визуализация разницы

```
ОБЫЧНАЯ РЕКУРСИЯ:                    ХВОСТОВАЯ РЕКУРСИЯ (с TCO):

┌─────────────────┐                  ┌─────────────────┐
│ factorial(4)    │                  │ factorial_tail  │
│   ожидает...    │                  │ (4, 1)          │
│ ┌─────────────┐ │                  └─────────────────┘
│ │ factorial(3)│ │                          ↓
│ │   ожидает...│ │                  ┌─────────────────┐
│ │ ┌─────────┐ │ │                  │ factorial_tail  │
│ │ │ fact(2) │ │ │                  │ (3, 4)          │
│ │ │ ожид... │ │ │                  └─────────────────┘
│ │ │ ┌─────┐ │ │ │                          ↓
│ │ │ │f(1) │ │ │ │        ══►       ┌─────────────────┐
│ │ │ │ =1  │ │ │ │                  │ factorial_tail  │
│ │ │ └─────┘ │ │ │                  │ (2, 12)         │
│ │ └─────────┘ │ │                  └─────────────────┘
│ └─────────────┘ │                          ↓
└─────────────────┘                  ┌─────────────────┐
                                     │ factorial_tail  │
Память: O(n)                         │ (1, 24) → 24    │
                                     └─────────────────┘

                                     Память: O(1)
```

## Псевдокод

### Паттерн преобразования в хвостовую рекурсию:

```
# Исходная функция
function regular_recursive(n):
    if base_case:
        return base_value
    return some_operation(n, regular_recursive(n-1))

# Хвостовая версия с аккумулятором
function tail_recursive(n, accumulator=initial_value):
    if base_case:
        return accumulator  # Возвращаем накопленный результат
    return tail_recursive(n-1, update(accumulator, n))  # Последняя операция
```

### Общий принцип:
1. Добавить параметр-аккумулятор
2. Перенести операцию в обновление аккумулятора
3. В базовом случае вернуть аккумулятор

## Примеры преобразования

### Факториал

```python
# Обычная рекурсия
def factorial(n):
    if n <= 1:
        return 1
    return n * factorial(n - 1)

# Хвостовая рекурсия
def factorial_tail(n, acc=1):
    if n <= 1:
        return acc
    return factorial_tail(n - 1, n * acc)
```

### Сумма списка

```python
# Обычная рекурсия
def sum_list(arr):
    if not arr:
        return 0
    return arr[0] + sum_list(arr[1:])

# Хвостовая рекурсия
def sum_list_tail(arr, acc=0):
    if not arr:
        return acc
    return sum_list_tail(arr[1:], acc + arr[0])
```

### Числа Фибоначчи

```python
# Обычная рекурсия O(2^n)
def fib(n):
    if n <= 1:
        return n
    return fib(n-1) + fib(n-2)

# Хвостовая рекурсия O(n)
def fib_tail(n, a=0, b=1):
    if n == 0:
        return a
    if n == 1:
        return b
    return fib_tail(n - 1, b, a + b)
```

### Обратный порядок списка

```python
# Обычная рекурсия
def reverse(arr):
    if len(arr) <= 1:
        return arr
    return reverse(arr[1:]) + [arr[0]]

# Хвостовая рекурсия
def reverse_tail(arr, acc=None):
    if acc is None:
        acc = []
    if not arr:
        return acc
    return reverse_tail(arr[1:], [arr[0]] + acc)
```

## Анализ сложности

| Алгоритм | Обычная рекурсия | Хвостовая рекурсия (с TCO) |
|----------|------------------|---------------------------|
| Факториал | Время: O(n), Память: O(n) | Время: O(n), Память: O(1) |
| Фибоначчи | Время: O(2^n), Память: O(n) | Время: O(n), Память: O(1) |
| Сумма списка | Время: O(n), Память: O(n) | Время: O(n), Память: O(1) |

**Важно**: Без TCO (как в Python) хвостовая рекурсия всё равно использует O(n) памяти!

## Пример с пошаговым разбором

### Вычисление factorial_tail(5)

```python
def factorial_tail(n, acc=1):
    if n <= 1:
        return acc
    return factorial_tail(n - 1, n * acc)
```

### Пошаговое выполнение:

```
Шаг 1: factorial_tail(5, 1)
       n=5 > 1, поэтому вызываем factorial_tail(4, 5*1=5)

Шаг 2: factorial_tail(4, 5)
       n=4 > 1, поэтому вызываем factorial_tail(3, 4*5=20)

Шаг 3: factorial_tail(3, 20)
       n=3 > 1, поэтому вызываем factorial_tail(2, 3*20=60)

Шаг 4: factorial_tail(2, 60)
       n=2 > 1, поэтому вызываем factorial_tail(1, 2*60=120)

Шаг 5: factorial_tail(1, 120)
       n=1 <= 1, возвращаем acc=120

Результат: 120
```

### Сравнение с обычной рекурсией:

```
Обычная:                         Хвостовая:
factorial(5)                     factorial_tail(5, 1)
= 5 * factorial(4)               = factorial_tail(4, 5)
= 5 * 4 * factorial(3)           = factorial_tail(3, 20)
= 5 * 4 * 3 * factorial(2)       = factorial_tail(2, 60)
= 5 * 4 * 3 * 2 * factorial(1)   = factorial_tail(1, 120)
= 5 * 4 * 3 * 2 * 1              = 120
= 120

Вычисления происходят:
- При ВОЗВРАТЕ (обычная)         - При ВЫЗОВЕ (хвостовая)
```

## Типичные ошибки и Edge Cases

### 1. Ошибка: Операция после рекурсивного вызова

```python
# НЕ хвостовая рекурсия!
def not_tail(n, acc=1):
    if n <= 1:
        return acc
    result = not_tail(n - 1, n * acc)
    return result + 0  # Операция после вызова — НЕ хвостовая!

# Хвостовая рекурсия
def is_tail(n, acc=1):
    if n <= 1:
        return acc
    return is_tail(n - 1, n * acc)  # Ничего после вызова
```

### 2. Ошибка: Python не оптимизирует хвостовую рекурсию

```python
import sys
sys.setrecursionlimit(10000)

# Даже хвостовая рекурсия вызовет Stack Overflow в Python
def count_down_tail(n):
    if n == 0:
        return "Done"
    return count_down_tail(n - 1)

# count_down_tail(100000)  # RecursionError!

# Решение: использовать трамплин или итерацию
def count_down_iter(n):
    while n > 0:
        n -= 1
    return "Done"
```

### 3. Трамплин для эмуляции TCO в Python

```python
def trampoline(fn, *args):
    """Эмулирует TCO через трамплин"""
    result = fn(*args)
    while callable(result):
        result = result()
    return result

def factorial_trampoline(n, acc=1):
    if n <= 1:
        return acc
    return lambda: factorial_trampoline(n - 1, n * acc)

# Использование
result = trampoline(factorial_trampoline, 10000)
print(result)  # Работает без Stack Overflow!
```

### 4. Неправильная инициализация аккумулятора

```python
# ОШИБКА: неправильное начальное значение
def sum_tail_bad(arr, acc=1):  # acc должен быть 0!
    if not arr:
        return acc
    return sum_tail_bad(arr[1:], acc + arr[0])

sum_tail_bad([1, 2, 3])  # Вернёт 7 вместо 6!

# ПРАВИЛЬНО
def sum_tail_good(arr, acc=0):
    if not arr:
        return acc
    return sum_tail_good(arr[1:], acc + arr[0])
```

### 5. Использование изменяемого значения по умолчанию

```python
# ОШИБКА: изменяемый аргумент по умолчанию
def collect_tail_bad(n, result=[]):  # Опасно!
    if n == 0:
        return result
    result.append(n)
    return collect_tail_bad(n - 1, result)

collect_tail_bad(3)  # [3, 2, 1]
collect_tail_bad(3)  # [3, 2, 1, 3, 2, 1] — список сохранился!

# ПРАВИЛЬНО
def collect_tail_good(n, result=None):
    if result is None:
        result = []
    if n == 0:
        return result
    result.append(n)
    return collect_tail_good(n - 1, result)
```

### Рекомендации:
1. В Python лучше использовать итеративный подход для глубокой рекурсии
2. Хвостовая рекурсия полезна для понимания концепции и работы в языках с TCO
3. Используйте трамплин или декораторы для эмуляции TCO в Python
4. Всегда проверяйте начальное значение аккумулятора
5. Избегайте изменяемых значений по умолчанию

---
[prev: 01-recursion-basics](./01-recursion-basics.md) | [next: 03-recursion-vs-iteration](./03-recursion-vs-iteration.md)