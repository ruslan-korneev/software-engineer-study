# Best Practices для SLI, SLO и SLA

## Введение

Внедрение SLI, SLO и SLA - это не просто техническая задача, а культурный сдвиг в организации. В этом разделе мы рассмотрим лучшие практики, которые помогут успешно внедрить и использовать эти концепции.

## Выбор правильных SLI

### 1. Измеряйте с точки зрения пользователя

```yaml
# Плохо: внутренние метрики
sli:
  - name: "cpu_usage"
    query: "avg(cpu_usage_percent)"
  - name: "memory_usage"
    query: "avg(memory_usage_percent)"

# Хорошо: метрики пользовательского опыта
sli:
  - name: "request_success_rate"
    query: "successful_requests / total_requests"
  - name: "user_perceived_latency"
    query: "p95(end_to_end_response_time)"
```

### 2. Используйте VALET framework

VALET помогает определить правильные SLI для разных типов систем:

| Буква | Значение | Пример |
|-------|----------|--------|
| **V** | Volume (объем) | Requests per second, Messages processed |
| **A** | Availability (доступность) | Uptime, Success rate |
| **L** | Latency (латентность) | Response time percentiles |
| **E** | Errors (ошибки) | Error rate, Exception count |
| **T** | Tickets (тикеты) | Support tickets, Manual interventions |

### 3. Правило 3-5 SLI на сервис

```yaml
# Оптимальный набор SLI для API-сервиса
slis:
  # Availability - всегда нужен
  - name: "availability"
    type: "request_success_rate"

  # Latency - для пользовательского опыта
  - name: "latency_p50"
    type: "median_response_time"
  - name: "latency_p99"
    type: "tail_latency"

  # Опционально: специфичные для бизнеса
  - name: "payment_success_rate"
    type: "business_transaction_success"
```

### 4. Избегайте vanity metrics

```yaml
# Плохо: метрики которые всегда выглядят хорошо
vanity_metrics:
  - "total_users_registered"      # Только растет
  - "total_requests_served"        # Не показывает качество
  - "average_response_time"        # Скрывает проблемы в хвосте

# Хорошо: actionable метрики
actionable_metrics:
  - "daily_active_users_success_rate"
  - "request_error_rate"
  - "p99_latency"
```

## Установка реалистичных SLO

### 1. Начните с измерений

```python
# Шаг 1: Соберите данные за 30 дней
historical_data = collect_metrics(last_30_days)

# Шаг 2: Проанализируйте текущую производительность
current_availability = historical_data.availability.mean()  # 99.87%
current_p99_latency = historical_data.latency.percentile(99)  # 450ms

# Шаг 3: Установите SLO на основе данных
# Немного ниже текущего уровня, чтобы был error budget
proposed_slo = {
    "availability": 99.9,  # Текущее 99.87%, округляем
    "latency_p99": 500,    # Текущее 450ms, даем запас
}
```

### 2. Не ставьте SLO выше возможностей зависимостей

```
Ваш сервис зависит от:
├── Database (SLA: 99.95%)
├── Redis (SLA: 99.9%)
├── External API (SLA: 99.5%)
└── Cloud Provider (SLA: 99.99%)

Максимальная теоретическая доступность:
99.95% × 99.9% × 99.5% × 99.99% = 99.34%

Рекомендуемый SLO: ≤ 99.3% (с учетом собственных сбоев)
```

### 3. Используйте правило "нескольких девяток"

```yaml
# Практические рекомендации по "девяткам"
guidelines:
  - target: "99%"
    suitable_for: "Внутренние инструменты, некритичные сервисы"
    downtime_per_month: "7.3 часа"

  - target: "99.9%"
    suitable_for: "Большинство production сервисов"
    downtime_per_month: "43 минуты"

  - target: "99.95%"
    suitable_for: "Критичные бизнес-сервисы"
    downtime_per_month: "21 минута"

  - target: "99.99%"
    suitable_for: "Инфраструктурные сервисы, платежи"
    downtime_per_month: "4 минуты"
    note: "Требует значительных инвестиций"
```

### 4. Не стремитесь к 100%

```
Почему 100% SLO - плохая идея:

1. Технически невозможно:
   - Сеть ненадежна
   - Железо ломается
   - ПО содержит баги

2. Экономически нецелесообразно:
   - Стоимость каждой "девятки" растет экспоненциально
   - 99.99% может стоить в 10 раз дороже 99.9%

3. Блокирует развитие:
   - Error budget = 0
   - Нельзя релизить
   - Нельзя экспериментировать

4. Психологически вредно:
   - Любой инцидент = катастрофа
   - Команда боится изменений
```

## Работа с Error Budget

### 1. Политика Error Budget

```yaml
error_budget_policy:
  version: "1.0"
  service: "payment-api"

  thresholds:
    healthy:
      budget_remaining: ">50%"
      actions:
        - "Обычный режим разработки"
        - "Можно проводить эксперименты"
        - "Стандартный review процесс"

    caution:
      budget_remaining: "25-50%"
      actions:
        - "Усиленное тестирование"
        - "Еженедельный обзор инцидентов"
        - "Приоритизация reliability работы"

    warning:
      budget_remaining: "10-25%"
      actions:
        - "Только критические релизы"
        - "Feature freeze для рискованных изменений"
        - "Ежедневный мониторинг"

    critical:
      budget_remaining: "<10%"
      actions:
        - "Полный feature freeze"
        - "Все ресурсы на стабилизацию"
        - "Эскалация руководству"

    exhausted:
      budget_remaining: "<=0%"
      actions:
        - "Инцидент-режим"
        - "Postmortem обязателен"
        - "Возврат к последней стабильной версии"

  escalation:
    - level: "Team Lead"
      trigger: "budget < 25%"
    - level: "Engineering Manager"
      trigger: "budget < 10%"
    - level: "VP Engineering"
      trigger: "budget exhausted"
```

### 2. Справедливое распределение бюджета

```yaml
# Распределение Error Budget между командами/активностями
error_budget_allocation:
  total_budget: "0.1%"  # SLO = 99.9%

  allocation:
    - category: "Planned deployments"
      budget: "40%"
      owner: "Development Team"

    - category: "Infrastructure changes"
      budget: "20%"
      owner: "Platform Team"

    - category: "Unexpected incidents"
      budget: "30%"
      owner: "All teams"

    - category: "Chaos engineering"
      budget: "10%"
      owner: "SRE Team"
```

### 3. Burn Rate алерты

```yaml
# Настройка алертов на основе burn rate
burn_rate_alerts:
  # Быстрое сжигание - требует немедленной реакции
  - name: "Fast Burn"
    short_window: "5m"
    long_window: "1h"
    burn_rate: 14.4
    action: "Page on-call"
    time_to_exhaustion: "2 hours"

  # Среднее сжигание - требует внимания
  - name: "Medium Burn"
    short_window: "30m"
    long_window: "6h"
    burn_rate: 6
    action: "Page on-call"
    time_to_exhaustion: "5 hours"

  # Медленное сжигание - создать тикет
  - name: "Slow Burn"
    short_window: "2h"
    long_window: "24h"
    burn_rate: 3
    action: "Create ticket"
    time_to_exhaustion: "10 hours"

  # Очень медленное - следить
  - name: "Very Slow Burn"
    short_window: "6h"
    long_window: "3d"
    burn_rate: 1
    action: "Add to weekly review"
    time_to_exhaustion: "30 days"
```

## Определение SLA

### 1. SLA должен быть проще SLO

```yaml
# Внутренний SLO (строже)
internal_slo:
  availability: 99.95%
  latency_p99: 300ms
  measurement_window: "30 days"

# Внешний SLA (мягче, с буфером)
external_sla:
  availability: 99.9%
  latency_p99: 500ms
  measurement_window: "Calendar month"
  buffer: "0.05% availability, 200ms latency"
```

### 2. Четкие определения и исключения

```markdown
# Пример секции определений в SLA

## Определения

**"Доступность"** - процент времени, когда Сервис отвечает
на HTTP запросы с кодом статуса < 500.

**"Время ответа"** - время от получения запроса до отправки
первого байта ответа, измеренное на стороне сервера.

**"Период измерения"** - календарный месяц в часовом поясе UTC.

## Исключения

Следующие ситуации не учитываются при расчете SLA:
1. Плановое техническое обслуживание (уведомление за 7 дней)
2. Форс-мажорные обстоятельства
3. Действия или бездействие Клиента
4. Проблемы сторонних провайдеров
5. DDoS-атаки (подтвержденные)
```

### 3. Разумные компенсации

```yaml
service_credits:
  calculation_basis: "Monthly invoice amount"

  tiers:
    - availability_range: "99.0% - 99.9%"
      credit_percentage: 10%
      description: "Minor SLA breach"

    - availability_range: "95.0% - 99.0%"
      credit_percentage: 25%
      description: "Moderate SLA breach"

    - availability_range: "< 95.0%"
      credit_percentage: 50%
      description: "Severe SLA breach"

  max_credit: "50% of monthly invoice"

  claim_process:
    deadline: "30 days after month end"
    required_info:
      - "Affected time period"
      - "Impact description"
      - "Request IDs (if applicable)"
```

## Организационные практики

### 1. Владение SLO

```yaml
slo_ownership:
  service: "payment-api"

  roles:
    slo_owner:
      team: "Payments Team"
      responsibilities:
        - "Определение и пересмотр SLO"
        - "Мониторинг error budget"
        - "Принятие решений по релизам"

    slo_contributor:
      team: "Platform Team"
      responsibilities:
        - "Инфраструктурная поддержка"
        - "Помощь в инцидентах"

    slo_stakeholder:
      role: "Product Manager"
      responsibilities:
        - "Согласование SLO с бизнес-требованиями"
        - "Участие в review"
```

### 2. Регулярный пересмотр SLO

```yaml
slo_review_process:
  frequency: "Quarterly"

  participants:
    - "SLO Owner"
    - "Engineering Lead"
    - "Product Manager"
    - "SRE representative"

  agenda:
    - topic: "SLO compliance review"
      duration: "15 min"
      questions:
        - "Выполнялся ли SLO?"
        - "Сколько было инцидентов?"

    - topic: "Error budget analysis"
      duration: "15 min"
      questions:
        - "Как расходовался error budget?"
        - "Были ли проблемы с политикой?"

    - topic: "SLO adjustment discussion"
      duration: "20 min"
      questions:
        - "Соответствует ли SLO ожиданиям пользователей?"
        - "Нужно ли изменить целевые значения?"

    - topic: "Action items"
      duration: "10 min"

  outputs:
    - "Updated SLO document (if changes)"
    - "Action items with owners"
    - "Meeting notes"
```

### 3. Культура SLO

```yaml
slo_culture_practices:
  communication:
    - "Публикуйте SLO статус в общих каналах"
    - "Празднуйте достижение SLO"
    - "Blameless postmortems при нарушениях"

  empowerment:
    - "Команды владеют своими SLO"
    - "Свобода принимать решения на основе error budget"
    - "Поддержка экспериментов в рамках бюджета"

  learning:
    - "Обучение новых сотрудников концепциям SLO"
    - "Sharing best practices между командами"
    - "Внешние ресурсы (книги, конференции)"
```

## Типичные ошибки и как их избежать

### 1. SLO как наказание

```yaml
# Плохо: SLO для контроля команды
bad_practice:
  behavior: "Связывать бонусы с выполнением SLO"
  problem: "Команды будут ставить заниженные SLO"
  result: "SLO теряют смысл"

# Хорошо: SLO как инструмент
good_practice:
  behavior: "Использовать SLO для принятия решений"
  benefit: "Команды заинтересованы в правильных SLO"
  result: "SLO отражают реальные потребности"
```

### 2. Слишком много SLI

```yaml
# Плохо: измерять всё
too_many_slis:
  count: 50
  problem: "Сложно следить, неясны приоритеты"

# Хорошо: фокус на важном
right_number:
  count: 3-5
  criteria:
    - "Отражает пользовательский опыт"
    - "Actionable"
    - "Понятен всем"
```

### 3. Игнорирование зависимостей

```yaml
# Плохо: изолированный SLO
isolated_slo:
  service: "frontend"
  availability: "99.99%"
  problem: "Backend имеет SLO 99.9%"
  result: "Невозможно достичь SLO"

# Хорошо: учёт всей цепочки
chain_aware_slo:
  frontend:
    target: "99.9%"
    note: "Учитывает SLO всех зависимостей"
  dependencies:
    - backend: "99.95%"
    - database: "99.99%"
```

### 4. Отсутствие Error Budget политики

```yaml
# Плохо: SLO без последствий
no_policy:
  problem: "SLO нарушен, но ничего не происходит"
  result: "SLO не воспринимают серьёзно"

# Хорошо: чёткие правила
with_policy:
  trigger: "Error budget < 10%"
  actions:
    - "Feature freeze"
    - "Escalation"
    - "Action plan required"
```

### 5. SLO установлен и забыт

```yaml
# Плохо: статичный SLO
static_slo:
  created: "2 года назад"
  reviewed: "никогда"
  problem: "Не соответствует текущим требованиям"

# Хорошо: живой документ
living_slo:
  created: "2 года назад"
  last_review: "1 месяц назад"
  next_review: "через 2 месяца"
  changes_log:
    - "Q1 2024: повысили latency SLO с 500ms до 300ms"
    - "Q3 2023: добавили throughput SLI"
```

## Checklist для внедрения SLO

### Подготовка

```yaml
preparation:
  - task: "Определить критические user journeys"
    status: "[ ]"
  - task: "Выбрать SLI на основе user journeys"
    status: "[ ]"
  - task: "Настроить сбор метрик"
    status: "[ ]"
  - task: "Собрать исторические данные (минимум 30 дней)"
    status: "[ ]"
```

### Определение SLO

```yaml
slo_definition:
  - task: "Установить первоначальные SLO на основе данных"
    status: "[ ]"
  - task: "Согласовать SLO с stakeholders"
    status: "[ ]"
  - task: "Задокументировать SLO"
    status: "[ ]"
  - task: "Настроить дашборды"
    status: "[ ]"
```

### Операционализация

```yaml
operationalization:
  - task: "Настроить алерты (burn rate)"
    status: "[ ]"
  - task: "Написать Error Budget политику"
    status: "[ ]"
  - task: "Обучить команду"
    status: "[ ]"
  - task: "Провести dry-run (2-4 недели)"
    status: "[ ]"
```

### Запуск и итерация

```yaml
launch_and_iterate:
  - task: "Официальный запуск SLO"
    status: "[ ]"
  - task: "Запланировать первый review"
    status: "[ ]"
  - task: "Собрать feedback"
    status: "[ ]"
  - task: "Итерировать на основе опыта"
    status: "[ ]"
```

## Рекомендуемые ресурсы

### Книги

```yaml
books:
  - title: "Site Reliability Engineering"
    author: "Google"
    focus: "Основы SRE и SLO"

  - title: "The Site Reliability Workbook"
    author: "Google"
    focus: "Практические примеры"

  - title: "Implementing Service Level Objectives"
    author: "Alex Hidalgo"
    focus: "Глубокое погружение в SLO"
```

### Инструменты

```yaml
tools:
  slo_generation:
    - name: "Sloth"
      url: "https://github.com/slok/sloth"
    - name: "OpenSLO"
      url: "https://openslo.com"

  monitoring:
    - name: "Prometheus + Grafana"
    - name: "Datadog SLO"
    - name: "Nobl9"

  incident_management:
    - name: "PagerDuty"
    - name: "Opsgenie"
    - name: "Incident.io"
```

## Заключение

Успешное внедрение SLI, SLO и SLA требует:

1. **Правильного выбора метрик** - измеряйте то, что важно для пользователей
2. **Реалистичных целей** - основывайтесь на данных, не на желаниях
3. **Чётких политик** - error budget должен влиять на решения
4. **Организационной поддержки** - SLO требует культурных изменений
5. **Постоянной итерации** - SLO должны развиваться вместе с продуктом

SLO - это не просто метрики, это способ мышления о надёжности как о продуктовой характеристике, которая требует баланса между скоростью развития и стабильностью системы.
