# Управление Redis в Production

## Конфигурация redis.conf

### Основные параметры

```conf
# Привязка к интерфейсам
bind 127.0.0.1 -::1
# В production: bind к внутренней сети, не к 0.0.0.0

# Порт
port 6379

# Пароль (обязательно в production!)
requirepass your_strong_password_here

# Максимальное количество клиентов
maxclients 10000

# Максимальная память
maxmemory 2gb
maxmemory-policy allkeys-lru
```

### Параметры персистентности

```conf
# RDB snapshots
save 900 1      # Снимок если 1 изменение за 900 сек
save 300 10     # Снимок если 10 изменений за 300 сек
save 60 10000   # Снимок если 10000 изменений за 60 сек

# Файл RDB
dbfilename dump.rdb
dir /var/lib/redis

# AOF
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec   # Оптимальный баланс

# AOF rewrite
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

### Параметры производительности

```conf
# TCP keepalive
tcp-keepalive 300

# Таймаут неактивных клиентов (0 = отключено)
timeout 0

# Количество баз данных
databases 16

# Lazy freeing (освобождение памяти в фоне)
lazyfree-lazy-eviction yes
lazyfree-lazy-expire yes
lazyfree-lazy-server-del yes

# I/O threads (Redis 6+)
io-threads 4
io-threads-do-reads yes
```

### Логирование

```conf
# Уровень логов
loglevel notice   # debug, verbose, notice, warning

# Файл логов
logfile /var/log/redis/redis.log

# Slow log
slowlog-log-slower-than 10000  # 10ms
slowlog-max-len 128
```

## Изменение конфигурации на лету

```redis
# Просмотр параметра
CONFIG GET maxmemory

# Изменение параметра
CONFIG SET maxmemory 4gb

# Сохранение изменений в redis.conf
CONFIG REWRITE
```

**Нельзя изменить на лету:**
- `bind`, `port` — требуют перезапуска
- `daemonize` — требует перезапуска
- `pidfile` — требует перезапуска

## Стратегии резервного копирования

### Backup RDB файла

```bash
# 1. Создать снимок
redis-cli BGSAVE

# 2. Дождаться завершения
redis-cli LASTSAVE  # Проверить timestamp

# 3. Скопировать файл
cp /var/lib/redis/dump.rdb /backup/redis-$(date +%Y%m%d).rdb
```

**Автоматизация через cron:**
```bash
0 */6 * * * redis-cli BGSAVE && sleep 60 && cp /var/lib/redis/dump.rdb /backup/redis-$(date +\%Y\%m\%d-\%H\%M).rdb
```

### Backup AOF файла

```bash
# AOF файлы в директории appendonlydir (Redis 7+)
cp -r /var/lib/redis/appendonlydir /backup/aof-$(date +%Y%m%d)/
```

### Backup с репликой

Лучшая практика — делать backup с реплики:

```
Master ──> Replica (для чтения)
               └──> Backup (BGSAVE на реплике)
```

Так BGSAVE не влияет на производительность мастера.

## Восстановление из backup

### Восстановление RDB

```bash
# 1. Остановить Redis
systemctl stop redis

# 2. Заменить dump.rdb
cp /backup/redis-20240101.rdb /var/lib/redis/dump.rdb
chown redis:redis /var/lib/redis/dump.rdb

# 3. Запустить Redis
systemctl start redis
```

### Восстановление AOF

```bash
# 1. Остановить Redis
systemctl stop redis

# 2. Заменить AOF директорию
rm -rf /var/lib/redis/appendonlydir
cp -r /backup/aof-20240101 /var/lib/redis/appendonlydir
chown -R redis:redis /var/lib/redis/appendonlydir

# 3. Запустить Redis
systemctl start redis
```

### Восстановление повреждённого AOF

```bash
# Проверка целостности
redis-check-aof --fix /var/lib/redis/appendonlydir/appendonly.aof.1.incr.aof

# Подтвердить исправление
# Redis покажет, сколько байт будет отброшено
```

## Апгрейд Redis с минимальным downtime

### Метод 1: Rolling upgrade (с репликацией)

```
1. Текущая конфигурация:
   Master (v7.0) ──> Replica (v7.0)

2. Апгрейд реплики:
   Master (v7.0) ──> Replica (v7.2) ✓

3. Failover:
   Master (v7.0) <── Replica (v7.2) становится Master

4. Апгрейд старого мастера:
   Replica (v7.2) <── Master (v7.2) ✓
```

**Команды:**
```redis
# На реплике (будет новым мастером)
REPLICAOF NO ONE

# На старом мастере (станет репликой)
REPLICAOF new_master_ip 6379
```

### Метод 2: Blue-Green deployment

```
1. Запустить новый кластер (v7.2)
2. Настроить репликацию: Old Master ──> New Master
3. Дождаться синхронизации
4. Переключить приложения на новый кластер
5. Отключить старый кластер
```

## Disaster Recovery Plan

### 1. Определение RPO и RTO

- **RPO (Recovery Point Objective)** — максимальная потеря данных
  - AOF everysec: ~1 секунда
  - RDB каждые 5 минут: ~5 минут

- **RTO (Recovery Time Objective)** — время восстановления
  - Зависит от размера данных и инфраструктуры

### 2. Географическая репликация

```
Datacenter A          Datacenter B
┌─────────────┐       ┌─────────────┐
│   Master    │──────>│   Replica   │
│  (Primary)  │       │  (DR Site)  │
└─────────────┘       └─────────────┘
```

### 3. Чек-лист восстановления

```markdown
[ ] Определить источник backup (RDB/AOF/Replica)
[ ] Проверить целостность файлов
[ ] Развернуть новый Redis instance
[ ] Восстановить данные
[ ] Проверить данные (DBSIZE, выборочные ключи)
[ ] Переключить приложения
[ ] Мониторинг после восстановления
```

## Мониторинг в Production

### Ключевые метрики

| Метрика | Источник | Alert threshold |
|---------|----------|-----------------|
| Memory usage | INFO memory | >85% maxmemory |
| Connected clients | INFO clients | >80% maxclients |
| Blocked clients | INFO clients | >0 (длительно) |
| Evicted keys | INFO stats | >0 |
| Keyspace hit rate | INFO stats | <80% |
| Replication lag | INFO replication | >10 sec |
| Latency | redis-cli --latency | >10ms p99 |

### Prometheus + Grafana

```yaml
# docker-compose.yml
services:
  redis-exporter:
    image: oliver006/redis_exporter
    environment:
      - REDIS_ADDR=redis://redis:6379
      - REDIS_PASSWORD=your_password
    ports:
      - "9121:9121"
```

### Alertmanager rules

```yaml
groups:
- name: redis
  rules:
  - alert: RedisMemoryHigh
    expr: redis_memory_used_bytes / redis_memory_max_bytes > 0.85
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Redis memory usage is high"

  - alert: RedisDown
    expr: redis_up == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Redis is down"
```

## Redis Enterprise

### Когда рассмотреть Enterprise

- **Active-Active geo-replication** — нужна запись в нескольких регионах
- **Redis on Flash** — данные больше RAM, нужен tiering на SSD
- **Enterprise security** — LDAP, encryption at rest, audit logs
- **99.999% SLA** — критичный бизнес-сервис
- **Поддержка 24/7** — нет внутренней экспертизы

### Возможности Enterprise

| Feature | Open Source | Enterprise |
|---------|-------------|------------|
| Active-Active | Нет | Да |
| Auto-tiering (Flash) | Нет | Да |
| CRDT (conflict resolution) | Нет | Да |
| Encryption at rest | Нет | Да |
| LDAP/SAML | Нет | Да |
| 24/7 Support | Community | Да |

## Security Checklist для Production

```markdown
[ ] Установлен сильный пароль (requirepass)
[ ] Redis не слушает 0.0.0.0 (bind на internal IP)
[ ] Firewall ограничивает доступ к порту 6379
[ ] TLS включён для шифрования трафика
[ ] Опасные команды переименованы или отключены
[ ] Регулярные backup настроены и тестируются
[ ] Мониторинг и алерты настроены
[ ] Логи ротируются и сохраняются
```

### Отключение опасных команд

```conf
# redis.conf
rename-command FLUSHDB ""
rename-command FLUSHALL ""
rename-command DEBUG ""
rename-command CONFIG "CONFIG_b4d8f2a1"  # Переименование
```

## Типичные ошибки в Production

1. **Нет maxmemory** — Redis съест всю память
2. **KEYS в production** — блокирует Redis
3. **Большие значения (>100KB)** — замедляют операции
4. **Нет мониторинга** — проблемы обнаруживаются поздно
5. **Один инстанс** — single point of failure
6. **Backup не тестируются** — могут быть неработоспособны
