# explain.dalibo.com - Визуализатор планов Dalibo

[prev: 03-pev2](./03-pev2.md) | [next: 01-use-method](../20-monitoring-techniques/01-use-method.md)

---

## Введение

**explain.dalibo.com** - это бесплатный онлайн-сервис от компании Dalibo (французская компания, специализирующаяся на PostgreSQL) для визуализации и анализа планов выполнения запросов. Инструмент основан на PEV2 и предоставляет удобный веб-интерфейс.

## Ключевые особенности

- **Бесплатный и без регистрации**
- **Графическая визуализация** плана запроса
- **Поддержка всех форматов** EXPLAIN (TEXT, JSON, XML, YAML)
- **Возможность шаринга** планов по ссылке
- **Open Source** - можно развернуть локально
- **Приватный режим** - план не сохраняется на сервере

## Интерфейс

### Главная страница

При открытии https://explain.dalibo.com/analyze/new вы увидите:

1. **Текстовое поле** для вставки плана
2. **Опция "Keep plan private"** - план не будет сохранен
3. **Кнопка "Submit"** для анализа

### Поддерживаемые форматы ввода

```sql
-- Рекомендуемый формат (JSON)
EXPLAIN (ANALYZE, COSTS, VERBOSE, BUFFERS, FORMAT JSON) SELECT ...

-- Текстовый формат (также поддерживается)
EXPLAIN (ANALYZE, COSTS, VERBOSE, BUFFERS) SELECT ...

-- XML формат
EXPLAIN (ANALYZE, FORMAT XML) SELECT ...

-- YAML формат
EXPLAIN (ANALYZE, FORMAT YAML) SELECT ...
```

## Пошаговое использование

### Шаг 1: Выполните EXPLAIN в PostgreSQL

```sql
EXPLAIN (ANALYZE, COSTS, VERBOSE, BUFFERS, FORMAT JSON)
SELECT
    u.email,
    u.name,
    COUNT(DISTINCT o.id) as order_count,
    SUM(oi.quantity * oi.unit_price) as total_spent
FROM users u
INNER JOIN orders o ON u.id = o.user_id
INNER JOIN order_items oi ON o.id = oi.order_id
WHERE u.created_at >= '2023-01-01'
  AND o.status IN ('completed', 'shipped')
GROUP BY u.id, u.email, u.name
HAVING SUM(oi.quantity * oi.unit_price) > 500
ORDER BY total_spent DESC
LIMIT 50;
```

### Шаг 2: Скопируйте результат

Скопируйте весь JSON-вывод, включая квадратные скобки:

```json
[
  {
    "Plan": {
      "Node Type": "Limit",
      "Actual Rows": 50,
      "Actual Total Time": 234.567,
      ...
    },
    "Planning Time": 1.234,
    "Execution Time": 235.678,
    ...
  }
]
```

### Шаг 3: Вставьте и проанализируйте

1. Вставьте план в текстовое поле на https://explain.dalibo.com/analyze/new
2. Опционально отметьте "Keep plan private"
3. Нажмите "Submit"

## Компоненты результата

### 1. Plan Tab (Вкладка плана)

Графическое представление плана в виде интерактивного дерева:

```
┌─────────────┐
│   Limit     │ ──► [50 rows, 0.12ms]
└──────┬──────┘
       │
       ▼
┌─────────────┐
│    Sort     │ ──► [50 rows, 0.89ms, top-N heapsort]
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ HashAggregate│ ──► [2500 rows, 45.23ms]
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  Hash Join  │ ──► [125000 rows, 89.45ms]
└──────┬──────┘
      ╱ ╲
     ╱   ╲
    ▼     ▼
  ...    ...
```

### 2. Stats Tab (Вкладка статистики)

Общая информация:

| Метрика | Значение |
|---------|----------|
| Planning Time | 1.234 ms |
| Execution Time | 235.678 ms |
| Slowest Node | Hash Join on orders |
| Largest Node | Seq Scan on order_items |
| Most Rows | 125000 (Hash Join) |

### 3. Raw Tab (Исходный план)

Исходный текст плана EXPLAIN для справки.

## Детали узла

При клике на узел появляется панель с детальной информацией:

### Основные метрики

```
Node Type:           Hash Join
Parent Relationship: Outer
Join Type:           Inner

Timing:
  Startup:  12.345 ms
  Total:    89.456 ms

Rows:
  Estimated: 100000
  Actual:    125000
  Ratio:     1.25x

I/O:
  Shared Hit:    5432
  Shared Read:   234
  Shared Written: 0
```

### Цветовая индикация времени

Dalibo использует прогрессивную цветовую шкалу:

| Процент времени | Цвет | Индикация |
|-----------------|------|-----------|
| 0-10% | Серый/Зеленый | Нормально |
| 10-30% | Желтый | Обратите внимание |
| 30-50% | Оранжевый | Возможная проблема |
| 50%+ | Красный | Узкое место |

### Workers (параллельные процессы)

Если план использует параллельное выполнение:

```
Workers Planned: 4
Workers Launched: 4

Per-Worker Stats:
  Worker 0: 1234 rows, 23.45ms
  Worker 1: 1256 rows, 24.12ms
  Worker 2: 1198 rows, 22.89ms
  Worker 3: 1312 rows, 25.01ms
```

## Шаринг планов

### Публичные планы

После отправки плана (без опции "Keep plan private") вы получаете URL:

```
https://explain.dalibo.com/analyze/plan/abc123xyz
```

Этот URL можно:
- Отправить коллегам
- Добавить в тикет или документацию
- Использовать для обсуждения на форумах

### Приватные планы

С опцией "Keep plan private":
- План обрабатывается только в браузере
- Не сохраняется на сервере
- URL содержит полный план в закодированном виде (длинный URL)

## Практические примеры анализа

### Пример 1: Неэффективный Seq Scan

**План показывает:**
```
Seq Scan on orders
  Filter: (status = 'completed')
  Rows Removed by Filter: 950000
  Actual Rows: 50000
  Time: 1234.56 ms  [КРАСНЫЙ]
```

**Рекомендация:** Создать индекс
```sql
CREATE INDEX idx_orders_status ON orders(status);
```

### Пример 2: Неточная оценка строк

**План показывает:**
```
Hash Join
  Estimated Rows: 1000
  Actual Rows: 150000
  Rows Ratio: 150x
```

**Рекомендация:** Обновить статистику
```sql
ANALYZE orders;
ANALYZE order_items;
```

### Пример 3: Сортировка на диске

**План показывает:**
```
Sort
  Sort Method: external merge
  Sort Space: 234 MB (Disk)
  Time: 5678.90 ms  [КРАСНЫЙ]
```

**Рекомендация:** Увеличить work_mem
```sql
SET work_mem = '512MB';
-- или в postgresql.conf
```

## Сравнение с depesz

| Аспект | Dalibo | depesz |
|--------|--------|--------|
| Визуализация | Графическое дерево | Таблица |
| Интерактивность | Высокая (клик на узлы) | Средняя |
| Приватный режим | Да | Нет (по умолчанию) |
| Exclusive time | Да | Да |
| JSON поддержка | Отличная | Хорошая |
| Мобильная версия | Адаптивная | Ограниченная |

## Локальная установка

Для конфиденциальных данных можно развернуть локально:

```bash
# Клонирование репозитория
git clone https://github.com/dalibo/pev2.git
cd pev2

# Установка зависимостей
npm install

# Запуск в режиме разработки
npm run serve

# Или сборка для продакшена
npm run build
```

После сборки статические файлы можно разместить на любом веб-сервере.

## Docker

```bash
docker run -d -p 8080:80 dalibo/pev2
```

Откройте http://localhost:8080

## Интеграция с другими инструментами

### pgAdmin

pgAdmin 4 использует похожую визуализацию для планов запросов.

### psql

Можно напрямую копировать вывод EXPLAIN из psql:

```bash
psql -c "EXPLAIN (ANALYZE, FORMAT JSON) SELECT ..." | pbcopy
```

### Скрипты автоматизации

```python
import psycopg2
import json
import webbrowser
import urllib.parse

conn = psycopg2.connect("...")
cur = conn.cursor()
cur.execute("EXPLAIN (ANALYZE, FORMAT JSON) SELECT ...")
plan = cur.fetchone()[0]

# Открыть в браузере
encoded = urllib.parse.quote(json.dumps(plan))
webbrowser.open(f"https://explain.dalibo.com/analyze/new?plan={encoded}")
```

## Советы по использованию

1. **Используйте JSON** для наиболее полного анализа
2. **Включайте BUFFERS** для понимания I/O
3. **Проверяйте "rows ratio"** - отношение actual/estimated
4. **Обращайте внимание на красные узлы** - это узкие места
5. **Сравнивайте планы** до и после оптимизации
6. **Используйте приватный режим** для конфиденциальных запросов

## Ссылки

- [explain.dalibo.com](https://explain.dalibo.com/)
- [Dalibo - PostgreSQL experts](https://dalibo.com/)
- [PEV2 на GitHub](https://github.com/dalibo/pev2)
- [Документация PostgreSQL](https://www.postgresql.org/docs/current/sql-explain.html)

---

[prev: 03-pev2](./03-pev2.md) | [next: 01-use-method](../20-monitoring-techniques/01-use-method.md)
