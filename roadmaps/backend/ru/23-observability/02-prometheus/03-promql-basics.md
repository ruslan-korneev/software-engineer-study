# Основы PromQL

[prev: 02-setup-and-configuration](./02-setup-and-configuration.md) | [next: 04-monitoring-targets](./04-monitoring-targets.md)

---

## Введение

PromQL (Prometheus Query Language) — это функциональный язык запросов, разработанный специально для работы с временными рядами в Prometheus. PromQL позволяет выполнять агрегации, фильтрации и математические операции над метриками в реальном времени.

## Типы данных

### 1. Instant Vector (Мгновенный вектор)

Набор временных рядов с единственным значением для каждого ряда на определённый момент времени.

```promql
# Возвращает текущее значение всех временных рядов метрики
http_requests_total

# С фильтрацией по лейблам
http_requests_total{job="api-server"}

# Результат:
# http_requests_total{job="api-server", method="GET", status="200"} 1234
# http_requests_total{job="api-server", method="POST", status="200"} 567
```

### 2. Range Vector (Диапазонный вектор)

Набор временных рядов со значениями за указанный период времени.

```promql
# Значения за последние 5 минут
http_requests_total[5m]

# Результат:
# http_requests_total{...} 100 @timestamp1
# http_requests_total{...} 105 @timestamp2
# http_requests_total{...} 112 @timestamp3
# ...
```

### 3. Scalar (Скаляр)

Простое числовое значение с плавающей точкой.

```promql
# Пример скалярного значения
42
3.14
-1.5e-3
```

### 4. String (Строка)

Строковое значение (используется редко).

```promql
"hello world"
```

## Селекторы и матчеры

### Операторы сопоставления лейблов

```promql
# Точное равенство
http_requests_total{method="GET"}

# Неравенство
http_requests_total{method!="GET"}

# Регулярное выражение (соответствует)
http_requests_total{method=~"GET|POST"}

# Регулярное выражение (не соответствует)
http_requests_total{method!~"OPTIONS|HEAD"}

# Комбинация условий (логическое И)
http_requests_total{job="api", method="GET", status=~"2.."}

# Селектор по имени метрики через регулярное выражение
{__name__=~"http_requests_.*"}

# Все метрики для job="prometheus"
{job="prometheus"}
```

### Модификаторы времени

```promql
# Значения 5 минут назад (offset)
http_requests_total offset 5m

# Range vector с offset
rate(http_requests_total[5m] offset 1h)

# Значение на конкретный момент времени (@ modifier)
http_requests_total @ 1609459200

# Начало/конец указанного диапазона
http_requests_total @ start()
http_requests_total @ end()
```

### Диапазоны времени

```promql
# Поддерживаемые единицы времени
[5s]   # секунды
[5m]   # минуты
[5h]   # часы
[5d]   # дни
[5w]   # недели
[5y]   # годы

# Комбинации
[1h30m]
[1d12h]
```

## Арифметические операторы

### Бинарные операторы

```promql
# Сложение
node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes

# Вычитание
http_requests_total - http_requests_total offset 1h

# Умножение
node_cpu_seconds_total * 1000

# Деление
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes

# Модуль (остаток от деления)
http_requests_total % 100

# Возведение в степень
2 ^ 10
```

### Операторы сравнения

```promql
# Возвращают 1 (true) или 0 (false) / фильтруют значения
http_requests_total > 1000
http_requests_total >= 1000
http_requests_total < 100
http_requests_total <= 100
http_requests_total == 0
http_requests_total != 0

# С модификатором bool возвращают 0 или 1
http_requests_total > bool 1000
```

### Логические операторы

```promql
# AND - возвращает элементы из левого вектора, для которых есть соответствие справа
http_requests_total and on(job) up

# OR - объединение
http_requests_total{status="200"} or http_requests_total{status="201"}

# UNLESS - левый вектор минус элементы, соответствующие правому
http_requests_total unless http_requests_total{status=~"4.."}
```

## Vector Matching (Сопоставление векторов)

### One-to-one

```promql
# Сопоставление по всем лейблам
method_code:http_errors:rate5m / method_code:http_requests:rate5m

# Игнорирование определённых лейблов
method_code:http_errors:rate5m / ignoring(code) method:http_requests:rate5m

# Сопоставление только по указанным лейблам
method_code:http_errors:rate5m / on(method) method:http_requests:rate5m
```

### One-to-many / Many-to-one

```promql
# group_left - много элементов слева соответствуют одному справа
method_code:http_errors:rate5m / on(method) group_left method:http_requests:rate5m

# group_right - один элемент слева соответствует многим справа
method:http_requests:rate5m / on(method) group_right method_code:http_errors:rate5m

# С копированием лейблов
method_code:http_errors:rate5m
  / on(method) group_left(team)
  method:http_requests:rate5m
```

## Агрегирующие операторы

### Основные операторы

```promql
# Сумма по всем сериям
sum(http_requests_total)

# Сумма по группам (by)
sum by (job, method) (http_requests_total)

# Сумма без указанных лейблов (without)
sum without (instance) (http_requests_total)

# Среднее значение
avg(node_cpu_seconds_total)

# Минимум и максимум
min(node_memory_MemAvailable_bytes)
max(node_memory_MemAvailable_bytes)

# Количество элементов
count(up)

# Количество уникальных значений
count_values("version", prometheus_build_info)

# Стандартное отклонение
stddev(http_request_duration_seconds)

# Дисперсия
stdvar(http_request_duration_seconds)

# Топ-N
topk(3, http_requests_total)

# Последние N
bottomk(3, http_requests_total)

# Квантиль (для gauge метрик)
quantile(0.95, http_request_duration_seconds)

# Группировка без агрегации
group by (job) (up)
```

### Примеры агрегации

```promql
# Общее количество запросов по статусу
sum by (status) (rate(http_requests_total[5m]))

# Средняя загрузка CPU по instance
avg by (instance) (rate(node_cpu_seconds_total{mode!="idle"}[5m]))

# Топ-5 сервисов по количеству ошибок
topk(5, sum by (service) (rate(http_requests_total{status=~"5.."}[5m])))

# Количество активных инстансов по job
count by (job) (up == 1)
```

## Функции

### Функции для Counter (rate, irate, increase)

```promql
# rate - средняя скорость изменения за период
rate(http_requests_total[5m])

# irate - мгновенная скорость (последние 2 точки)
irate(http_requests_total[5m])

# increase - абсолютное изменение за период
increase(http_requests_total[1h])

# Примеры использования
# Запросов в секунду за последние 5 минут
rate(http_requests_total[5m])

# Запросов за последний час
increase(http_requests_total[1h])
```

### Функции для Gauge

```promql
# delta - изменение значения gauge за период
delta(node_memory_MemAvailable_bytes[1h])

# idelta - изменение между последними двумя точками
idelta(node_memory_MemAvailable_bytes[5m])

# deriv - производная (скорость изменения) через линейную регрессию
deriv(node_memory_MemAvailable_bytes[1h])

# predict_linear - прогноз значения через N секунд
predict_linear(node_filesystem_avail_bytes[1h], 4 * 3600)
```

### Функции для Histogram

```promql
# histogram_quantile - вычисление квантиля из гистограммы
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# С группировкой
histogram_quantile(0.99,
  sum by (le, job) (rate(http_request_duration_seconds_bucket[5m]))
)

# Медиана (50-й перцентиль)
histogram_quantile(0.5, rate(http_request_duration_seconds_bucket[5m]))

# histogram_count и histogram_sum (для native histograms)
histogram_count(http_request_duration_seconds)
histogram_sum(http_request_duration_seconds)
```

### Математические функции

```promql
# Абсолютное значение
abs(delta(temperature_celsius[1h]))

# Округление
ceil(cpu_usage_percent)    # Вверх
floor(cpu_usage_percent)   # Вниз
round(cpu_usage_percent)   # К ближайшему
round(cpu_usage_percent, 0.5)  # С шагом 0.5

# Логарифмы и экспоненты
ln(metric)
log2(metric)
log10(metric)
exp(metric)

# Квадратный корень
sqrt(metric)

# Тригонометрические функции
sin(metric)
cos(metric)
tan(metric)
asin(metric)
acos(metric)
atan(metric)

# Зажим значений в диапазон
clamp(cpu_usage, 0, 100)
clamp_min(cpu_usage, 0)
clamp_max(cpu_usage, 100)
```

### Функции для работы со временем

```promql
# Текущее время в Unix timestamp
time()

# Возраст последнего скрейпинга (время с момента получения)
timestamp(up)

# Минуты от начала часа
minute()

# Часы
hour()

# День месяца (1-31)
day_of_month()

# День недели (0=воскресенье, 6=суббота)
day_of_week()

# День года (1-366)
day_of_year()

# Месяц (1-12)
month()

# Год
year()

# Количество дней в месяце
days_in_month()
```

### Функции для работы с лейблами

```promql
# Добавление/замена лейбла
label_replace(up, "host", "$1", "instance", "(.*):.*")

# Объединение лейблов
label_join(up, "full_address", "-", "job", "instance")

# Примеры
# Извлечь hostname из instance
label_replace(
  node_cpu_seconds_total,
  "hostname",
  "$1",
  "instance",
  "(.*):\\d+"
)

# Объединить job и instance в один лейбл
label_join(
  up,
  "target",
  ":",
  "job", "instance"
)
```

### Функции для отсутствующих данных

```promql
# Вернуть пустой вектор если данные есть, 1 если данных нет
absent(up{job="nonexistent"})

# То же для range vector
absent_over_time(up{job="api"}[5m])

# Использование в алертах
absent(up{job="critical-service"}) == 1
```

### Агрегация по времени

```promql
# Среднее за период
avg_over_time(node_cpu_seconds_total[5m])

# Минимум/максимум за период
min_over_time(temperature[1h])
max_over_time(temperature[1h])

# Сумма за период
sum_over_time(http_requests_total[1h])

# Количество точек за период
count_over_time(up[1h])

# Квантиль за период
quantile_over_time(0.95, http_request_duration_seconds[1h])

# Стандартное отклонение за период
stddev_over_time(cpu_usage[1h])

# Последнее значение за период
last_over_time(cpu_usage[5m])

# Наличие данных за период (1 если есть хотя бы одна точка)
present_over_time(up[5m])
```

### Прочие функции

```promql
# Сортировка
sort(http_requests_total)
sort_desc(http_requests_total)

# Вектор из скаляра
vector(1)

# Скаляр из вектора (только если вектор имеет 1 элемент)
scalar(sum(up))

# Информация о сериях
changes(process_start_time_seconds[1h])  # Количество изменений
resets(http_requests_total[1h])          # Количество сбросов counter

# Сглаживание
holt_winters(node_cpu_seconds_total[1h], 0.5, 0.5)
```

## Практические примеры запросов

### Мониторинг HTTP-сервиса

```promql
# Общий RPS (requests per second)
sum(rate(http_requests_total[5m]))

# RPS по эндпоинтам
sum by (endpoint) (rate(http_requests_total[5m]))

# Процент ошибок (Error Rate)
sum(rate(http_requests_total{status=~"5.."}[5m]))
  / sum(rate(http_requests_total[5m])) * 100

# 95-й перцентиль латентности
histogram_quantile(0.95,
  sum by (le) (rate(http_request_duration_seconds_bucket[5m]))
)

# Apdex Score (порог 0.5s, толерантный 2s)
(
  sum(rate(http_request_duration_seconds_bucket{le="0.5"}[5m]))
  + sum(rate(http_request_duration_seconds_bucket{le="2"}[5m]))
) / 2 / sum(rate(http_request_duration_seconds_count[5m]))
```

### Мониторинг системных ресурсов

```promql
# CPU Usage (%)
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory Usage (%)
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# Disk Usage (%)
(1 - node_filesystem_avail_bytes{mountpoint="/"}
  / node_filesystem_size_bytes{mountpoint="/"}) * 100

# Network I/O (bytes/sec)
rate(node_network_receive_bytes_total[5m])
rate(node_network_transmit_bytes_total[5m])

# Disk I/O (operations/sec)
rate(node_disk_reads_completed_total[5m])
rate(node_disk_writes_completed_total[5m])

# Прогноз заполнения диска через 24 часа
predict_linear(node_filesystem_avail_bytes{mountpoint="/"}[1h], 24 * 3600)
```

### Мониторинг Kubernetes

```promql
# Количество подов по namespace
count by (namespace) (kube_pod_info)

# Поды в статусе не Running
kube_pod_status_phase{phase!="Running"} == 1

# CPU запрошенный vs лимит
sum by (namespace) (kube_pod_container_resource_requests{resource="cpu"})
sum by (namespace) (kube_pod_container_resource_limits{resource="cpu"})

# Контейнеры с OOMKilled
increase(kube_pod_container_status_restarts_total[1h]) > 0
  and on(namespace, pod, container)
  kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}

# Ноды NotReady
kube_node_status_condition{condition="Ready",status="true"} == 0
```

### Мониторинг базы данных (PostgreSQL)

```promql
# Активные соединения
pg_stat_activity_count{state="active"}

# Транзакции в секунду
rate(pg_stat_database_xact_commit[5m])
  + rate(pg_stat_database_xact_rollback[5m])

# Cache hit ratio
pg_stat_database_blks_hit
  / (pg_stat_database_blks_hit + pg_stat_database_blks_read) * 100

# Размер базы данных
pg_database_size_bytes

# Долгие запросы
pg_stat_activity_max_tx_duration{state!="idle"}
```

## Recording Rules с PromQL

```yaml
groups:
  - name: http_rules
    rules:
      - record: job:http_requests:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))

      - record: job:http_errors:ratio5m
        expr: |
          sum by (job) (rate(http_requests_total{status=~"5.."}[5m]))
          / sum by (job) (rate(http_requests_total[5m]))

      - record: job:http_latency:p99
        expr: |
          histogram_quantile(0.99,
            sum by (job, le) (rate(http_request_duration_seconds_bucket[5m]))
          )
```

## Best Practices

### 1. Используйте rate() вместо irate() для алертов

```promql
# Хорошо - стабильный результат для алертов
rate(http_requests_total[5m])

# Осторожно - слишком волатильный для алертов
irate(http_requests_total[5m])
```

### 2. Правильный диапазон для rate()

```promql
# Диапазон должен быть минимум 4x scrape_interval
# Если scrape_interval=15s, минимальный диапазон = 1m
rate(http_requests_total[1m])  # При scrape_interval=15s
```

### 3. Избегайте высокой кардинальности в запросах

```promql
# Плохо - создаёт много временных рядов
sum by (user_id, request_id) (http_requests_total)

# Хорошо - агрегация по лейблам с низкой кардинальностью
sum by (method, status) (http_requests_total)
```

### 4. Используйте Recording Rules для сложных запросов

```promql
# Вместо вычисления каждый раз
histogram_quantile(0.99, sum by (le, job) (rate(http_request_duration_seconds_bucket[5m])))

# Предварительно вычисленный recording rule
job:http_latency:p99
```

## Частые ошибки

1. **rate() от Gauge** — rate() предназначен только для Counter
2. **Неправильный диапазон для rate()** — слишком короткий диапазон даёт неточные результаты
3. **Деление на ноль** — всегда проверяйте, что знаменатель не равен нулю
4. **Игнорирование сбросов Counter** — rate() корректно обрабатывает сбросы, но increase() может давать отрицательные значения при сбросах
5. **Сравнение несовместимых векторов** — лейблы должны совпадать или использовать on()/ignoring()

## Заключение

PromQL — мощный инструмент для анализа метрик. Освоение селекторов, функций rate/histogram_quantile и агрегаций позволит вам создавать эффективные дашборды и алерты для мониторинга любых систем.
