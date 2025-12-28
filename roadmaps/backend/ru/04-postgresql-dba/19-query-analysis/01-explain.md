# EXPLAIN и EXPLAIN ANALYZE

[prev: 05-postgres-tools](../18-troubleshooting/05-postgres-tools.md) | [next: 02-depesz](./02-depesz.md)

---

## Введение

EXPLAIN - это команда PostgreSQL, которая показывает план выполнения запроса без его фактического выполнения. Это фундаментальный инструмент для анализа и оптимизации производительности SQL-запросов.

## Базовый синтаксис

```sql
EXPLAIN [опции] запрос;
EXPLAIN ANALYZE [опции] запрос;
```

### Основные опции

| Опция | Описание |
|-------|----------|
| `ANALYZE` | Выполняет запрос и показывает реальное время выполнения |
| `VERBOSE` | Показывает дополнительную информацию (выводимые столбцы) |
| `COSTS` | Показывает оценочную стоимость (по умолчанию включено) |
| `BUFFERS` | Показывает информацию об использовании буферов |
| `TIMING` | Показывает время выполнения каждого узла |
| `FORMAT` | Формат вывода: TEXT, XML, JSON, YAML |

## Примеры использования

### Простой EXPLAIN

```sql
EXPLAIN SELECT * FROM users WHERE id = 100;
```

Результат:
```
Index Scan using users_pkey on users  (cost=0.29..8.31 rows=1 width=68)
  Index Cond: (id = 100)
```

### EXPLAIN ANALYZE

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE created_at > '2024-01-01';
```

Результат:
```
Seq Scan on orders  (cost=0.00..1250.00 rows=5000 width=124) (actual time=0.015..12.456 rows=4823 loops=1)
  Filter: (created_at > '2024-01-01'::date)
  Rows Removed by Filter: 15177
Planning Time: 0.089 ms
Execution Time: 12.892 ms
```

### С опцией BUFFERS

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM products WHERE category_id = 5;
```

Результат:
```
Index Scan using products_category_idx on products  (cost=0.29..52.31 rows=50 width=96) (actual time=0.025..0.156 rows=47 loops=1)
  Index Cond: (category_id = 5)
  Buffers: shared hit=12
Planning Time: 0.112 ms
Execution Time: 0.189 ms
```

## Понимание метрик

### Стоимость (Cost)

```
(cost=0.29..8.31 rows=1 width=68)
       │     │     │       │
       │     │     │       └── Средний размер строки в байтах
       │     │     └── Оценочное количество возвращаемых строк
       │     └── Общая стоимость получения всех строк
       └── Стоимость запуска (до получения первой строки)
```

Стоимость измеряется в условных единицах, основанных на параметрах конфигурации:
- `seq_page_cost = 1.0` - стоимость чтения одной страницы последовательно
- `random_page_cost = 4.0` - стоимость случайного чтения страницы
- `cpu_tuple_cost = 0.01` - стоимость обработки одной строки
- `cpu_index_tuple_cost = 0.005` - стоимость обработки индексной записи
- `cpu_operator_cost = 0.0025` - стоимость выполнения оператора

### Actual Time

```
(actual time=0.015..12.456 rows=4823 loops=1)
              │        │      │        │
              │        │      │        └── Количество итераций (для вложенных циклов)
              │        │      └── Фактическое количество строк
              │        └── Время получения всех строк (мс)
              └── Время до первой строки (мс)
```

## Типы узлов (Node Types)

### Методы сканирования таблиц

| Тип | Описание | Когда используется |
|-----|----------|-------------------|
| **Seq Scan** | Последовательное сканирование | Большая выборка или нет подходящего индекса |
| **Index Scan** | Сканирование по индексу | Малая выборка с подходящим индексом |
| **Index Only Scan** | Только индекс | Все нужные данные есть в индексе |
| **Bitmap Index Scan** | Битовая карта индекса | Средняя выборка или OR-условия |
| **Bitmap Heap Scan** | Чтение по битовой карте | После Bitmap Index Scan |

### Методы соединения (JOIN)

| Тип | Описание | Когда оптимален |
|-----|----------|-----------------|
| **Nested Loop** | Вложенный цикл | Малые таблицы или индексированный поиск |
| **Hash Join** | Хеш-соединение | Большие таблицы, равенство |
| **Merge Join** | Сортировка-слияние | Отсортированные данные |

### Агрегация и сортировка

| Тип | Описание |
|-----|----------|
| **Sort** | Сортировка (может использовать диск) |
| **Hash Aggregate** | Агрегация через хеш-таблицу |
| **Group Aggregate** | Агрегация отсортированных данных |
| **Limit** | Ограничение количества строк |

## Чтение сложных планов

### Пример многотабличного запроса

```sql
EXPLAIN ANALYZE
SELECT u.name, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.created_at > '2023-01-01'
GROUP BY u.id, u.name
ORDER BY order_count DESC
LIMIT 10;
```

```
Limit  (cost=1234.56..1234.58 rows=10 width=40) (actual time=45.123..45.130 rows=10 loops=1)
  ->  Sort  (cost=1234.56..1237.06 rows=1000 width=40) (actual time=45.121..45.125 rows=10 loops=1)
        Sort Key: (count(o.id)) DESC
        Sort Method: top-N heapsort  Memory: 25kB
        ->  HashAggregate  (cost=1200.00..1212.50 rows=1000 width=40) (actual time=43.567..44.012 rows=856 loops=1)
              Group Key: u.id
              Batches: 1  Memory Usage: 128kB
              ->  Hash Right Join  (cost=125.00..1150.00 rows=10000 width=36) (actual time=1.234..35.678 rows=8965 loops=1)
                    Hash Cond: (o.user_id = u.id)
                    ->  Seq Scan on orders o  (cost=0.00..800.00 rows=20000 width=8) (actual time=0.012..15.234 rows=20000 loops=1)
                    ->  Hash  (cost=100.00..100.00 rows=2000 width=36) (actual time=1.189..1.190 rows=1856 loops=1)
                          Buckets: 2048  Batches: 1  Memory Usage: 120kB
                          ->  Seq Scan on users u  (cost=0.00..100.00 rows=2000 width=36) (actual time=0.008..0.856 rows=1856 loops=1)
                                Filter: (created_at > '2023-01-01'::date)
                                Rows Removed by Filter: 144
Planning Time: 0.456 ms
Execution Time: 45.234 ms
```

### Как читать план

1. **Читайте снизу вверх** - выполнение начинается с самых вложенных узлов
2. **Отступы показывают вложенность** - дочерние узлы имеют больший отступ
3. **Стрелки `->` указывают на дочерние операции**
4. **Сравнивайте estimated и actual** - большие расхождения указывают на устаревшую статистику

## Проблемы и их диагностика

### Устаревшая статистика

Если `rows` сильно отличается от `actual rows`:

```sql
ANALYZE table_name;
```

### Неиспользуемые индексы

Если видите Seq Scan вместо Index Scan:
- Проверьте наличие индекса
- Проверьте тип данных в условии
- Убедитесь, что выборка достаточно селективна

### Сортировка на диске

```
Sort Method: external merge  Disk: 15000kB
```

Решение: увеличить `work_mem` или оптимизировать запрос.

## Формат JSON для программного анализа

```sql
EXPLAIN (ANALYZE, FORMAT JSON) SELECT * FROM users WHERE id = 1;
```

```json
[
  {
    "Plan": {
      "Node Type": "Index Scan",
      "Relation Name": "users",
      "Index Name": "users_pkey",
      "Startup Cost": 0.29,
      "Total Cost": 8.31,
      "Plan Rows": 1,
      "Plan Width": 68,
      "Actual Startup Time": 0.025,
      "Actual Total Time": 0.027,
      "Actual Rows": 1,
      "Actual Loops": 1
    },
    "Planning Time": 0.089,
    "Execution Time": 0.052
  }
]
```

## Советы по оптимизации

1. **Всегда используйте ANALYZE** для реальных измерений
2. **Добавляйте BUFFERS** для понимания I/O
3. **Сравнивайте планы** до и после оптимизации
4. **Обращайте внимание на**:
   - Seq Scan на больших таблицах
   - Nested Loop с большим количеством loops
   - Сортировку на диске
   - Большое расхождение между estimated и actual rows

## Ссылки

- [Официальная документация EXPLAIN](https://www.postgresql.org/docs/current/sql-explain.html)
- [Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)

---

[prev: 05-postgres-tools](../18-troubleshooting/05-postgres-tools.md) | [next: 02-depesz](./02-depesz.md)
