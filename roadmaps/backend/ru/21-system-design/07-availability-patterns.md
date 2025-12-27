# Availability Patterns (Паттерны доступности)

## Что такое доступность (Availability)?

**Доступность (Availability)** — это процент времени, в течение которого система остаётся работоспособной и способной обрабатывать запросы. Это один из ключевых показателей надёжности любой распределённой системы.

### Формула расчёта доступности

```
Availability = (Uptime / (Uptime + Downtime)) × 100%
```

или эквивалентно:

```
Availability = MTBF / (MTBF + MTTR)
```

Где:
- **MTBF (Mean Time Between Failures)** — среднее время между отказами
- **MTTR (Mean Time To Recovery)** — среднее время восстановления после сбоя

---

## Измерение доступности: "Девятки" (Nines)

В индустрии доступность измеряется в "девятках" — количестве девяток после запятой в проценте доступности:

| Уровень | Доступность | Простой в год | Простой в месяц | Простой в неделю |
|---------|-------------|---------------|-----------------|------------------|
| 1 nine  | 90%         | 36.5 дней     | 72 часа         | 16.8 часов       |
| 2 nines | 99%         | 3.65 дней     | 7.2 часа        | 1.68 часов       |
| 3 nines | 99.9%       | 8.76 часов    | 43.8 минут      | 10.1 минут       |
| 4 nines | 99.99%      | 52.56 минут   | 4.38 минут      | 1.01 минут       |
| 5 nines | 99.999%     | 5.26 минут    | 25.9 секунд     | 6.05 секунд      |
| 6 nines | 99.9999%    | 31.5 секунд   | 2.59 секунд     | 0.605 секунд     |

### Примеры SLA различных сервисов

| Сервис | Заявленный SLA |
|--------|----------------|
| Amazon EC2 | 99.99% |
| Google Cloud Compute | 99.99% |
| Azure VMs | 99.95% - 99.99% |
| AWS S3 | 99.99% |
| Stripe API | 99.99% |

### Важно: Последовательное соединение компонентов

Если система состоит из нескольких последовательных компонентов, общая доступность вычисляется как произведение:

```
Общая доступность = A₁ × A₂ × A₃ × ... × Aₙ
```

**Пример:**
```
Веб-сервер (99.9%) → API (99.9%) → БД (99.9%)

Общая доступность = 0.999 × 0.999 × 0.999 = 99.7%
```

Это значит, что при добавлении компонентов доступность падает!

### Параллельное соединение (резервирование)

При резервировании компонентов доступность повышается:

```
Общая доступность = 1 - (1 - A₁) × (1 - A₂)
```

**Пример:**
```
Два сервера по 99%, работающих параллельно:

Доступность = 1 - (1 - 0.99) × (1 - 0.99) = 1 - 0.0001 = 99.99%
```

---

## Fail-over паттерны

**Fail-over (переключение при отказе)** — механизм автоматического переключения на резервную систему при сбое основной.

```
┌─────────────────────────────────────────────────────────────┐
│                    FAIL-OVER PATTERNS                        │
│                                                              │
│   ┌─────────────────────┐    ┌─────────────────────┐        │
│   │   Active-Passive    │    │   Active-Active     │        │
│   │                     │    │                     │        │
│   │  ┌─────┐  ┌─────┐  │    │  ┌─────┐  ┌─────┐  │        │
│   │  │ ACT │  │PASS │  │    │  │ ACT │  │ ACT │  │        │
│   │  │  ▼  │  │     │  │    │  │  ▼  │  │  ▼  │  │        │
│   │  └─────┘  └─────┘  │    │  └─────┘  └─────┘  │        │
│   │     │        │     │    │     │        │     │        │
│   │     └───→────┘     │    │     │        │     │        │
│   │    (при сбое)      │    │     └───┬────┘     │        │
│   │                     │    │         │          │        │
│   └─────────────────────┘    │   Load Balancer    │        │
│                              └─────────────────────┘        │
└─────────────────────────────────────────────────────────────┘
```

---

### Active-Passive (Master-Slave Failover)

В этом паттерне один сервер активно обрабатывает запросы, а один или несколько резервных серверов находятся в режиме ожидания.

#### Типы standby серверов:

**1. Cold Standby (Холодный резерв)**
```
┌─────────────┐         ┌─────────────┐
│   ACTIVE    │         │  STANDBY    │
│   Server    │         │   (OFF)     │
│             │         │             │
│  [Running]  │         │ [Powered    │
│             │         │    Off]     │
└─────────────┘         └─────────────┘
      │                       │
      │   Время переключения: │
      │   минуты - часы       │
      └───────────────────────┘
```

- Резервный сервер **выключен** или находится в минимальном режиме
- При сбое требуется время на запуск и настройку
- **Время переключения:** от нескольких минут до часов
- **Стоимость:** минимальная (сервер не потребляет ресурсы)
- **Применение:** некритичные системы, dev/test окружения

**2. Warm Standby (Тёплый резерв)**
```
┌─────────────┐         ┌─────────────┐
│   ACTIVE    │  Sync   │  STANDBY    │
│   Server    │◄───────►│  (Ready)    │
│             │   Data  │             │
│  [Running]  │         │ [Running,   │
│             │         │  No Traffic]│
└─────────────┘         └─────────────┘
      │                       │
      │   Время переключения: │
      │   секунды - минуты    │
      └───────────────────────┘
```

- Резервный сервер **запущен** и получает обновления данных
- Не обслуживает трафик до переключения
- **Время переключения:** от секунд до минут
- **Стоимость:** средняя (сервер работает, но без нагрузки)
- **Применение:** бизнес-критичные системы

**3. Hot Standby (Горячий резерв)**
```
┌─────────────┐         ┌─────────────┐
│   ACTIVE    │  Real-  │  STANDBY    │
│   Server    │◄═══════►│   (Hot)     │
│             │  Time   │             │
│  [Running,  │  Sync   │ [Running,   │
│   Traffic]  │         │  Ready]     │
└─────────────┘         └─────────────┘
      │                       │
      │   Время переключения: │
      │   мгновенное          │
      └───────────────────────┘
```

- Резервный сервер **полностью синхронизирован** в реальном времени
- Готов принять трафик **мгновенно**
- **Время переключения:** секунды или меньше
- **Стоимость:** высокая (полное дублирование ресурсов)
- **Применение:** mission-critical системы (банки, биржи)

#### Сравнение типов standby:

| Характеристика | Cold | Warm | Hot |
|---------------|------|------|-----|
| Время переключения | Минуты-часы | Секунды-минуты | Мгновенно |
| Стоимость | Низкая | Средняя | Высокая |
| Потеря данных | Возможна | Минимальная | Нет |
| Сложность | Низкая | Средняя | Высокая |
| Синхронизация | Нет | Периодическая | Real-time |

#### Преимущества Active-Passive:
- Простота реализации и управления
- Чёткое разделение ролей
- Меньше проблем с согласованностью данных
- Подходит для stateful приложений

#### Недостатки Active-Passive:
- Неэффективное использование ресурсов (standby простаивает)
- Время переключения может быть значительным
- Один сервер обрабатывает весь трафик

---

### Active-Active (Multi-Master)

В этом паттерне **все серверы активно обрабатывают запросы** одновременно.

```
                    ┌─────────────────┐
                    │  Load Balancer  │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
        ┌──────────┐   ┌──────────┐   ┌──────────┐
        │ Server 1 │   │ Server 2 │   │ Server 3 │
        │ [ACTIVE] │   │ [ACTIVE] │   │ [ACTIVE] │
        └────┬─────┘   └────┬─────┘   └────┬─────┘
             │              │              │
             └──────────────┼──────────────┘
                            │
                    ┌───────┴───────┐
                    │   Shared DB   │
                    │  (or Sync)    │
                    └───────────────┘
```

#### Преимущества Active-Active:
- **Максимальное использование ресурсов** — все серверы работают
- **Горизонтальное масштабирование** — добавляй серверы по мере роста
- **Нулевой простой при отказе** — трафик просто перераспределяется
- **Лучшая производительность** — нагрузка распределена

#### Недостатки Active-Active:
- **Сложность синхронизации** — данные должны быть согласованы
- **Конфликты при записи** — несколько серверов могут писать одновременно
- **Более сложная логика** — требуется разрешение конфликтов
- **Подходит в основном для stateless** — или требует сложной репликации

#### Когда использовать какой паттерн:

| Критерий | Active-Passive | Active-Active |
|----------|----------------|---------------|
| Тип приложения | Stateful | Stateless |
| Бюджет | Ограничен | Достаточный |
| Требования к RTO | Минуты допустимы | Секунды/мгновенно |
| Сложность ops | Простое управление | Готовы к сложности |
| Нагрузка | Умеренная | Высокая |

---

## Replication (Репликация)

**Репликация** — это процесс копирования и поддержания идентичных данных на нескольких узлах. Это ключевой механизм для обеспечения доступности данных.

```
┌─────────────────────────────────────────────────────────────┐
│                    REPLICATION TYPES                         │
│                                                              │
│   ┌─────────────────────┐    ┌─────────────────────┐        │
│   │   Master-Slave      │    │   Master-Master     │        │
│   │                     │    │                     │        │
│   │      [Master]       │    │  [Master] [Master]  │        │
│   │         │           │    │      │       │      │        │
│   │    ┌────┼────┐      │    │      └───┬───┘      │        │
│   │    ▼    ▼    ▼      │    │          │          │        │
│   │  [S1] [S2] [S3]     │    │   ◄═══════════►     │        │
│   │                     │    │   bi-directional   │        │
│   └─────────────────────┘    └─────────────────────┘        │
└─────────────────────────────────────────────────────────────┘
```

---

### Master-Slave репликация (Primary-Replica)

В этой модели один узел (Master) принимает все записи, а реплики (Slaves) получают копии данных.

```
                         ┌───────────────────┐
                         │      CLIENT       │
                         └─────────┬─────────┘
                                   │
                    ┌──────────────┴──────────────┐
                    │                             │
               Write│                             │Read
                    ▼                             ▼
           ┌─────────────┐                ┌─────────────┐
           │   MASTER    │                │   SLAVES    │
           │  (Primary)  │                │ (Replicas)  │
           │             │                │             │
           │ [Read/Write]│───Replicate───►│ [Read Only] │
           └─────────────┘                └─────────────┘
```

#### Как это работает:

1. **Все записи** направляются на Master
2. Master записывает изменения в **Write-Ahead Log (WAL)** или **Binary Log**
3. Slaves получают изменения и применяют их локально
4. **Чтения** могут выполняться как с Master, так и со Slaves

#### Пример: PostgreSQL Streaming Replication

```sql
-- На Master: настройка репликации
-- postgresql.conf
wal_level = replica
max_wal_senders = 3
wal_keep_size = 1GB

-- pg_hba.conf
host replication replicator 192.168.1.0/24 md5

-- На Slave: подключение к Master
-- postgresql.conf
primary_conninfo = 'host=master port=5432 user=replicator'
```

#### Пример: MySQL Master-Slave

```sql
-- На Master
CHANGE MASTER TO
    MASTER_HOST='master_host',
    MASTER_USER='replication_user',
    MASTER_PASSWORD='password',
    MASTER_LOG_FILE='mysql-bin.000001',
    MASTER_LOG_POS=4;

START SLAVE;
```

#### Преимущества Master-Slave:
- **Простота** — чёткое разделение ролей
- **Масштабирование чтения** — добавляй реплики для read-heavy нагрузки
- **Нет конфликтов записи** — только Master пишет
- **Отказоустойчивость** — при падении Slave система работает

#### Недостатки Master-Slave:
- **Single Point of Failure** — Master может стать bottleneck
- **Задержка репликации** — Slave может отставать
- **Сложный failover** — требуется promotion Slave в Master
- **Ограниченное масштабирование записи** — только вертикальное

---

### Master-Master репликация (Multi-Master)

В этой модели **несколько узлов могут принимать записи**, и изменения синхронизируются между ними.

```
                         ┌───────────────────┐
                         │      CLIENTS      │
                         └─────────┬─────────┘
                                   │
                    ┌──────────────┴──────────────┐
                    │                             │
               R/W  │                             │  R/W
                    ▼                             ▼
           ┌─────────────┐                ┌─────────────┐
           │   MASTER 1  │◄═══════════════►│  MASTER 2  │
           │ [Read/Write]│  Bi-directional │[Read/Write]│
           │             │   Replication   │            │
           └─────────────┘                └─────────────┘
```

#### Проблема: Конфликты записи

Когда два Master'а одновременно изменяют одни и те же данные:

```
                 Master 1                    Master 2
                    │                           │
  UPDATE user       │                           │   UPDATE user
  SET name='Alice'  │                           │   SET name='Bob'
  WHERE id=1        │                           │   WHERE id=1
                    │                           │
                    └───────────┬───────────────┘
                                │
                         CONFLICT!
                      Какое значение
                       правильное?
```

#### Стратегии разрешения конфликтов:

**1. Last Write Wins (LWW)**
```
Используется timestamp — побеждает последняя запись

Master 1: name='Alice' @ 10:00:01
Master 2: name='Bob'   @ 10:00:02  ← Победитель

Результат: name='Bob'
```

**2. Application-level resolution**
```python
def resolve_conflict(local_value, remote_value, metadata):
    # Логика приложения решает, какое значение оставить
    if metadata['priority'][local_value] > metadata['priority'][remote_value]:
        return local_value
    return remote_value
```

**3. CRDT (Conflict-free Replicated Data Types)**
```
Специальные структуры данных, которые автоматически
мержатся без конфликтов (например, G-Counter, LWW-Register)
```

#### Пример: MySQL Group Replication

```sql
-- Включение Group Replication
SET GLOBAL group_replication_bootstrap_group=ON;
START GROUP_REPLICATION;

-- Проверка статуса
SELECT * FROM performance_schema.replication_group_members;
```

#### Пример: CockroachDB (Multi-active)

```sql
-- CockroachDB автоматически распределяет данные
-- и поддерживает multi-master записи

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name STRING,
    region STRING
);

-- Данные автоматически реплицируются между узлами
```

#### Преимущества Master-Master:
- **Нет single point of failure** — любой узел может принять запись
- **Лучшая локальность** — запись в ближайший датацентр
- **Масштабирование записи** — распределение нагрузки

#### Недостатки Master-Master:
- **Конфликты записи** — требуется стратегия разрешения
- **Сложность** — более сложная настройка и отладка
- **Latency** — синхронизация между узлами занимает время
- **Eventual consistency** — данные могут быть временно рассогласованы

---

### Синхронная vs Асинхронная репликация

#### Синхронная репликация

```
Client          Master          Replica
  │                │                │
  │   WRITE        │                │
  ├───────────────►│                │
  │                │  Replicate     │
  │                ├───────────────►│
  │                │                │
  │                │      ACK       │
  │                │◄───────────────┤
  │      OK        │                │
  │◄───────────────┤                │
  │                │                │

  Клиент получает OK только после подтверждения от реплики
```

**Характеристики:**
- **Гарантия сохранности** — данные точно на нескольких узлах
- **Высокая latency** — ждём подтверждения от реплики
- **Reduced availability** — если реплика недоступна, запись блокируется
- **Strong consistency** — все узлы всегда синхронизированы

#### Асинхронная репликация

```
Client          Master          Replica
  │                │                │
  │   WRITE        │                │
  ├───────────────►│                │
  │      OK        │                │
  │◄───────────────┤                │
  │                │                │
  │                │  Replicate     │
  │                ├───────────────►│ (позже)
  │                │                │

  Клиент получает OK сразу, репликация происходит позже
```

**Характеристики:**
- **Низкая latency** — клиент не ждёт репликацию
- **Высокая availability** — запись работает даже при проблемах с репликой
- **Возможна потеря данных** — при сбое Master до репликации
- **Eventual consistency** — реплики могут отставать

#### Сравнение:

| Аспект | Синхронная | Асинхронная |
|--------|------------|-------------|
| Latency записи | Высокая | Низкая |
| Durability | Гарантирована | Возможна потеря |
| Availability | Зависит от реплик | Высокая |
| Consistency | Strong | Eventual |
| Применение | Финансы, критичные данные | Логи, аналитика, кэши |

#### Semi-synchronous репликация (Гибридный подход)

```
Client          Master         Replica 1      Replica 2
  │                │               │              │
  │   WRITE        │               │              │
  ├───────────────►│               │              │
  │                ├──Replicate───►│              │
  │                │      ACK      │              │
  │                │◄──────────────┤              │
  │      OK        │               │              │
  │◄───────────────┤               │              │
  │                │               │              │
  │                ├──Replicate────┼─────────────►│
  │                │               │    (async)   │

  Ждём подтверждения от ОДНОЙ реплики, остальные — async
```

**Пример: MySQL Semi-sync Replication**
```sql
-- На Master
INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
SET GLOBAL rpl_semi_sync_master_enabled = 1;
SET GLOBAL rpl_semi_sync_master_wait_for_slave_count = 1;

-- На Slave
INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';
SET GLOBAL rpl_semi_sync_slave_enabled = 1;
```

---

## Примеры использования в реальных системах

### Netflix: Active-Active Multi-Region

```
┌─────────────────────────────────────────────────────────────────┐
│                     NETFLIX ARCHITECTURE                         │
│                                                                  │
│   ┌─────────────┐      ┌─────────────┐      ┌─────────────┐     │
│   │  US-EAST    │      │  US-WEST    │      │  EU-WEST    │     │
│   │  Region     │      │  Region     │      │  Region     │     │
│   │             │      │             │      │             │     │
│   │ ┌─────────┐ │      │ ┌─────────┐ │      │ ┌─────────┐ │     │
│   │ │ Services│ │      │ │ Services│ │      │ │ Services│ │     │
│   │ └─────────┘ │      │ └─────────┘ │      │ └─────────┘ │     │
│   │      │      │      │      │      │      │      │      │     │
│   │ ┌─────────┐ │      │ ┌─────────┐ │      │ ┌─────────┐ │     │
│   │ │Cassandra│◄├══════├►│Cassandra│◄├══════├►│Cassandra│ │     │
│   │ └─────────┘ │      │ └─────────┘ │      │ └─────────┘ │     │
│   └─────────────┘      └─────────────┘      └─────────────┘     │
│                                                                  │
│   - Active-Active: все регионы обслуживают трафик               │
│   - Cassandra: Multi-master репликация                          │
│   - Zuul: умный роутинг при сбоях                               │
└─────────────────────────────────────────────────────────────────┘
```

**Ключевые практики:**
- Каждый регион полностью автономен
- Cassandra с репликацией между регионами
- Chaos Monkey для тестирования отказоустойчивости
- При сбое региона трафик перенаправляется в другие

### Amazon Aurora: Multi-AZ with Read Replicas

```
┌─────────────────────────────────────────────────────────────────┐
│                     AMAZON AURORA                                │
│                                                                  │
│   Availability Zone A      Availability Zone B                  │
│   ┌─────────────────┐      ┌─────────────────┐                  │
│   │                 │      │                 │                  │
│   │  ┌───────────┐  │      │  ┌───────────┐  │                  │
│   │  │  Primary  │  │      │  │  Replica  │  │                  │
│   │  │  Writer   │  │      │  │  Reader   │  │                  │
│   │  └─────┬─────┘  │      │  └─────┬─────┘  │                  │
│   │        │        │      │        │        │                  │
│   └────────┼────────┘      └────────┼────────┘                  │
│            │                        │                            │
│            └────────────┬───────────┘                            │
│                         │                                        │
│              ┌──────────┴──────────┐                             │
│              │   Shared Storage    │                             │
│              │   (6 copies, 3 AZ)  │                             │
│              └─────────────────────┘                             │
│                                                                  │
│   - Synchronous replication to storage                          │
│   - Automatic failover < 30 seconds                             │
│   - Up to 15 read replicas                                      │
└─────────────────────────────────────────────────────────────────┘
```

### PostgreSQL: Patroni для автоматического failover

```yaml
# patroni.yml
scope: postgres-cluster
name: node1

restapi:
  listen: 0.0.0.0:8008
  connect_address: node1:8008

etcd:
  hosts: etcd1:2379,etcd2:2379,etcd3:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    maximum_lag_on_failover: 1048576  # 1MB
    synchronous_mode: true

postgresql:
  listen: 0.0.0.0:5432
  connect_address: node1:5432
  data_dir: /data/patroni

  parameters:
    synchronous_commit: "on"
    synchronous_standby_names: "*"
```

### Redis: Sentinel для High Availability

```
┌─────────────────────────────────────────────────────────────────┐
│                    REDIS SENTINEL                                │
│                                                                  │
│   ┌───────────┐     ┌───────────┐     ┌───────────┐             │
│   │ Sentinel  │     │ Sentinel  │     │ Sentinel  │             │
│   │    1      │     │    2      │     │    3      │             │
│   └─────┬─────┘     └─────┬─────┘     └─────┬─────┘             │
│         │                 │                 │                    │
│         └─────────────────┼─────────────────┘                    │
│                           │                                      │
│                    ┌──────┴──────┐                               │
│                    │             │                               │
│              ┌─────┴─────┐ ┌─────┴─────┐                         │
│              │  Master   │ │  Slave    │                         │
│              │   Redis   │ │   Redis   │                         │
│              └───────────┘ └───────────┘                         │
│                                                                  │
│   - Мониторинг состояния Master                                 │
│   - Автоматический failover при сбое                            │
│   - Кворум для принятия решений                                 │
└─────────────────────────────────────────────────────────────────┘
```

```python
# Подключение к Redis через Sentinel
from redis.sentinel import Sentinel

sentinel = Sentinel([
    ('sentinel1', 26379),
    ('sentinel2', 26379),
    ('sentinel3', 26379)
], socket_timeout=0.1)

# Получение текущего Master
master = sentinel.master_for('mymaster', socket_timeout=0.1)
master.set('key', 'value')

# Получение Slave для чтения
slave = sentinel.slave_for('mymaster', socket_timeout=0.1)
value = slave.get('key')
```

---

## Trade-offs: Доступность vs Согласованность

Согласно CAP-теореме, при сетевых разделениях (partition) мы должны выбирать между доступностью и согласованностью.

### Спектр систем

```
Strong                                              High
Consistency                                    Availability
    │                                                │
    ├─── Spanner (Google) ──────────────────────────┤
    │    Synchronous, global transactions            │
    │                                                │
    ├─── PostgreSQL (sync replication) ─────────────┤
    │    Synchronous, single-master                  │
    │                                                │
    ├─── MongoDB ────────────────────────────────────┤
    │    Configurable consistency                    │
    │                                                │
    ├─── Cassandra ──────────────────────────────────┤
    │    Tunable consistency (QUORUM, ONE, ALL)      │
    │                                                │
    ├─── DynamoDB ───────────────────────────────────┤
    │    Eventual consistency by default             │
    │                                                │
    └─── DNS ────────────────────────────────────────┘
         Eventually consistent, high availability
```

### Практические рекомендации

| Сценарий | Приоритет | Паттерн |
|----------|-----------|---------|
| Финансовые транзакции | Consistency | Sync replication, Master-Slave |
| Социальные сети (лайки) | Availability | Async replication, Eventual consistency |
| E-commerce корзина | Availability | CRDT, Multi-master |
| Инвентарь (остатки) | Consistency | Synchronous, Strong consistency |
| Сессии пользователей | Availability | Async replication |
| Аудит логи | Durability | Sync to at least 2 nodes |

### Пример: Tunable Consistency в Cassandra

```python
from cassandra.cluster import Cluster
from cassandra import ConsistencyLevel
from cassandra.query import SimpleStatement

cluster = Cluster(['node1', 'node2', 'node3'])
session = cluster.connect('keyspace')

# Высокая согласованность для критичных операций
critical_query = SimpleStatement(
    "UPDATE accounts SET balance = ? WHERE id = ?",
    consistency_level=ConsistencyLevel.QUORUM
)
session.execute(critical_query, [1000, 'user123'])

# Высокая доступность для некритичных чтений
fast_query = SimpleStatement(
    "SELECT * FROM user_preferences WHERE id = ?",
    consistency_level=ConsistencyLevel.ONE
)
session.execute(fast_query, ['user123'])
```

---

## Резюме

### Ключевые паттерны доступности:

| Паттерн | Когда использовать | Trade-offs |
|---------|-------------------|------------|
| **Cold Standby** | Некритичные системы, экономия | Долгое восстановление |
| **Warm Standby** | Бизнес-системы | Баланс стоимости/скорости |
| **Hot Standby** | Mission-critical | Высокая стоимость |
| **Active-Active** | Высокая нагрузка, geo-distributed | Сложность синхронизации |
| **Master-Slave** | Read-heavy нагрузки | Bottleneck на Master |
| **Master-Master** | Write scaling, geo-locality | Конфликты записи |
| **Sync Replication** | Критичные данные | Высокая latency |
| **Async Replication** | Логи, аналитика | Возможная потеря данных |

### Формула успешной системы высокой доступности:

```
HA System = Redundancy + Failover + Monitoring + Testing
```

1. **Redundancy** — дублирование компонентов
2. **Failover** — автоматическое переключение
3. **Monitoring** — мониторинг здоровья системы
4. **Testing** — регулярное тестирование отказов (Chaos Engineering)

---

## Дополнительные ресурсы

- [AWS Well-Architected Framework: Reliability Pillar](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/)
- [Google SRE Book: Chapter on Availability](https://sre.google/sre-book/availability-table/)
- [Netflix Tech Blog: Active-Active](https://netflixtechblog.com/)
- [Patroni Documentation](https://patroni.readthedocs.io/)
- [Cassandra Consistency Levels](https://cassandra.apache.org/doc/latest/cassandra/cql/consistency.html)
