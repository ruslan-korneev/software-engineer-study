# Стратегии миграции (Migration Strategies)

## Определение

**Стратегии миграции** — это методы и подходы для безопасного обновления, развёртывания и трансформации программных систем с минимальным или нулевым временем простоя. Они позволяют переходить от одной версии приложения к другой, обновлять базы данных и постепенно заменять legacy-системы без нарушения работы для конечных пользователей.

```
┌─────────────────────────────────────────────────────────────────┐
│                    СТРАТЕГИИ МИГРАЦИИ                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │ Blue-Green  │  │   Canary    │  │   Rolling   │             │
│  │ Deployment  │  │  Releases   │  │   Updates   │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │  Database   │  │   Feature   │  │  Strangler  │             │
│  │ Migrations  │  │    Flags    │  │ Fig Pattern │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1. Blue-Green Deployment (Сине-зелёное развёртывание)

### Концепция

Blue-Green Deployment — стратегия, при которой поддерживаются две идентичные production-среды. В любой момент времени активна только одна из них, а вторая используется для подготовки новой версии.

```
┌─────────────────────────────────────────────────────────────────┐
│                         LOAD BALANCER                          │
│                              │                                  │
│              ┌───────────────┼───────────────┐                 │
│              ▼               │               ▼                  │
│     ┌─────────────────┐      │      ┌─────────────────┐        │
│     │   BLUE (v1.0)   │◄─────┘      │  GREEN (v1.1)   │        │
│     │    [ACTIVE]     │             │   [STANDBY]     │        │
│     └─────────────────┘             └─────────────────┘        │
│                                                                 │
│                    После переключения:                         │
│                                                                 │
│     ┌─────────────────┐             ┌─────────────────┐        │
│     │   BLUE (v1.0)   │             │  GREEN (v1.1)   │        │
│     │   [STANDBY]     │      ┌─────►│    [ACTIVE]     │        │
│     └─────────────────┘      │      └─────────────────┘        │
│                              │                                  │
│                         LOAD BALANCER                          │
└─────────────────────────────────────────────────────────────────┘
```

### Пример конфигурации Kubernetes

```yaml
# blue-deployment.yaml
# Текущая активная версия приложения (синяя среда)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
  labels:
    app: myapp
    version: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
      - name: myapp
        image: myapp:1.0.0
        ports:
        - containerPort: 8080
        # Проверка готовности приложения
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        # Проверка жизнеспособности
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 20
---
# green-deployment.yaml
# Новая версия приложения для тестирования (зелёная среда)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
  labels:
    app: myapp
    version: green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
      - name: myapp
        image: myapp:1.1.0  # Новая версия
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
---
# service.yaml
# Сервис для переключения между blue и green
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
    version: blue  # Переключение: измените на 'green' для смены версии
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer
```

### Скрипт переключения

```bash
#!/bin/bash
# blue-green-switch.sh
# Скрипт для безопасного переключения между blue и green средами

CURRENT_VERSION=$(kubectl get svc myapp-service -o jsonpath='{.spec.selector.version}')

if [ "$CURRENT_VERSION" == "blue" ]; then
    NEW_VERSION="green"
else
    NEW_VERSION="blue"
fi

echo "Текущая версия: $CURRENT_VERSION"
echo "Переключение на: $NEW_VERSION"

# Проверяем готовность новой версии
READY_PODS=$(kubectl get pods -l app=myapp,version=$NEW_VERSION \
    -o jsonpath='{.items[*].status.conditions[?(@.type=="Ready")].status}' | tr ' ' '\n' | grep -c True)

TOTAL_PODS=$(kubectl get pods -l app=myapp,version=$NEW_VERSION --no-headers | wc -l)

if [ "$READY_PODS" -lt "$TOTAL_PODS" ]; then
    echo "ОШИБКА: Не все поды $NEW_VERSION готовы ($READY_PODS/$TOTAL_PODS)"
    exit 1
fi

# Выполняем переключение
kubectl patch svc myapp-service -p "{\"spec\":{\"selector\":{\"version\":\"$NEW_VERSION\"}}}"

echo "Переключение выполнено успешно!"
echo "Для отката выполните скрипт повторно"
```

---

## 2. Canary Releases (Канареечные релизы)

### Концепция

Canary Release — это стратегия постепенного выкатывания новой версии на небольшой процент пользователей, с постепенным увеличением охвата при отсутствии проблем.

```
┌─────────────────────────────────────────────────────────────────┐
│                      CANARY RELEASE ЭТАПЫ                       │
│                                                                 │
│  Этап 1 (5%)       Этап 2 (25%)      Этап 3 (50%)     Этап 4   │
│  ┌──────────┐      ┌──────────┐      ┌──────────┐    (100%)    │
│  │██████████│      │██████████│      │██████████│   ┌────────┐ │
│  │██████████│      │██████████│      │████░░░░░░│   │░░░░░░░░│ │
│  │██████████│      │██░░░░░░░░│      │░░░░░░░░░░│   │░░░░░░░░│ │
│  │█░░░░░░░░░│      │░░░░░░░░░░│      │░░░░░░░░░░│   │░░░░░░░░│ │
│  └──────────┘      └──────────┘      └──────────┘   └────────┘ │
│                                                                 │
│  ██ = Старая версия (v1.0)    ░░ = Новая версия (v1.1)        │
└─────────────────────────────────────────────────────────────────┘
```

### Конфигурация с Istio

```yaml
# canary-virtual-service.yaml
# Настройка маршрутизации трафика с помощью Istio
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp-canary
spec:
  hosts:
  - myapp.example.com
  http:
  - match:
    - headers:
        x-canary:
          exact: "true"  # Принудительное направление на canary по заголовку
    route:
    - destination:
        host: myapp-canary
        port:
          number: 80
  - route:
    # 95% трафика на стабильную версию
    - destination:
        host: myapp-stable
        port:
          number: 80
      weight: 95
    # 5% трафика на canary версию
    - destination:
        host: myapp-canary
        port:
          number: 80
      weight: 5
---
# destination-rules.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: myapp-destination
spec:
  host: myapp
  subsets:
  - name: stable
    labels:
      version: stable
  - name: canary
    labels:
      version: canary
```

### Автоматизация Canary с Flagger

```yaml
# flagger-canary.yaml
# Автоматический canary deployment с Flagger
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: myapp
  namespace: production
spec:
  # Ссылка на целевой deployment
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  # Сервис для маршрутизации
  service:
    port: 80
    targetPort: 8080
  # Анализ метрик
  analysis:
    # Интервал между увеличением трафика
    interval: 1m
    # Пороговое значение успешных проверок для продолжения
    threshold: 5
    # Максимальное количество неудачных проверок до отката
    maxWeight: 50
    # Шаг увеличения трафика (в процентах)
    stepWeight: 10
    # Метрики для анализа
    metrics:
    - name: request-success-rate
      # Требуем 99% успешных запросов
      thresholdRange:
        min: 99
      interval: 1m
    - name: request-duration
      # P99 задержка не более 500ms
      thresholdRange:
        max: 500
      interval: 1m
    # Webhook для дополнительных проверок
    webhooks:
    - name: load-test
      url: http://flagger-loadtester/
      timeout: 5s
      metadata:
        cmd: "hey -z 1m -q 10 -c 2 http://myapp-canary:80/"
```

### Скрипт мониторинга canary

```python
#!/usr/bin/env python3
"""
canary_monitor.py
Мониторинг метрик canary deployment и автоматический откат при проблемах
"""

import time
import requests
from prometheus_api_client import PrometheusConnect

class CanaryMonitor:
    def __init__(self, prometheus_url: str, canary_service: str):
        self.prom = PrometheusConnect(url=prometheus_url)
        self.canary_service = canary_service
        # Пороговые значения для отката
        self.error_rate_threshold = 0.01  # 1% ошибок
        self.latency_p99_threshold = 500  # 500ms

    def get_error_rate(self) -> float:
        """Получение процента ошибок за последние 5 минут"""
        query = f'''
            sum(rate(http_requests_total{{
                service="{self.canary_service}",
                status=~"5.."
            }}[5m])) /
            sum(rate(http_requests_total{{
                service="{self.canary_service}"
            }}[5m]))
        '''
        result = self.prom.custom_query(query)
        return float(result[0]['value'][1]) if result else 0.0

    def get_latency_p99(self) -> float:
        """Получение P99 задержки за последние 5 минут"""
        query = f'''
            histogram_quantile(0.99,
                sum(rate(http_request_duration_seconds_bucket{{
                    service="{self.canary_service}"
                }}[5m])) by (le)
            ) * 1000
        '''
        result = self.prom.custom_query(query)
        return float(result[0]['value'][1]) if result else 0.0

    def should_rollback(self) -> tuple[bool, str]:
        """Проверка необходимости отката"""
        error_rate = self.get_error_rate()
        latency = self.get_latency_p99()

        if error_rate > self.error_rate_threshold:
            return True, f"Высокий процент ошибок: {error_rate:.2%}"

        if latency > self.latency_p99_threshold:
            return True, f"Высокая задержка P99: {latency:.0f}ms"

        return False, "Метрики в норме"

    def monitor_loop(self, check_interval: int = 60):
        """Основной цикл мониторинга"""
        print(f"Начало мониторинга canary: {self.canary_service}")

        while True:
            should_rollback, reason = self.should_rollback()

            if should_rollback:
                print(f"⚠️ ОТКАТ НЕОБХОДИМ: {reason}")
                self.trigger_rollback()
                break
            else:
                print(f"✓ {reason}")

            time.sleep(check_interval)

    def trigger_rollback(self):
        """Инициация отката canary deployment"""
        # Вызов webhook или kubectl для отката
        print("Выполняется откат...")
        # kubectl rollout undo deployment/myapp-canary

if __name__ == "__main__":
    monitor = CanaryMonitor(
        prometheus_url="http://prometheus:9090",
        canary_service="myapp-canary"
    )
    monitor.monitor_loop()
```

---

## 3. Rolling Updates (Последовательные обновления)

### Концепция

Rolling Update — стратегия обновления, при которой старые экземпляры постепенно заменяются новыми без прерывания обслуживания.

```
┌─────────────────────────────────────────────────────────────────┐
│                    ROLLING UPDATE ПРОЦЕСС                       │
│                                                                 │
│  Начало:     [v1] [v1] [v1] [v1] [v1]                          │
│                                                                 │
│  Шаг 1:      [v2] [v1] [v1] [v1] [v1]   ← Один под обновлён    │
│                                                                 │
│  Шаг 2:      [v2] [v2] [v1] [v1] [v1]   ← Два пода обновлены   │
│                                                                 │
│  Шаг 3:      [v2] [v2] [v2] [v1] [v1]   ← Три пода обновлены   │
│                                                                 │
│  Шаг 4:      [v2] [v2] [v2] [v2] [v1]   ← Четыре пода          │
│                                                                 │
│  Конец:      [v2] [v2] [v2] [v2] [v2]   ← Все обновлены        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Конфигурация Kubernetes

```yaml
# rolling-update-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 5
  # Стратегия обновления
  strategy:
    type: RollingUpdate
    rollingUpdate:
      # Максимум подов, которые могут быть недоступны во время обновления
      maxUnavailable: 1
      # Максимум дополнительных подов сверх replicas
      maxSurge: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:1.0.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        # Критически важно для rolling update
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
      # Время на graceful shutdown
      terminationGracePeriodSeconds: 30
```

### Хуки жизненного цикла

```yaml
# lifecycle-hooks.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-with-hooks
spec:
  template:
    spec:
      containers:
      - name: myapp
        image: myapp:1.0.0
        lifecycle:
          # Хук после запуска контейнера
          postStart:
            exec:
              command:
              - /bin/sh
              - -c
              - |
                echo "Контейнер запущен, выполняем инициализацию..."
                # Прогрев кеша, регистрация в service discovery и т.д.
                curl -X POST http://localhost:8080/warmup
          # Хук перед остановкой контейнера
          preStop:
            exec:
              command:
              - /bin/sh
              - -c
              - |
                echo "Подготовка к остановке..."
                # Дерегистрация из service discovery
                curl -X POST http://localhost:8080/drain
                # Ожидание завершения текущих запросов
                sleep 10
```

---

## 4. Миграции баз данных (Database Migrations)

### Концепция

Миграции БД — это версионированные изменения схемы базы данных, которые позволяют эволюционировать структуру данных без потери информации и без даунтайма.

```
┌─────────────────────────────────────────────────────────────────┐
│              EXPAND AND CONTRACT PATTERN                        │
│                                                                 │
│  Этап 1: EXPAND (расширение)                                   │
│  ┌─────────────────┐                                           │
│  │ users           │    Добавляем новый столбец                │
│  │ ├─ id           │    full_name, старый name остаётся        │
│  │ ├─ name         │                                           │
│  │ └─ full_name ←──│─── НОВЫЙ                                  │
│  └─────────────────┘                                           │
│                                                                 │
│  Этап 2: MIGRATE (миграция данных)                             │
│  ┌─────────────────┐                                           │
│  │ UPDATE users    │    Копируем данные:                       │
│  │ SET full_name   │    name → full_name                       │
│  │ = name          │                                           │
│  └─────────────────┘                                           │
│                                                                 │
│  Этап 3: CONTRACT (сжатие)                                     │
│  ┌─────────────────┐                                           │
│  │ users           │    Удаляем старый столбец                 │
│  │ ├─ id           │    после обновления всего кода            │
│  │ └─ full_name    │                                           │
│  └─────────────────┘                                           │
└─────────────────────────────────────────────────────────────────┘
```

### Пример миграций с Alembic (Python/SQLAlchemy)

```python
# migrations/versions/001_add_full_name_column.py
"""
Миграция: Добавление столбца full_name
Expand фаза - добавляем новый столбец, сохраняя старый
"""

from alembic import op
import sqlalchemy as sa

# Идентификатор ревизии
revision = '001_add_full_name'
down_revision = None
branch_labels = None
depends_on = None


def upgrade():
    """
    Expand фаза: добавляем новый столбец full_name
    Старый столбец name остаётся для обратной совместимости
    """
    # Добавляем новый столбец (nullable для совместимости)
    op.add_column(
        'users',
        sa.Column('full_name', sa.String(255), nullable=True)
    )

    # Создаём индекс для нового столбца
    op.create_index('ix_users_full_name', 'users', ['full_name'])


def downgrade():
    """Откат: удаляем столбец full_name"""
    op.drop_index('ix_users_full_name', table_name='users')
    op.drop_column('users', 'full_name')
```

```python
# migrations/versions/002_migrate_name_data.py
"""
Миграция: Копирование данных из name в full_name
Migrate фаза - переносим данные батчами для избежания блокировок
"""

from alembic import op
from sqlalchemy import text

revision = '002_migrate_name_data'
down_revision = '001_add_full_name'


def upgrade():
    """
    Батчевое копирование данных из name в full_name
    Используем небольшие батчи для минимизации блокировок
    """
    connection = op.get_bind()

    batch_size = 1000
    offset = 0

    while True:
        # Обновляем батч записей
        result = connection.execute(text(f"""
            UPDATE users
            SET full_name = name
            WHERE id IN (
                SELECT id FROM users
                WHERE full_name IS NULL
                LIMIT {batch_size}
            )
        """))

        # Если обновлено 0 записей - завершаем
        if result.rowcount == 0:
            break

        print(f"Обновлено {result.rowcount} записей")


def downgrade():
    """Откат данных не требуется - данные остаются в name"""
    pass
```

```python
# migrations/versions/003_remove_name_column.py
"""
Миграция: Удаление старого столбца name
Contract фаза - выполняется ПОСЛЕ обновления всего кода
"""

from alembic import op
import sqlalchemy as sa

revision = '003_remove_name_column'
down_revision = '002_migrate_name_data'


def upgrade():
    """
    Contract фаза: удаляем старый столбец
    ВАЖНО: выполнять только после обновления всего кода!
    """
    # Сначала делаем full_name обязательным
    op.alter_column(
        'users',
        'full_name',
        nullable=False,
        existing_type=sa.String(255)
    )

    # Удаляем старый столбец
    op.drop_column('users', 'name')


def downgrade():
    """Откат: восстанавливаем столбец name"""
    op.add_column(
        'users',
        sa.Column('name', sa.String(255), nullable=True)
    )

    # Копируем данные обратно
    op.execute("UPDATE users SET name = full_name")

    op.alter_column('users', 'full_name', nullable=True)
```

### Zero-downtime миграция с переименованием таблицы

```sql
-- migration_rename_table_zero_downtime.sql
-- Безопасное переименование таблицы без даунтайма

-- Шаг 1: Создаём новую таблицу с нужным именем
CREATE TABLE orders_new (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL,
    total_amount DECIMAL(10,2) NOT NULL,
    status VARCHAR(50) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT fk_user FOREIGN KEY (user_id) REFERENCES users(id)
);

-- Шаг 2: Создаём триггер для синхронизации данных
CREATE OR REPLACE FUNCTION sync_orders_to_new()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO orders_new (id, user_id, total_amount, status, created_at)
        VALUES (NEW.id, NEW.user_id, NEW.total_amount, NEW.status, NEW.created_at);
    ELSIF TG_OP = 'UPDATE' THEN
        UPDATE orders_new SET
            user_id = NEW.user_id,
            total_amount = NEW.total_amount,
            status = NEW.status
        WHERE id = NEW.id;
    ELSIF TG_OP = 'DELETE' THEN
        DELETE FROM orders_new WHERE id = OLD.id;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER orders_sync_trigger
AFTER INSERT OR UPDATE OR DELETE ON orders
FOR EACH ROW EXECUTE FUNCTION sync_orders_to_new();

-- Шаг 3: Копируем существующие данные батчами
DO $$
DECLARE
    batch_size INTEGER := 10000;
    max_id INTEGER;
    current_id INTEGER := 0;
BEGIN
    SELECT MAX(id) INTO max_id FROM orders;

    WHILE current_id < max_id LOOP
        INSERT INTO orders_new (id, user_id, total_amount, status, created_at)
        SELECT id, user_id, total_amount, status, created_at
        FROM orders
        WHERE id > current_id AND id <= current_id + batch_size
        ON CONFLICT (id) DO NOTHING;

        current_id := current_id + batch_size;
        RAISE NOTICE 'Скопировано до id: %', current_id;
    END LOOP;
END $$;

-- Шаг 4: После обновления всего кода - переключаемся
-- (выполняется в отдельной миграции)
DROP TRIGGER orders_sync_trigger ON orders;
DROP FUNCTION sync_orders_to_new();
ALTER TABLE orders RENAME TO orders_old;
ALTER TABLE orders_new RENAME TO orders;

-- Шаг 5: Удаляем старую таблицу (через несколько дней)
DROP TABLE orders_old;
```

---

## 5. Feature Flags (Флаги функций)

### Концепция

Feature Flags — это механизм включения/выключения функциональности в runtime без развёртывания нового кода. Позволяет разделить деплой кода и релиз функций.

```
┌─────────────────────────────────────────────────────────────────┐
│                     FEATURE FLAGS АРХИТЕКТУРА                   │
│                                                                 │
│  ┌─────────────┐      ┌─────────────────────────────────────┐  │
│  │ Приложение  │◄────►│        Feature Flag Service          │  │
│  └─────────────┘      │  ┌─────────────────────────────────┐│  │
│                       │  │ new_checkout: true              ││  │
│                       │  │ dark_mode: { users: [1,2,3] }   ││  │
│                       │  │ beta_feature: { percent: 10 }   ││  │
│                       │  └─────────────────────────────────┘│  │
│                       └─────────────────────────────────────┘  │
│                                                                 │
│  Типы флагов:                                                  │
│  • Boolean - простое вкл/выкл                                  │
│  • Percentage - для определённого % пользователей              │
│  • User-based - для конкретных пользователей                   │
│  • Environment - зависит от окружения                          │
└─────────────────────────────────────────────────────────────────┘
```

### Реализация Feature Flag сервиса

```python
# feature_flags/service.py
"""
Сервис управления Feature Flags
Поддерживает различные стратегии включения функций
"""

from enum import Enum
from dataclasses import dataclass
from typing import Any, Optional
import hashlib
import json
import redis


class FlagStrategy(Enum):
    """Стратегии включения флага"""
    BOOLEAN = "boolean"           # Простое вкл/выкл
    PERCENTAGE = "percentage"     # Процент пользователей
    USER_LIST = "user_list"       # Список конкретных пользователей
    GRADUAL_ROLLOUT = "gradual"   # Постепенный выкат


@dataclass
class FeatureFlag:
    """Модель флага функции"""
    name: str
    enabled: bool
    strategy: FlagStrategy
    config: dict


class FeatureFlagService:
    """Сервис управления feature flags"""

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
        self.cache_ttl = 60  # Кеш на 60 секунд

    def is_enabled(
        self,
        flag_name: str,
        user_id: Optional[str] = None,
        context: Optional[dict] = None
    ) -> bool:
        """
        Проверка, включён ли флаг для пользователя

        Args:
            flag_name: Имя флага
            user_id: ID пользователя (для персонализированных флагов)
            context: Дополнительный контекст

        Returns:
            True если функция включена для пользователя
        """
        flag = self._get_flag(flag_name)

        if flag is None or not flag.enabled:
            return False

        # Обработка по стратегии
        if flag.strategy == FlagStrategy.BOOLEAN:
            return flag.enabled

        elif flag.strategy == FlagStrategy.PERCENTAGE:
            if user_id is None:
                return False
            return self._check_percentage(flag_name, user_id, flag.config.get("percentage", 0))

        elif flag.strategy == FlagStrategy.USER_LIST:
            allowed_users = flag.config.get("users", [])
            return user_id in allowed_users

        elif flag.strategy == FlagStrategy.GRADUAL_ROLLOUT:
            return self._check_gradual_rollout(flag_name, user_id, flag.config)

        return False

    def _check_percentage(self, flag_name: str, user_id: str, percentage: int) -> bool:
        """
        Детерминированная проверка процента
        Один и тот же пользователь всегда получает одинаковый результат
        """
        # Создаём хеш из имени флага и ID пользователя
        hash_input = f"{flag_name}:{user_id}"
        hash_value = int(hashlib.md5(hash_input.encode()).hexdigest(), 16)

        # Получаем число от 0 до 100
        user_bucket = hash_value % 100

        return user_bucket < percentage

    def _check_gradual_rollout(
        self,
        flag_name: str,
        user_id: str,
        config: dict
    ) -> bool:
        """Проверка постепенного выката по расписанию"""
        current_percentage = config.get("current_percentage", 0)
        return self._check_percentage(flag_name, user_id, current_percentage)

    def _get_flag(self, flag_name: str) -> Optional[FeatureFlag]:
        """Получение флага из Redis с кешированием"""
        data = self.redis.get(f"feature_flag:{flag_name}")

        if data is None:
            return None

        flag_data = json.loads(data)
        return FeatureFlag(
            name=flag_data["name"],
            enabled=flag_data["enabled"],
            strategy=FlagStrategy(flag_data["strategy"]),
            config=flag_data.get("config", {})
        )

    def set_flag(self, flag: FeatureFlag) -> None:
        """Установка флага"""
        data = {
            "name": flag.name,
            "enabled": flag.enabled,
            "strategy": flag.strategy.value,
            "config": flag.config
        }
        self.redis.set(
            f"feature_flag:{flag.name}",
            json.dumps(data)
        )

    def update_percentage(self, flag_name: str, new_percentage: int) -> None:
        """Обновление процента для gradual rollout"""
        flag = self._get_flag(flag_name)
        if flag:
            flag.config["current_percentage"] = new_percentage
            self.set_flag(flag)


# Использование в коде приложения
class CheckoutService:
    """Пример использования feature flags в бизнес-логике"""

    def __init__(self, feature_flags: FeatureFlagService):
        self.flags = feature_flags

    def process_checkout(self, user_id: str, cart: dict) -> dict:
        """Обработка оформления заказа"""

        # Проверяем, включена ли новая версия checkout для пользователя
        if self.flags.is_enabled("new_checkout_flow", user_id):
            return self._new_checkout(user_id, cart)
        else:
            return self._legacy_checkout(user_id, cart)

    def _new_checkout(self, user_id: str, cart: dict) -> dict:
        """Новая версия checkout с улучшенным UX"""
        # Новая логика
        return {"version": "new", "status": "success"}

    def _legacy_checkout(self, user_id: str, cart: dict) -> dict:
        """Старая версия checkout"""
        # Старая логика
        return {"version": "legacy", "status": "success"}
```

### Feature Flag Decorator

```python
# feature_flags/decorators.py
"""Декораторы для упрощения работы с feature flags"""

from functools import wraps
from typing import Callable, Optional


def feature_flag(
    flag_name: str,
    fallback: Optional[Callable] = None,
    default: bool = False
):
    """
    Декоратор для включения функции по флагу

    Args:
        flag_name: Имя флага
        fallback: Альтернативная функция при выключенном флаге
        default: Значение по умолчанию если флаг не найден
    """
    def decorator(func: Callable):
        @wraps(func)
        def wrapper(*args, **kwargs):
            # Получаем сервис флагов из контекста
            from flask import g, current_app

            flags_service = current_app.config.get('FEATURE_FLAGS')
            user_id = getattr(g, 'user_id', None)

            if flags_service and flags_service.is_enabled(flag_name, user_id):
                return func(*args, **kwargs)
            elif fallback:
                return fallback(*args, **kwargs)
            elif default:
                return func(*args, **kwargs)
            else:
                # Флаг выключен и нет fallback
                raise FeatureDisabledError(f"Feature '{flag_name}' is disabled")

        return wrapper
    return decorator


class FeatureDisabledError(Exception):
    """Исключение для выключенной функции"""
    pass


# Пример использования декоратора
@feature_flag("new_recommendation_engine", fallback=get_legacy_recommendations)
def get_recommendations(user_id: str) -> list:
    """Новый алгоритм рекомендаций"""
    return advanced_ml_recommendations(user_id)


def get_legacy_recommendations(user_id: str) -> list:
    """Старый алгоритм рекомендаций"""
    return simple_recommendations(user_id)
```

---

## 6. Strangler Fig Pattern (Паттерн удушающей фиги)

### Концепция

Strangler Fig Pattern — это стратегия постепенной замены legacy-системы путём создания новых компонентов вокруг старой системы, подобно тому как фиговое дерево постепенно окутывает и замещает дерево-хозяина.

```
┌─────────────────────────────────────────────────────────────────┐
│                  STRANGLER FIG PATTERN                          │
│                                                                 │
│  Этап 1: Монолит                                               │
│  ┌─────────────────────────────────────────────┐               │
│  │                 LEGACY SYSTEM                │               │
│  │  [Users] [Orders] [Products] [Payments]     │               │
│  └─────────────────────────────────────────────┘               │
│                                                                 │
│  Этап 2: Facade + первый сервис                                │
│  ┌─────────────────────────────────────────────┐               │
│  │                  API FACADE                  │               │
│  └──────┬──────────────┬───────────────────────┘               │
│         │              │                                        │
│         ▼              ▼                                        │
│  ┌────────────┐  ┌─────────────────────────────┐               │
│  │ New Users  │  │         LEGACY              │               │
│  │  Service   │  │  [Orders][Products][Pay]    │               │
│  └────────────┘  └─────────────────────────────┘               │
│                                                                 │
│  Этап 3: Больше сервисов                                       │
│  ┌─────────────────────────────────────────────┐               │
│  │                  API FACADE                  │               │
│  └──┬──────┬──────────┬───────────────────────┘               │
│     │      │          │                                        │
│     ▼      ▼          ▼                                        │
│  ┌──────┐┌──────┐┌─────────────────┐                          │
│  │Users ││Orders││    LEGACY       │                          │
│  └──────┘└──────┘│ [Products][Pay] │                          │
│                  └─────────────────┘                          │
│                                                                 │
│  Этап 4: Полная миграция                                       │
│  ┌─────────────────────────────────────────────┐               │
│  │                  API GATEWAY                 │               │
│  └──┬──────┬──────────┬──────────┬────────────┘               │
│     │      │          │          │                             │
│     ▼      ▼          ▼          ▼                             │
│  ┌──────┐┌──────┐┌────────┐┌──────────┐                       │
│  │Users ││Orders││Products││ Payments │                       │
│  └──────┘└──────┘└────────┘└──────────┘                       │
└─────────────────────────────────────────────────────────────────┘
```

### Реализация Strangler Facade

```python
# strangler/facade.py
"""
Facade для постепенной миграции с legacy на новые микросервисы
"""

from abc import ABC, abstractmethod
from typing import Any, Dict
import httpx
from enum import Enum


class ServiceTarget(Enum):
    """Куда направлять запрос"""
    LEGACY = "legacy"
    NEW = "new"


class StranglerFacade:
    """
    Фасад для маршрутизации запросов между legacy и новыми сервисами
    """

    def __init__(self, config: Dict[str, Any]):
        self.legacy_url = config["legacy_url"]
        self.services_config = config.get("services", {})
        self.http_client = httpx.AsyncClient(timeout=30.0)

    async def route_request(
        self,
        service: str,
        endpoint: str,
        method: str = "GET",
        data: Any = None,
        headers: Dict[str, str] = None
    ) -> httpx.Response:
        """
        Маршрутизация запроса на соответствующий сервис
        """
        target = self._get_target(service, endpoint)

        if target == ServiceTarget.NEW:
            url = self._build_new_service_url(service, endpoint)
        else:
            url = self._build_legacy_url(service, endpoint)

        # Выполняем запрос
        response = await self.http_client.request(
            method=method,
            url=url,
            json=data,
            headers=headers or {}
        )

        return response

    def _get_target(self, service: str, endpoint: str) -> ServiceTarget:
        """
        Определение целевой системы на основе конфигурации
        """
        service_config = self.services_config.get(service, {})

        # Проверяем, мигрирован ли сервис
        if service_config.get("migrated", False):
            return ServiceTarget.NEW

        # Проверяем конкретный endpoint
        migrated_endpoints = service_config.get("migrated_endpoints", [])
        if endpoint in migrated_endpoints:
            return ServiceTarget.NEW

        return ServiceTarget.LEGACY

    def _build_new_service_url(self, service: str, endpoint: str) -> str:
        """Построение URL для нового микросервиса"""
        service_url = self.services_config[service]["url"]
        return f"{service_url}{endpoint}"

    def _build_legacy_url(self, service: str, endpoint: str) -> str:
        """Построение URL для legacy системы"""
        return f"{self.legacy_url}/api/{service}{endpoint}"


# Конфигурация миграции
migration_config = {
    "legacy_url": "http://legacy-monolith:8080",
    "services": {
        "users": {
            "migrated": True,  # Сервис полностью мигрирован
            "url": "http://users-service:8080"
        },
        "orders": {
            "migrated": False,  # Частично мигрирован
            "url": "http://orders-service:8080",
            "migrated_endpoints": [
                "/create",
                "/list",
                # /details и /cancel ещё на legacy
            ]
        },
        "products": {
            "migrated": False,  # Ещё на legacy
            "url": None
        },
        "payments": {
            "migrated": False,
            "url": None
        }
    }
}
```

### API Gateway конфигурация (Kong)

```yaml
# kong-strangler-config.yaml
# Конфигурация Kong для strangler pattern

_format_version: "3.0"

services:
  # Полностью мигрированный сервис Users
  - name: users-service
    url: http://users-service:8080
    routes:
    - name: users-routes
      paths:
      - /api/users
      strip_path: true

  # Частично мигрированный сервис Orders
  - name: orders-service-new
    url: http://orders-service:8080
    routes:
    - name: orders-new-routes
      paths:
      - /api/orders/create
      - /api/orders/list
      strip_path: false

  # Legacy для оставшихся endpoints Orders
  - name: orders-service-legacy
    url: http://legacy-monolith:8080
    routes:
    - name: orders-legacy-routes
      paths:
      - /api/orders/details
      - /api/orders/cancel
      strip_path: false

  # Полностью legacy сервисы
  - name: legacy-products
    url: http://legacy-monolith:8080
    routes:
    - name: products-routes
      paths:
      - /api/products
      strip_path: false

  - name: legacy-payments
    url: http://legacy-monolith:8080
    routes:
    - name: payments-routes
      paths:
      - /api/payments
      strip_path: false

plugins:
  # Логирование для отслеживания миграции
  - name: file-log
    config:
      path: /var/log/kong/migration.log
      reopen: true
```

---

## Best Practices (Лучшие практики)

### 1. Общие принципы

```
┌─────────────────────────────────────────────────────────────────┐
│                      BEST PRACTICES                             │
│                                                                 │
│  ✓ Всегда имейте план отката                                   │
│  ✓ Тестируйте миграции на staging перед production             │
│  ✓ Используйте мониторинг и алерты                             │
│  ✓ Делайте бекапы перед любыми миграциями БД                   │
│  ✓ Документируйте каждый шаг миграции                          │
│  ✓ Проводите миграции в низконагруженное время                 │
│  ✓ Используйте feature flags для разделения deploy и release   │
│                                                                 │
│  ✗ НЕ делайте breaking changes в один шаг                      │
│  ✗ НЕ удаляйте старый код сразу после миграции                 │
│  ✗ НЕ игнорируйте метрики во время canary                      │
│  ✗ НЕ запускайте миграции без возможности отката               │
└─────────────────────────────────────────────────────────────────┘
```

### 2. Чеклист для миграций

```markdown
## Чеклист миграции

### Перед миграцией
- [ ] Бекап базы данных создан
- [ ] План отката документирован и протестирован
- [ ] Мониторинг и алерты настроены
- [ ] Команда оповещена о миграции
- [ ] Миграция протестирована на staging
- [ ] Нагрузочное тестирование проведено

### Во время миграции
- [ ] Метрики отслеживаются в реальном времени
- [ ] Error rate в пределах нормы
- [ ] Latency в пределах нормы
- [ ] Нет аномалий в логах

### После миграции
- [ ] Smoke тесты пройдены
- [ ] Бизнес-метрики в норме
- [ ] Пользовательские жалобы отсутствуют
- [ ] Документация обновлена
```

---

## Частые ошибки

### 1. Breaking changes без обратной совместимости

```python
# ❌ ПЛОХО: Удаление поля сразу
class UserSchema:
    id: int
    # name: str  # Удалили - сломали всех клиентов!
    full_name: str


# ✅ ХОРОШО: Deprecation с поддержкой старого поля
class UserSchema:
    id: int
    name: str  # deprecated, будет удалено в v3.0
    full_name: str  # Новое поле

    @property
    def name(self):
        """Deprecated: используйте full_name"""
        warnings.warn("name is deprecated, use full_name", DeprecationWarning)
        return self.full_name
```

### 2. Отсутствие мониторинга при canary

```python
# ❌ ПЛОХО: Canary без мониторинга
def deploy_canary():
    kubectl_apply("canary-deployment.yaml")
    print("Canary deployed!")  # И всё?

# ✅ ХОРОШО: Canary с мониторингом и автоматическим откатом
async def deploy_canary():
    kubectl_apply("canary-deployment.yaml")

    # Мониторим 10 минут
    for _ in range(10):
        metrics = await get_canary_metrics()

        if metrics.error_rate > 0.01:  # Больше 1% ошибок
            await rollback_canary()
            raise CanaryFailedError(f"Error rate too high: {metrics.error_rate}")

        if metrics.p99_latency > 500:  # Больше 500ms
            await rollback_canary()
            raise CanaryFailedError(f"Latency too high: {metrics.p99_latency}ms")

        await asyncio.sleep(60)

    print("Canary healthy, proceeding with rollout")
```

### 3. Блокирующие миграции БД

```sql
-- ❌ ПЛОХО: Блокирующее добавление NOT NULL столбца
ALTER TABLE users ADD COLUMN email VARCHAR(255) NOT NULL;
-- Заблокирует всю таблицу!

-- ✅ ХОРОШО: Поэтапное добавление
-- Шаг 1: Добавляем nullable столбец (мгновенно)
ALTER TABLE users ADD COLUMN email VARCHAR(255);

-- Шаг 2: Заполняем данные батчами
UPDATE users SET email = 'unknown@example.com'
WHERE email IS NULL AND id BETWEEN 1 AND 10000;
-- Повторяем для следующих батчей...

-- Шаг 3: После заполнения всех данных
ALTER TABLE users ALTER COLUMN email SET NOT NULL;
```

---

## Резюме

**Стратегии миграции** — критически важный инструмент для поддержания стабильности production-систем при их развитии:

| Стратегия | Когда использовать | Сложность |
|-----------|-------------------|-----------|
| **Blue-Green** | Полная замена версии с мгновенным откатом | Средняя |
| **Canary** | Постепенный выкат с валидацией на части трафика | Высокая |
| **Rolling** | Стандартное обновление без даунтайма | Низкая |
| **DB Migrations** | Эволюция схемы данных | Средняя-Высокая |
| **Feature Flags** | Разделение deploy и release | Средняя |
| **Strangler Fig** | Постепенная замена legacy-системы | Высокая |

**Ключевые принципы:**
1. Всегда имейте план отката
2. Тестируйте на staging перед production
3. Мониторьте метрики в реальном времени
4. Делайте изменения обратно совместимыми
5. Используйте поэтапный подход вместо "большого взрыва"
